---
tags: [라즈베리파이, dht11, 센서, adafruit, gpio, 온습도]
updated: 2026-09-15
---

# DHT11 온습도 센서

온도·습도를 **디지털 신호선 한 가닥(DATA)** 으로 보내는 저가 센서. Raspberry Pi 4 + Debian 13(trixie) + Python 3.13에서는 `adafruit-circuitpython-dht`를 venv에 pip로 설치하고 **`use_pulseio=False`로 읽는 조합**이 동작했다. `DHT sensor not found`는 대부분 배선 문제이고, 핀을 입력 모드로 읽어 보면 어디가 잘못 꽂혔는지 찾을 수 있다.

## 설치

venv는 `--system-site-packages`로 만든 것을 전제로 한다 → [[라즈베리파이-PyQt5-설치-ARM64]]

```bash
source .venv/bin/activate
python -m pip install adafruit-blinka adafruit-circuitpython-dht
python -c "import board, adafruit_dht; print(board.D5)"   # 핀 번호(5)가 찍히면 준비 완료
```

이 과정에서 pip이 `RPi.GPIO 0.7.1`과 `rpi_ws281x`를 **소스에서 빌드**해 venv에 넣는다. 몇 분 걸린다. 설치된 버전은 `adafruit-circuitpython-dht 4.0.12`, `adafruit-blinka 9.2.0`.

## 읽기

```python
import board, adafruit_dht

sensor = adafruit_dht.DHT11(board.D5, use_pulseio=False)
try:
    t, h = sensor.temperature, sensor.humidity
except RuntimeError as e:
    ...   # 실패는 흔하다. 다음 주기에 다시 읽는다
```

| 읽기 방식 | 이 환경 결과 |
|---|---|
| `use_pulseio=False` (비트뱅) | **동작** |
| `use_pulseio=True` (기본값) | `Unable to set line 4 to input`으로 실패 |

- `RuntimeError`는 재시도 대상으로 취급하고, 그 밖의 예외만 진짜 오류로 본다.
- 읽기 간격은 3초로 운용했다. 2초 미만으로 읽으면 안 된다는 것은 데이터시트 권장값이다 (Claude 보충).
- GUI에서는 `QTimer`로 주기 호출하고, **라이브러리 import나 센서 초기화가 실패해도 나머지 기능(조명 등)은 살아 있게** 센서를 선택 기능으로 둔다.

## 에러 메시지로 원인 구분

| 메시지 | 의미 |
|---|---|
| `DHT sensor not found, check wiring` | 센서에서 **응답이 전혀 없음**. 배선·전원 문제. 이 세션에서 실제로 겪음 |
| `Unable to set line N to input` | 읽기 방식이 커널과 안 맞음 → `use_pulseio=False`. 실제로 겪음 |
| `Checksum did not validate` | 신호는 오는데 비트가 깨짐. 재시도하면 되는 흔한 실패 (Claude 보충 — 이 환경에서는 관찰하지 못함) |

## 배선

3핀 모듈(작은 기판 위 파란 격자 센서)은 **기판에 인쇄된 글자를 보고** 연결한다. 제조사마다 핀 순서가 다르다.

| 모듈 표시 | 파이 |
|---|---|
| `+` / VCC | **3.3V** — 물리 1번 또는 17번 |
| `S` / DATA | 아무 GPIO — 여기선 GPIO5 (물리 29번) |
| `-` / GND | GND — 브레드보드 GND 줄을 LED와 공유해도 된다 |

- 3핀 모듈은 풀업 저항이 기판에 달려 있다. 다리 4개짜리 생짜 센서는 DATA–VCC 사이에 10kΩ을 따로 달아야 한다 (Claude 보충).
- ⚠️ **VCC를 5V(물리 2·4번)에 꽂지 않는다.** 모듈 풀업이 DATA 선을 VCC 전압까지 끌어올리므로, 5V로 주면 3.3V 입력인 GPIO에 5V가 걸려 손상될 수 있다 (Claude 보충). **1번(3.3V) 바로 옆이 2번(5V)** 이라 한 칸만 틀려도 이렇게 된다 → [[GPIO-기초]]
- 수업 자료는 Pi 5 기준 `board.D4`(GPIO4)이지만, 핀은 비어 있는 아무 GPIO나 쓰고 코드의 핀 이름만 맞추면 된다. LED 선과 부딪혀서 DATA를 29번(GPIO5)으로 옮겼다 → [[gpiozero-GPIO-배선-확인]]

## 배선 진단 — 풀다운을 걸고 읽는다

`sensor not found`일 때는 핀을 **입력 모드 + 내부 풀다운**으로 읽는다. 출력을 쓰지 않으므로 안전하다.

```python
from gpiozero import DigitalInputDevice
import time
for p in [4, 5, 6, 12, 13, 16, 18, 19, 20, 21, 23, 24, 25, 26]:
    d = DigitalInputDevice(p, pull_up=False)
    time.sleep(0.05)
    print("GPIO%d =" % p, d.value)
    d.close()
```

아무것도 없는 핀은 풀다운 때문에 0, **모듈 DATA가 붙은 핀만 모듈 풀업 때문에 1**이 된다. 실행 중인 GUI가 핀을 잡고 있으면 `GPIO busy`가 나므로 먼저 종료한다.

실제 사례:

1. 7번 핀(GPIO4)에 꽂았다고 생각했는데 `GPIO4 = 0`, 대신 **`GPIO18`·`GPIO23` 두 개가 1**
2. 두 핀 각각에서 읽어도 모두 `sensor not found`
3. 둘 다 짝수(바깥) 줄 핀이라 선이 줄을 잘못 탄 것으로 보고 전원을 끈 뒤 다시 꽂음 → **`GPIO5 = 1`, 나머지 0**
4. 그 상태에서 읽기 성공

HIGH가 **두 개**였다는 점이 단서였다. 정상이라면 DATA 하나만 1이어야 한다. 그 원인을 "GND 선이 GPIO에 꽂혀 센서가 접지되지 않았다"고 추정했지만 물리적으로 확인하지는 않았다.

## 출처

Claude Code 세션 (2026-09-15), Raspberry Pi 4 Model B Rev 1.5 / Debian 13 (trixie) / Python 3.13. 수업 자료 `heartcom/Linux-Program` 2번 PPT(슬라이드 7~9) 실습을 조명 GUI에 통합하며 확인.

관련: [[gpiozero-GPIO-배선-확인]], [[GPIO-기초]], [[라즈베리파이-PyQt5-설치-ARM64]], [[라즈베리파이-GUI-개발-워크플로]], [[Qt-Designer-위젯-승격]]
