---
tags: [qt, pyqt5, designer, python, gui]
updated: 2026-09-08
---

# Qt Designer와 PyQt5 코드 연결

Qt Designer로 `.ui`를 그리고 `pyuic5`로 `.py`로 변환한 뒤, 별도 파이썬 파일에서 그걸 import해 위젯에 동작을 붙이는 워크플로. **위젯의 `objectName`이 디자인과 코드를 잇는 유일한 열쇠**이고, 폼 자체의 `objectName`을 잘못 건드리면 생성되는 클래스 이름이 바뀌어 코드가 통째로 어긋난다.

## 기본 흐름

```bash
pyuic5 -x led_gui.ui -o led_gui.py     # .ui → .py
```

변환된 `led_gui.py`에는 `Ui_<폼의 objectName>` 클래스가 생기고, 그 안의 `setupUi(self)`가 각 위젯을 `self.<objectName>` 속성으로 만든다.

```python
import led_gui

class LedApp(QtWidgets.QDialog):
    def __init__(self):
        super().__init__()
        self.ui = led_gui.Ui_Dialog()
        self.ui.setupUi(self)              # 여기서 self.ui.btnRed 등이 생성됨
        self.ui.btnRed.clicked.connect(self.toggle)
```

`.ui`를 고칠 때마다 재변환하는 게 번거로우면 실행 시점에 직접 읽는 방법도 있다. 이때는 `self.ui.btnRed`가 아니라 `self.btnRed`로 접근한다.

```python
from PyQt5 import uic
uic.loadUi("led_gui.ui", self)
```

## ⚠️ 폼의 objectName을 바꾸면 클래스 이름이 바뀐다

Designer에서 버튼 이름을 바꾸려다 **폼(최상위 Dialog)이 선택된 상태**로 `objectName`을 입력하면, `.ui`의 최상위 클래스가 통째로 바뀐다.

```xml
<class>LED1_Button</class>          <!-- 폼 objectName을 실수로 바꾼 상태 -->
```

이러면 `pyuic5`가 `Ui_Dialog`가 아니라 `Ui_LED1_Button`을 만들어내고, 코드는 `AttributeError` 또는 import 실패로 죽는다. 폼의 `objectName`은 `Dialog`로 두고 창 제목은 `windowTitle` 속성으로 따로 지정한다.

`.ui`는 XML이라 이름 상태를 눈으로 확인할 수 있다.

```bash
grep -o '<class>[^<]*</class>' led_gui.ui
grep -o 'widget class="[A-Za-z]*" name="[A-Za-z0-9_]*"' led_gui.ui
```

## 레이아웃 단축키

Qt Designer의 배치 단축키는 `Ctrl+L`이 아니다.

| 동작 | 단축키 |
|---|---|
| 가로로 배치 (Lay Out Horizontally) | `Ctrl+1` |
| 세로로 배치 (Lay Out Vertically) | `Ctrl+2` |
| 격자 배치 (Lay Out in a Grid) | `Ctrl+5` |
| 레이아웃 해제 (Break Layout) | `Ctrl+0` |
| 미리보기 | `Ctrl+R` |

레이아웃을 하나도 적용하지 않으면 위젯이 절대 좌표로 고정되어 **창 크기를 바꿔도 따라오지 않는다.** 동작에는 지장이 없지만 리사이즈가 필요하면 반드시 잡아야 한다.

## Qt5와 Qt6 Designer는 섞이지 않는다

한 PC에 Designer가 여러 개 깔려 있을 수 있다. **`pyuic5`를 쓸 거라면 Qt5 Designer로 만든 `.ui`여야 한다.** PySide6(Qt6) Designer가 만든 `.ui`는 `pyuic5`가 읽지 못한다. (Claude 보충 — 버전 불일치 시 실패하는 것은 일반적인 사실이나 이 세션에서 직접 재현하지는 않았다)

Windows + Anaconda 조합에서의 실제 경로 예:

```
C:\Users\<사용자>\anaconda3\Library\bin\designer.exe      # Qt5 Designer (conda qt-main 동봉)
C:\Users\<사용자>\anaconda3\Scripts\pyuic5.exe            # 변환기
```

pip으로 설치한 순수 PyQt5에는 Designer가 들어 있지 않다. conda의 `qt-main` 패키지나 리눅스의 `qttools5-dev-tools`처럼 Qt 툴 패키지가 따로 있어야 한다.

## pyuic5가 실제로 만드는 것

Designer가 저장하는 `.ui`는 **XML 텍스트**라 파이썬이 그대로 실행할 수 없다. `pyuic5`는 이 XML을 읽어 **같은 화면을 만들어내는 파이썬 코드**로 번역한다.

```xml
<widget class="QSlider" name="sliderRed">
  <property name="maximum"><number>255</number></property>
  <property name="orientation"><enum>Qt::Vertical</enum></property>
</widget>
```

위 `.ui` 조각이 아래 코드가 된다.

```python
class Ui_Dialog(object):
    def setupUi(self, Dialog):
        self.sliderRed = QtWidgets.QSlider(Dialog)
        self.sliderRed.setMaximum(255)
        self.sliderRed.setOrientation(QtCore.Qt.Vertical)
```

Designer에서 지정한 **`objectName`이 그대로 파이썬 속성 이름**이 된다. 이름을 바꾸지 않으면 `verticalSlider_2` 같은 자동 이름이 붙어 코드가 읽기 어려워진다.

`-x` 옵션은 생성 파일 끝에 실행 블록을 붙여, `python led_gui.py`만으로 화면을 미리 볼 수 있게 한다.

⚠️ **생성된 `.py`는 직접 고치지 않는다.** `.ui`를 수정하고 재변환하면 통째로 덮어써진다. 동작 코드는 항상 별도 파일에 두고 `import`해서 쓴다.

## 반복문에서 시그널을 연결할 때 — 람다 늦은 바인딩

여러 위젯을 반복문으로 연결할 때 `lambda`가 루프 변수를 **참조로** 잡으면, 모든 콜백이 마지막 값 하나만 보게 된다. 기본 인자로 캡처해야 한다.

```python
for slider, label in rows.values():
    slider.valueChanged.connect(
        lambda v, lb=label: lb.setText(str(v))   # lb=label 로 그 시점 값을 고정
    )
```

`lambda v: label.setText(...)`로 쓰면 슬라이더 셋 다 마지막 라벨만 갱신한다.

## 표시와 적용을 분리하는 패턴

값을 바꾸는 즉시 하드웨어에 반영할지, 확인 버튼을 눌렀을 때 반영할지는 설계 선택이다. 분리하면 슬라이더를 드래그하는 동안 중간값이 계속 전송되는 것을 막을 수 있다.

```python
slider.valueChanged.connect(...)      # 숫자 라벨만 갱신 (가벼움)
btnSend.clicked.connect(self.send)    # 이때 한 번에 하드웨어 반영
```

## 출처

Claude Code 세션 (2026-09-08). 수업 자료 `heartcom/Linux-Program` 2번 PPT(Rpi_GPIO_DHT11_PyQt) 실습을 Windows PC + 라즈베리파이 4로 진행하며 확인.

관련: [[라즈베리파이-PyQt5-설치-ARM64]], [[라즈베리파이-GUI-SSH-VNC-실행]]
