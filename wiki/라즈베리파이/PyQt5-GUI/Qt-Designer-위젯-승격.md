---
tags: [qt, pyqt5, designer, promote, custom-widget, qpainter, 차트]
updated: 2026-09-15
---

# Qt Designer 위젯 승격 (Promote)

Designer 위젯 상자에 없는 **직접 만든 위젯**을 폼에 넣는 방법이다. 빈 `QWidget`을 자리 표시자로 놓고 "승격"으로 내 클래스를 지정하면, `pyuic5`가 그 자리에 내 클래스를 생성하는 코드를 만든다. 실시간 온습도 그래프를 matplotlib 없이 `QPainter`로 직접 그려서 이렇게 붙였다.

## Designer에서

1. 위젯 상자의 **Widget**(`QWidget`)을 원하는 자리에 놓고 objectName을 정한다 (`chartTemp`)
2. 우클릭 → **승격(Promote to...)**
3. 승격된 클래스 이름 `SparkChart`, 헤더 파일 `chart_widget` → 추가 → 승격

**헤더 파일 칸에는 파이썬 모듈 이름을 `.py` 없이** 쓴다. C++ 기준 UI라 "헤더"라고 부를 뿐이다.

(이 세션에서는 Designer 메뉴를 직접 조작하지 않고 `.ui`를 편집해 같은 결과를 만들었다. 메뉴 경로는 Claude 보충)

## `.ui`에 저장되는 모습

```xml
<widget class="SparkChart" name="chartTemp">
 <property name="minimumSize"><size><width>320</width><height>200</height></size></property>
</widget>
...
<customwidgets>
 <customwidget>
  <class>SparkChart</class>
  <extends>QWidget</extends>
  <header>chart_widget</header>
 </customwidget>
</customwidgets>
```

## `pyuic5`가 만드는 코드

```python
self.chartTemp = SparkChart(self.panelMonitor)
# ...
from chart_widget import SparkChart      # 생성 파일 맨 아래에 붙는다
```

- 그래서 **`chart_widget.py`가 실행 폴더에 같이 있어야** 한다. 파이로 옮길 때 빠뜨리면 import 단계에서 프로그램이 뜨지 않는다 (Claude 보충).
- 생성된 객체는 평범한 속성이라 `self.ui.chartTemp.add(24.6)`처럼 바로 쓴다.
- Designer는 내 클래스를 실행하지 않으므로 그래프 자리가 **빈 상자로만 보인다** (Claude 보충).

## 예제 — 실시간 라인 차트 `SparkChart`

추가 설치 없이 `QWidget.paintEvent`에서 `QPainter`로 그린다.

**데이터 보관** — `collections.deque(maxlen=N)`에 넣으면 오래된 값이 자동으로 밀려난다. 60개 × 3초 = 최근 3분.

**y축 범위** — 최소·최대에 15% 여유를 주되, 값이 거의 안 변할 때를 대비해 **최소 폭(`min_span`)** 을 보장한다. 이게 없으면 0.1°C의 흔들림이 화면 전체를 오르내리는 톱니처럼 과장된다. 온도는 2.0, 습도는 5.0으로 잡았다.

```python
lo, hi = min(values), max(values)
if hi - lo < min_span:
    mid = (hi + lo) / 2
    lo, hi = mid - min_span / 2, mid + min_span / 2
```

**그리는 순서**

1. 점선 격자, y축 값 라벨, x축 시간 라벨(`-3:00 · -1:30 · now`). 최신 값이 항상 오른쪽 끝에 온다.
2. 선 경로(`QPainterPath`)를 바닥까지 닫아서 위→아래로 투명해지는 `QLinearGradient`로 채운다.
3. **같은 경로를 굵고 옅은 선 → 가늘고 진한 선 순으로 여러 번** 그려서 네온 발광처럼 보이게 한다.
4. 최신 점에 반투명 원과 진한 점을 찍는다.

```python
for width, alpha in ((9, 35), (5, 70), (2, 255)):
    c = QtGui.QColor(self.color)
    c.setAlpha(alpha)
    pen = QtGui.QPen(c, width)
    pen.setCapStyle(QtCore.Qt.RoundCap)
    pen.setJoinStyle(QtCore.Qt.RoundJoin)
    p.setPen(pen)
    p.drawPath(line)
```

- 값이 하나도 없으면 격자 위에 `NO DATA`만 그린다.
- `add()` 안에서 `self.update()`를 불러 다시 그리게 한다.
- 센서 읽기에 **성공했을 때만** `add()`하므로 실패한 주기에는 점이 생기지 않는다.

## 레이아웃 늘림 비율

좌우 2단 배치에서 왼쪽 조작부는 폭을 고정하고 오른쪽 그래프 패널만 늘어나게 했다. 레이아웃 요소에 `stretch` 속성을 준다.

```xml
<layout class="QHBoxLayout" name="bodyLayout" stretch="0,1">
```

`pyuic5`는 이를 `self.bodyLayout.setStretch(1, 1)`로 변환한다.

## 출처

Claude Code 세션 (2026-09-15). 조명·온습도 GUI의 창을 오른쪽으로 넓혀 온도·습도 실시간 그래프("환경 모니터링") 패널을 추가하며 정리. PC 렌더링과 Raspberry Pi 4 실행으로 확인.

관련: [[Qt-Designer-PyQt5-연결]], [[Qt-스타일시트-QSS]], [[DHT11-온습도-센서]], [[가짜-모듈로-하드웨어-없이-테스트]]
