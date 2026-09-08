---
tags: [라즈베리파이, pyqt5, 워크플로, ssh, scp, vnc, designer]
updated: 2026-09-08
---

# 라즈베리파이 GUI 개발 워크플로

**PC에서 만들고, 파이에서 돌린다.** UI는 PC의 Qt Designer로 그려 `pyuic5`로 변환까지 끝내고, 파일만 파이로 넘겨 실행한다. 파이에서 Designer를 띄워 작업하는 것보다 빠르고, PC에서 미리 검증할 수 있는 범위도 넓다.

## 전체 흐름

```
[PC]  Designer로 .ui 작성 → pyuic5로 .py 변환 → 동작 코드 작성
  ↓   scp (또는 VS Code Remote-SSH 드래그)
[Pi]  venv 준비 (apt 패키지 사용) → VNC 데스크톱에서 실행
```

역할이 셋으로 갈린다. **어디서 무엇을 하는지 헷갈리는 게 이 워크플로의 주된 실수 원인**이다.

| 작업 | 위치 | 프롬프트 |
|---|---|---|
| UI 작성, 변환, 코드 편집 | PC | `PS C:\Users\...>` |
| 파일 전송 (`scp`) | **PC** | `PS C:\Users\...>` |
| 패키지 설치, 파일 확인 | 파이 (SSH) | `pi@호스트:~ $` |
| **GUI 실행** | 파이 (**VNC**) | `pi@호스트:~ $` |

## 1. PC — UI와 코드

```bash
designer.exe                                  # .ui 작성, objectName 지정
pyuic5 -x light_gui.ui -o light_gui.py        # → Ui_Dialog 클래스 생성
```

자세한 내용과 함정은 [[Qt-Designer-PyQt5-연결]].

GPIO가 없는 PC에서도 **가짜 `gpiozero` 모듈을 끼워 넣으면 로직을 검증할 수 있다.** 위젯 연결, 시그널, 값 매핑이 맞는지 파이로 옮기기 전에 확인된다.

```python
import sys, types
fake = types.ModuleType("gpiozero")
class PWMLED:
    def __init__(self, pin, **kw): self.pin, self.value = pin, 0.0
fake.PWMLED = PWMLED
sys.modules["gpiozero"] = fake          # 실제 모듈보다 먼저 등록

import lightRun                          # 이제 GPIO 없이 import 된다
```

이때 `QT_QPA_PLATFORM=offscreen`으로 두면 창을 띄우지 않고 `widget.grab().save(...)`로 레이아웃 그림까지 뽑을 수 있다. 단 **앱 코드가 `os.environ`으로 플랫폼을 덮어쓰면 import 이후에 다시 지정해야 한다** → [[라즈베리파이-GUI-SSH-VNC-실행]]

## 2. 전송 — PC에서 실행한다

```powershell
scp -r $env:USERPROFILE\OneDrive\Desktop\LIGHT_GUI pi@192.168.1.9:~/work/
```

`scp`는 PC에서 파이로 미는 명령이라 **PC 셸에서** 친다. 파이에 접속된 창에 그대로 붙여 넣으면 리눅스 셸이 `$env:USERPROFILE`를 이해하지 못하고 아래처럼 문자 그대로 처리한다.

```
scp: stat local ":USERPROFILEOneDriveDesktopLIGHT_GUI": No such file or directory
```

반대로 PowerShell에서는 `&&`가 파서 오류를 낸다(Windows PowerShell 5.1). 명령을 이을 때는 `;`를 쓴다.

VS Code Remote-SSH로 파이 폴더를 열어 **드래그 앤 드롭**하는 방법이 가장 헷갈릴 일이 없다.

## 3. 파이 — 설치와 실행

설치는 SSH에서 해도 된다. 패키지 구성은 [[라즈베리파이-PyQt5-설치-ARM64]].

**GUI 실행만은 VNC 데스크톱 안의 터미널에서** 해야 한다. SSH 터미널은 `DISPLAY`가 비어 있어 창을 띄울 수 없다 → [[라즈베리파이-GUI-SSH-VNC-실행]]

웹 서버(Flask 등)처럼 창이 없는 프로그램은 SSH에서 실행해도 되고, 브라우저로 접속해 쓰면 된다.

## 자주 막히는 지점

| 증상 | 원인 |
|---|---|
| `could not connect to display` | SSH 터미널에서 GUI 실행 |
| `ModuleNotFoundError: gpiozero` | venv를 `--system-site-packages` 없이 생성 |
| `lgpio.error: 'GPIO busy'` | 같은 핀을 쓰는 프로그램이 이미 실행 중 |
| `AttributeError` / `Ui_` 클래스 없음 | Designer에서 폼 objectName을 잘못 변경 |
| LED가 안 켜짐 | 배선 핀 번호 불일치 (BCM ≠ 물리 핀) |
| `scp: stat local ":USERPROFILE..."` | PC 명령을 파이 터미널에서 실행 |

## 출처

Claude Code 세션 (2026-09-08). 수업 자료 `heartcom/Linux-Program` 2번 PPT 실습을 Windows PC + Raspberry Pi 4로 진행하며 정리. LED ON/OFF GUI와 RGB 밝기 조절 GUI 두 개를 이 흐름으로 만들어 동작 확인.

관련: [[Qt-Designer-PyQt5-연결]], [[라즈베리파이-PyQt5-설치-ARM64]], [[라즈베리파이-GUI-SSH-VNC-실행]], [[gpiozero-GPIO-배선-확인]], [[gpiozero-PWMLED-밝기-제어]]
