---
tags: [라즈베리파이, pyqt5, python, venv, arm64, 설치]
updated: 2026-09-08
---

# 라즈베리파이에 PyQt5 설치 (ARM64)

라즈베리파이에서 `pip install PyQt5`를 하면 안 된다. **PyQt5는 ARM64용 미리 빌드된 휠을 제공하지 않아** pip이 소스 빌드로 넘어가고, Qt 개발 파일이 없으면 한참 뒤 실패한다. `apt`의 `python3-pyqt5`를 쓰고, venv는 `--system-site-packages`로 만들어 그 패키지를 그대로 빌려 쓴다.

## 올바른 순서

```bash
sudo apt update
sudo apt install -y python3-pyqt5 python3-gpiozero python3-lgpio python3-venv

python3 -m venv --system-site-packages .venv     # 핵심: --system-site-packages
source .venv/bin/activate
python -c "import PyQt5, gpiozero; print('OK')"
```

`--system-site-packages`가 빠지면 venv가 시스템 패키지를 못 보고 `ModuleNotFoundError: No module named 'gpiozero'`가 난다. 수업 자료에서 자주 나오는 그 에러의 실체가 이것이다.

## 왜 apt인가

| 방식 | 결과 |
|---|---|
| `pip install PyQt5` | ARM64 휠 없음 → 소스 빌드 시도 → Qt 개발 파일 부재로 실패 (Claude 보충 — 휠 부재는 미검증, 이 세션에서는 apt로 우회) |
| `apt install python3-pyqt5` | 배포판이 빌드해 둔 바이너리를 즉시 설치 |

Debian 13 (trixie) 기준 실제 설치 버전:

- `python3-pyqt5` 5.15.11+dfsg-2
- `python3-gpiozero` 2.0.1-0+rpt1+trixie
- `python3-lgpio` 0.2.2-1~rpt1+trixie

## gpiozero 핀 팩토리

수업 자료 등에서 자주 보이는 `LGPIOFactory` 명시는 **Pi 5용 우회책**이다. Pi 5에서는 `RPi.GPIO`가 동작하지 않아 lgpio를 강제로 지정해야 했다.

```python
from gpiozero.pins.lgpio import LGPIOFactory
led = LED(13, pin_factory=LGPIOFactory())
```

Pi 4 + Bookworm/Trixie에서는 **gpiozero 기본 팩토리가 이미 lgpio**라 그냥 `LED(13)`으로 충분하다. 반대로 `RPi.GPIO`를 쓰는 예제 코드는 Pi 4에서는 그대로 동작하지만 Pi 5에서는 못 쓴다.

## 출처

Claude Code 세션 (2026-09-08), Raspberry Pi 4 Model B Rev 1.5 / Debian 13 (trixie). 수업 자료 `heartcom/Linux-Program`.

관련: [[라즈베리파이-GUI-SSH-VNC-실행]], [[gpiozero-GPIO-배선-확인]], [[Qt-Designer-PyQt5-연결]]
