---
tags: [qt, pyqt5, qss, stylesheet, designer, ui, 테마]
updated: 2026-09-15
---

# Qt 스타일시트(QSS)로 테마 입히기

**QSS**는 CSS와 비슷한 문법으로 Qt 위젯의 색·테두리·글꼴을 바꾸는 기능이다. 폼(Dialog)의 `styleSheet` 속성에 통째로 넣으면 `.ui`에 저장되어 `pyuic5` 변환 결과에 그대로 들어간다. 다만 **네온 발광 같은 번짐 효과는 QSS로 안 되고, 코드에서 `QGraphicsDropShadowEffect`로** 붙여야 한다.

## 어디에 넣나

`.ui` 최상위 폼의 `styleSheet` 속성에 넣었다. `pyuic5`는 이를 `Dialog.setStyleSheet("...")`로 변환한다. 스타일이 코드 여기저기 흩어지지 않고 디자인 파일 한 곳에 모인다.

```xml
<property name="styleSheet">
 <string notr="true">
QDialog#Dialog {
  background: qlineargradient(x1:0, y1:0, x2:0, y2:1, stop:0 #0b1120, stop:1 #03050a);
}
QLabel { color: #6f8bab; font-family: "DejaVu Sans Mono"; font-size: 11px; }
 </string>
</property>
```

- `.ui`는 XML이라 QSS 안에 `<`나 `&`를 그대로 쓰면 안 된다.
- Designer에서 열면 이 스타일이 미리보기에 반영된다 (Claude 보충 — Designer로 직접 열어 확인하지는 않았다).

## ID 선택자로 범위를 좁힌다

`QFrame { border: ... }`처럼 클래스 전체에 걸면 의도하지 않은 위젯까지 바뀐다. **`QLabel`이 `QFrame`을 상속하기 때문**에 패널 테두리가 모든 라벨에도 붙게 된다 (Claude 보충 — 상속 관계는 Qt API 사실이고, 잘못 건 경우를 이 세션에서 재현하지는 않았다). 그래서 objectName을 쓰는 **ID 선택자**로 한정한다.

```css
QFrame#panelLight, QFrame#panelSensor {
  background-color: rgba(12, 22, 40, 215);   /* 알파는 0~255 */
  border: 1px solid #173a5e;
  border-radius: 12px;
}
QLabel#valRed { color: #ff2e63; font-size: 24px; font-weight: bold; }
```

Designer에서 붙인 **objectName이 여기서도 열쇠**다 → [[Qt-Designer-PyQt5-연결]]

## 세로 슬라이더

```css
QSlider::groove:vertical   { background: #081222; width: 10px; border-radius: 5px; }
QSlider::sub-page:vertical { background: transparent; }

QSlider#sliderRed::add-page:vertical {
  background: qlineargradient(x1:0, y1:0, x2:0, y2:1, stop:0 #ff2e63, stop:1 #3d0a1b);
  border-radius: 5px;
}
QSlider#sliderRed::handle:vertical {
  background: #ffe3ea; border: 2px solid #ff2e63;
  height: 12px; margin: 0px -8px; border-radius: 4px;
}
```

- ⚠️ **세로 슬라이더에서는 `add-page`가 손잡이 아래쪽(값이 차오르는 부분)** 이다. 렌더링해서 확인했다. 가로 슬라이더에서 채워진 쪽을 `sub-page`로 쓰던 감각과 반대라 헷갈린다.
- `handle`의 `margin: 0px -8px`은 손잡이를 홈보다 좌우로 넓게 튀어나오게 한다.
- 슬라이더마다 색을 다르게 하려면 `QSlider#sliderRed::add-page:vertical`처럼 ID와 하위 컨트롤을 함께 쓴다.

## 발광은 코드로

QSS에는 `box-shadow`가 없다. 네온 번짐은 **같은 색 그림자를 오프셋 0으로** 깔아서 만든다.

```python
def glow(widget, color, radius=22):
    effect = QtWidgets.QGraphicsDropShadowEffect(widget)
    c = QtGui.QColor(color)
    c.setAlpha(200)
    effect.setColor(c)
    effect.setBlurRadius(radius)
    effect.setOffset(0, 0)      # 오프셋이 0이면 그림자가 아니라 사방으로 번진다
    widget.setGraphicsEffect(effect)
```

- 위젯 하나에 그래픽 효과는 **하나만** 걸린다 (Claude 보충).
- QSS는 `letter-spacing`도 지원하지 않는다 (Claude 보충).

## 실행 중 색 바꾸기

상태에 따라 색이 바뀌는 위젯은 그 위젯에만 `setStyleSheet`를 다시 건다. 폼 전체 스타일은 그대로 두고 그 부분만 덮어쓴다.

```python
self.ui.lblLive.setText("● LIVE")
self.ui.lblLive.setStyleSheet("color: #00ff9c;")   # RETRY 면 노랑, NO SENSOR 면 빨강
```

## 글꼴과 한글

`font-family: "DejaVu Sans Mono"`처럼 한글이 없는 글꼴을 지정해도, 파이(리눅스)에서는 **한글 글자만 설치된 다른 글꼴(나눔)로 자동 대체**되어 표시됐다. 한글 글꼴 자체가 없으면 □□로 깨진다 → [[라즈베리파이-PyQt5-설치-ARM64]]

PC에서 미리보기할 때는 파이 전용 글꼴을 PC 글꼴로 바꿔 끼우면 실제와 가까워진다 → [[가짜-모듈로-하드웨어-없이-테스트]]

## 사용한 팔레트 (다크 네온)

| 용도 | 색 |
|---|---|
| 배경 그라데이션 | `#0b1120` → `#03050a` |
| 패널 테두리 | `#173a5e` |
| 강조(제목·SEND·습도) | `#00e5ff` |
| RED / GREEN / BLUE 채널 | `#ff2e63` / `#00ff9c` / `#3d8bff` |
| 온도 | `#ffcb47` |
| 보조 텍스트 | `#6f8bab`, `#3b5675` |

## 출처

Claude Code 세션 (2026-09-15). 조명·온습도 GUI를 사이버펑크풍 다크 네온 테마로 바꾸며 정리. PC 렌더링 캡처와 Raspberry Pi 4 실행으로 확인.

관련: [[Qt-Designer-PyQt5-연결]], [[Qt-Designer-위젯-승격]], [[가짜-모듈로-하드웨어-없이-테스트]]
