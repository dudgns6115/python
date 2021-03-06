
# Test 

- Test 입니다.
    - Test2

```shell
!pip install PyPDF3
!pip install pillow
!pip install PyQt5
!pip install reportlaab
```

import sys, os
from PyQt5.QtWidgets import QApplication, QMainWindow, QListWidget, QListWidgetItem, QPushButton, QLabel, QFileDialog, \
    QBoxLayout, QMessageBox
from PyQt5.QtCore import Qt, QUrl
from PIL import Image
from reportlab.pdfgen import canvas
from PyPDF3 import PdfFileReader, PdfFileWriter


class ListBoxWidget(QListWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setAcceptDrops(True)
        self.resize(600, 600)

    def dragEnterEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
        else:
            event.ignore()

    def dragMoveEvent(self, event):
        if event.mimeData().hasUrls:
            event.setDropAction(Qt.CopyAction)
            event.accept()
        else:
            event.ignore()

    def dropEvent(self, event):
        if event.mimeData().hasUrls():
            event.setDropAction(Qt.CopyAction)
            event.accept()

            links = []

            for url in event.mimeData().urls():
                if url.isLocalFile():
                    links.append(str(url.toLocalFile()))
                else:
                    links.append(str(url.toString()))
            self.addItems(links)

        else:
            event.ignore()

class AppDemo(QMainWindow):
    def __init__(self):
        super().__init__()
        self.resize(1200, 600)
        self.setWindowTitle('Statusbar')

        self.itemListBox_view = ListBoxWidget(self)

        self.lb = QLabel('', self)
        self.lb.setGeometry(850,100,300,50)
        self.pb = QPushButton("Get full path of a file", self)
        self.pb.setGeometry(850,200,150,50)

        self.pb.clicked.connect(self.get_file_name)

        self.btn = QPushButton('Merge', self)
        self.btn.setGeometry(850,400,200,50)
        print(self.getSelectedItem())

        self.btn.clicked.connect(lambda : self.makeWatermarkPDF())

    def get_file_name(self):
        filename = QFileDialog.getOpenFileName()
        if filename[0]:
            fname, ext = os.path.splitext(filename[0])
            if ext == '.png':
                self.lb.setText(filename[0])
            else:
                QMessageBox.about(self, "Warning", "png 확장자를 가진 파일만 가능합니다.")
        else:
            QMessageBox.about(self, "Warning", "파일을 선택하지 않았습니다.")

    def getSelectedItem(self):
        item = QListWidgetItem(self.itemListBox_view.currentItem())
        return item.text()

    def makeWatermarkPDF(self):
        image = Image.open(self.lb.text(), 'r')
        print(self.lb.text())
        clearImage = self.clearWhiteBackground(image)
        print(os.getcwd())
        clearImage.save(os.getcwd() + '/clearSample.png')
        print(clearImage)

        self.imageToPDF(os.getcwd() + '/clearSample.png', os.getcwd() + '/watermarkImage.pdf')
        self.selectedList = self.itemListBox_view.selectedItems()
        print(self.selectedList)
        print(self.itemListBox_view.count())

        items = []
        for i in range(self.itemListBox_view.count()):
            items.append(self.itemListBox_view.item(i).text())
            print(items)

        for j in range(self.itemListBox_view.count()):
            self.pdfMerge(os.getcwd() + '/complete' + str(j)+ '.pdf', items[j], os.getcwd() + '/watermarkImage.pdf')
        self.itemListBox_view.clear()

    def clearWhiteBackground(self, inputImage):
        # RGBA로 속성 변경
        image = inputImage.convert('RGBA')

        # 해당 이미지의 배열 받아오기
        imageData = inputImage.getdata()

        newImageData = []

        for pixel in imageData:
            if pixel[0] > 240 and pixel[1] > 240 and pixel[2] > 240:
                # rgb값이 240,240,240 이상일 경우(흰색에 가까울 경우) 알파값을 0으로 준다
                newImageData.append((0, 0, 0, 0))
            else:
                # 아닐경우 그대로 쓴다
                newImageData.append(pixel)

        # 이미지를 덮어 씌움
        image.putdata(newImageData)
        # 투명도 값 (0~255), 절반정도 투명도
        image.putalpha(128)

        return image

    def imageToPDF(self, imagePath, pdfPath):
        newCanvas = canvas.Canvas(pdfPath, pagesize=Image.open(imagePath, 'r').size)

        newCanvas.drawImage(image=imagePath, x=0, y=0, mask='auto')

        newCanvas.save()

    def pdfMerge(self, savePath, pdfPath, watermarkPdfPath):
        # pdf파일 불러오기
        pdfFile = open(pdfPath, 'rb')
        pdfReader = PdfFileReader(pdfFile, strict=False)

        # 워터마크 PDF파일 불러오기
        watermarkPdfFile = open(watermarkPdfPath, 'rb')
        watermarkPdf = PdfFileReader(watermarkPdfFile, strict=False).getPage(0)

        pdfWriter = PdfFileWriter()

        # PDF 페이지 수만큼 반복
        for pageNum in range(pdfReader.numPages):
            # 페이지를 불러온다
            pageObj = pdfReader.getPage(pageNum)

            # 중앙으로 놓기 위해 좌표를 구한다
            x = (pageObj.mediaBox[2] - watermarkPdf.mediaBox[2]) / 2
            y = (pageObj.mediaBox[3] - watermarkPdf.mediaBox[3]) / 2

            # 워터마크페이지와 합친다
            pageObj.mergeTranslatedPage(page2=watermarkPdf, tx=x, ty=y, expand=False)

            # 합친걸 저장할 PDF파일에 추가한다
            pdfWriter.addPage(pageObj)

        # 저장
        resultFile = open(savePath, 'wb')
        pdfWriter.write(resultFile)


if __name__ == '__main__':
    app = QApplication(sys.argv)
    demo = AppDemo()
    demo.show()
    sys.exit(app.exec_())
