---
tags: [라즈베리파이, vnc, wayvnc, wayland, ssh, qt, gui]
updated: 2026-09-08
---

# 라즈베리파이 GUI 앱을 SSH가 아닌 VNC에서 실행

GUI 프로그램은 **SSH 터미널에서 실행되지 않는다.** 붙을 화면이 없기 때문이다. 라즈베리파이 데스크톱에 VNC로 접속해 그 안의 터미널에서 실행해야 한다. Debian 13은 데스크톱이 Wayland(labwc)라 VNC 서버도 `wayvnc`다.

## 증상

```
qt.qpa.xcb: could not connect to display
qt.qpa.plugin: Could not load the Qt platform plugin "xcb" in "" even though it was found.
Available platform plugins are: eglfs, linuxfb, minimal, ..., wayland, xcb.
Aborted
```

플러그인이 **"found"인데 실패**한다는 게 단서다. 플러그인이 없는 게 아니라 연결할 디스플레이가 없는 것이다.

## 어느 터미널인지 판별

```bash
echo "DISPLAY=[$DISPLAY]  WAYLAND_DISPLAY=[$WAYLAND_DISPLAY]"
```

- 둘 다 비어 있음 → **SSH 터미널**. GUI 실행 불가.
- `WAYLAND_DISPLAY=[wayland-0]` 등 값이 있음 → 데스크톱 세션 터미널. 실행 가능.

VS Code Remote-SSH의 내장 터미널도 결국 SSH라 마찬가지로 비어 있다. 파일 전송·설치·코드 편집에는 좋지만 **실행은 VNC 창에서** 해야 한다.

## ⚠️ 코드가 셸 환경변수를 덮어쓰는 함정

예제 코드 상단에 이런 줄이 흔히 있다.

```python
import os
os.environ["QT_QPA_PLATFORM"] = "xcb"    # import 시점에 무조건 덮어씀
```

이게 있으면 셸에서 `QT_QPA_PLATFORM=wayland python app.py`로 넘겨도 **아무 소용이 없다.** 파이썬이 모듈을 import 하면서 값을 다시 xcb로 밀어버리기 때문이다. 플랫폼을 바꿔 시도해보려면 이 줄 자체를 지워야 한다.

```bash
sed -i '/QT_QPA_PLATFORM/d' app.py     # Qt가 세션에 맞는 플랫폼을 자동 선택하게 둔다
```

Wayland 세션에서는 이 줄을 지우는 것만으로 정상 실행됐다. (xcb 강제가 Wayland 세션 *안에서도* 실패하는지는 확인하지 않았다 — XWayland가 있으면 동작할 수도 있다.)

## wayvnc 활성화와 접속

```bash
sudo raspi-config      # Interface Options → VNC → Yes
systemctl is-active vncserver-x11-serviced wayvnc
```

Debian 13에서는 `wayvnc` 쪽이 `active`가 된다 (RealVNC의 `vncserver-x11-serviced`는 `inactive`). 뷰어에서 `<IP>:5900`으로 접속한다. UltraVNC Viewer 1.8.2로 접속 확인.

`wayvnc`는 **이미 떠 있는 데스크톱 세션을 중계**하는 방식이라, 파이가 콘솔로 부팅되어 로그인된 세션이 없으면 붙을 게 없다. 그때는 `raspi-config` → `System Options` → `Boot / Auto Login` → `Desktop Autologin`으로 바꾼다.

파이에 모니터가 직접 연결돼 있다면 VNC 없이 그 화면에서 바로 실행하면 된다.

## SSH · SCP · VNC는 서로 다른 통로다

셋 다 같은 파이에 붙지만 하는 일이 다르다. **VNC는 저장 공간이 아니라 화면**이라는 점이 특히 헷갈린다.

| 통로 | 하는 일 | 파일을 저장하나 |
|---|---|---|
| SSH | 명령을 주고받는다 (글자만) | 아니오 |
| SCP | 파일을 복사한다 | 아니오 (옮기기만) |
| VNC | 데스크톱 화면을 중계한다 | 아니오 |
| 라즈베리파이 | **컴퓨터 (디스크 있음)** | **예** |

파일은 파이 디스크 **한 곳**에만 있다. SSH로 보든 VNC로 보든 같은 `~/work/...`가 보인다. "VNC 안에 파일이 들어갔다"는 건 오해이고, `scp`가 파이 디스크에 넣은 것이다.

확인 방법: VNC를 완전히 닫고 SSH에서 `ls ~/work/...`를 쳐보면 파일이 그대로 있다.

VNC 창에서 GUI를 조작하면, 입력이 파이로 전달되어 **파이 안의 프로그램이** 반응하고 그 화면이 되돌아온다. PC는 모니터와 마우스 역할만 한다. 그래서 **VNC 창을 닫아도 프로그램은 파이에서 계속 돌아간다** — 창이 안 보이는데 핀이 잡혀 있는 `GPIO busy`의 흔한 원인이다.

## PC 터미널과 파이 터미널을 구분한다

디스플레이 유무와 별개로, **명령을 어느 기계에서 치는지**도 자주 헷갈린다. 프롬프트로 판별한다.

| 프롬프트 | 어디 | 쓰는 명령 |
|---|---|---|
| `PS C:\Users\...>` | PC (PowerShell) | `ssh`, `scp`, `$env:USERPROFILE` |
| `pi@호스트:~ $` | 파이 (bash) | `apt`, `python`, `~/work/...` |

`scp`는 PC에서 파이로 미는 명령이라 **PC 쪽에서** 친다. 파이 창에 붙여 넣으면 리눅스 셸이 PowerShell 변수를 문자 그대로 처리한다.

```
scp: stat local ":USERPROFILEOneDriveDesktopLIGHT_GUI": No such file or directory
```

반대 방향 실수도 있다. Windows PowerShell 5.1은 `&&`를 지원하지 않아 리눅스용 명령을 그대로 붙여 넣으면 파서 오류가 난다. PowerShell에서는 `;`로 잇는다.

## 출처

Claude Code 세션 (2026-09-08), Raspberry Pi 4 Model B Rev 1.5 / Debian 13 (trixie) / labwc.

관련: [[라즈베리파이-PyQt5-설치-ARM64]], [[Qt-Designer-PyQt5-연결]]
