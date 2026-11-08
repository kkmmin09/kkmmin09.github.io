---
layout: post
title: "PyQt5 기초: 위젯, 레이아웃, 표(QTableWidget)"
excerpt: "PyQt5로 창을 만들 때 기본이 되는 위젯과 시그널, 레이아웃으로 자동 배치하는 법, 2차원 데이터를 표로 보여주는 QTableWidget을 정리했습니다."
comments: true
---

# PyQt5 기초

[OpenCV 웹캠](/OpenCV/)과 [뉴스 조회 프로그램](/News-Collection/)에서 PyQt5로 화면을 만들었는데, 그 밑바탕이 되는 기초를 따로 정리했습니다.

## 핵심 포인트

* 위젯을 만들고 **시그널을 `connect(함수)`로 연결**하면, 이벤트가 생길 때 그 함수가 실행된다.
* `move()`로 좌표를 하나하나 정하지 않고 **레이아웃**이 알아서 배치하게 한다.
* `QHBoxLayout`은 가로, `QVBoxLayout`은 세로이고, **레이아웃 안에 레이아웃**을 넣을 수 있다.
* 표는 **`QTableWidget`**에 칸마다 `QTableWidgetItem`을 넣어서 만든다.

## 1. 기본 틀

```python
import sys
from PyQt5.QtWidgets import *
from PyQt5.QtCore import *

class MyWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.initUI()

    def initUI(self):
        self.setWindowTitle('SWH Academy Window.')
        self.setGeometry(200, 200, 360, 100)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = MyWindow()
    window.show()
    sys.exit(app.exec_())
```

* 창 클래스를 만들고 `initUI()`에서 위젯을 배치하는 모양으로 거의 모든 예제를 만들었습니다.
* `app.exec_()`가 이벤트를 기다리는 반복을 돌리고, 창을 닫으면 끝납니다.

## 2. 자주 쓴 위젯과 시그널

| 위젯 | 용도 | 자주 쓰는 시그널 |
|---|---|---|
| `QPushButton` | 버튼 | `clicked` |
| `QLineEdit` | 한 줄 입력 | `textChanged` |
| `QCheckBox` | 체크박스 | `stateChanged` |
| `QRadioButton` + `QGroupBox` | 여러 개 중 하나만 | `clicked` |
| `QComboBox` | 드롭다운 | `currentTextChanged` |
| `QSpinBox` / `QSlider` / `QDial` | 숫자 조절 | `valueChanged` |
| `QProgressBar` + `QBasicTimer` | 진행 막대 | 타이머 이벤트 |
| `QLabel` + `QPixmap` | 글자, 이미지 표시 | — |

```python
button = QPushButton("지우기", self)
button.clicked.connect(self.button2Clicked)   # 함수 이름만! 괄호 X
```

* `connect(self.button2Clicked())`처럼 괄호를 붙이면 **누르기도 전에 바로 실행**되고, 그 결과값이 연결돼서 동작하지 않습니다.

## 3. 레이아웃으로 배치하기

입력창에 글자를 쓰면 위쪽 라벨과 텍스트창에 같은 글자가 바로 나오는 예제입니다.

```python
def initUI(self):
    self.label = QLabel("", self)
    self.textEdit = QTextEdit()

    self.lineEdit = QLineEdit("", self)
    self.lineEdit.textChanged.connect(self.lineEditChanged)   # 입력할 때마다

    button2 = QPushButton("지우기", self)
    button2.clicked.connect(self.button2Clicked)
    button3 = QPushButton("종료", self)
    button3.clicked.connect(QCoreApplication.instance().quit)

    bottomLayout = QHBoxLayout()      # 아래: 가로
    bottomLayout.addStretch(5)        # 왼쪽 빈 공간이 커서 오른쪽으로 몰림
    bottomLayout.addWidget(self.lineEdit)
    bottomLayout.addWidget(button2)
    bottomLayout.addWidget(button3)
    bottomLayout.addStretch(1)

    topLayout = QVBoxLayout()         # 위: 세로
    topLayout.addWidget(self.label)
    topLayout.addWidget(self.textEdit)

    layout = QVBoxLayout()            # 전체: 위 + 아래
    layout.addLayout(topLayout)
    layout.addLayout(bottomLayout)
    self.setLayout(layout)            # 이걸 해야 적용됨

def lineEditChanged(self):
    self.label.setText(self.lineEdit.text())
    self.textEdit.setText(self.lineEdit.text())

def button2Clicked(self):
    self.label.clear()
    self.lineEdit.clear()
    self.textEdit.clear()
```

```
┌──────────────────────────────┐
│ label                        │  ← topLayout (세로)
│ ┌──────────────────────────┐ │
│ │ textEdit                 │ │
│ └──────────────────────────┘ │
│        [ 입력창 ][지우기][종료] │  ← bottomLayout (가로)
└──────────────────────────────┘
```

* `addStretch(비율)`로 빈 공간을 넣습니다. 왼쪽 5, 오른쪽 1이라 위젯들이 오른쪽으로 몰립니다.
* 레이아웃을 쓰면 **창 크기를 바꿔도 위젯이 같이 늘어나고 줄어듭니다.**
* 로그인 폼처럼 표 모양으로 놓을 때는 `QGridLayout`, 창 두 개를 오갈 때는 `hide()` / `show()`를 썼습니다.

## 4. 표 만들기 (QTableWidget)

```python
def initUI(self):
    self.rows = [
        ['1', '코야', '2011.06'],
        ['2', '레종', '2013.02'],
        ['3', '유키', '2015.10']
    ]
    headers = ['No.', '이름', '출생일']

    self.tableWidget = QTableWidget(self)
    self.tableWidget.setRowCount(len(self.rows))       # 행 수 = 데이터 개수
    self.tableWidget.setColumnCount(len(headers))      # 열 수 = 제목 개수
    self.tableWidget.setHorizontalHeaderLabels(headers)
    self.tableWidget.setEditTriggers(QAbstractItemView.NoEditTriggers)   # 수정 막기
    self.setTableWidgetData()

    layout = QVBoxLayout()
    layout.addWidget(QLabel("우리집 고양이", self))
    layout.addWidget(self.tableWidget)
    self.setLayout(layout)

def setTableWidgetData(self):
    for r in range(len(self.rows)):
        for c in range(len(self.rows[r])):
            item = QTableWidgetItem(self.rows[r][c])
            if c == 0:
                item.setTextAlignment(Qt.AlignRight)   # 번호 칸만 오른쪽 정렬
            self.tableWidget.setItem(r, c, item)
    self.tableWidget.resizeColumnsToContents()          # 칸 너비를 내용에 맞추기
```

* 2차원 리스트를 **이중 for문**으로 돌면서 `setItem(행, 열, item)`으로 칸을 채웁니다.
* 행 수, 열 수를 숫자로 쓰지 않고 `len()`으로 정해서 데이터가 늘어나도 코드를 안 고쳐도 됩니다.
* [뉴스 조회 프로그램](/News-Collection/)에서 DB 조회 결과를 표로 보여줄 때도 이 방법을 썼습니다.

## 아쉬운 점

* 창을 화면 가운데로 옮기려고 `moveCenter()`까지만 했는데, 마지막에 `self.move(frameGeometry.topLeft())`를 안 해서 실제로는 창이 안 움직였습니다.
* 위젯이 많아지면 코드로 배치하기 힘들어서, 나중에는 Qt Designer로 `.ui` 파일을 만들어 `uic.loadUi`로 불러오는 방식으로 바꿨습니다.
