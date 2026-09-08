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

## 출처

Claude Code 세션 (2026-09-08), Raspberry Pi 4 Model B Rev 1.5 / Debian 13 (trixie) / labwc.

관련: [[라즈베리파이-PyQt5-설치-ARM64]], [[Qt-Designer-PyQt5-연결]]
