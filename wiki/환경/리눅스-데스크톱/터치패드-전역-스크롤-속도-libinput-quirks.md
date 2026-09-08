---
tags: [리눅스, GNOME, libinput, 터치패드, Wayland]
updated: 2026-09-08
---

Chrome 등 **모든 앱**에서 터치패드 스크롤이 느릴 때의 원인과 해결. [[Ptyxis-터치패드-스크롤-속도-패치]]와는 다른 경로다 — 그쪽은 터미널(VTE)에만 걸리는 별도 패치이고, 이 페이지는 libinput 레벨이라 터치패드를 쓰는 모든 앱에 공통으로 적용된다.

## 증상

터치패드로 스크롤하면 Chrome을 포함한 일반 앱 전체가 느림. GNOME 50 Wayland에는 터치패드 스크롤 배율을 조절하는 gsettings 항목이 아예 없다.

## 원인

`/etc/libinput/local-overrides.quirks`에 이미 들어 있던 설정(2026-06-15 작성):
```
[ZNT0001 Touchpad slower scrolling]
AttrResolutionHint=240x240
```
libinput은 스크롤 양을 "손가락이 몇 mm 움직였나"로 계산한다. `AttrResolutionHint`로 터치패드 해상도를 실제보다 높게(240/mm) 속이면 같은 물리적 이동거리가 더 작은 mm로 계산되어 스크롤이 느려진다. 6월에 의도적으로 절반 속도로 낮춰둔 값이 원인이었다.

이 터치패드(ZNT0001)의 실제 네이티브 해상도는 **120/mm**다 — 커널이 보고하는 물리 크기는 144x102mm인데, `AttrResolutionHint=240`일 때 libinput이 계산한 논리 크기가 145x102mm로 나온 것으로 역산 확인. 즉 240은 정확히 "2배 느리게"였고 120이 스톡(원래) 속도다.

**부작용**: 같은 `AttrResolutionHint` 값이 스크롤뿐 아니라 **포인터 이동 속도**와 **3손가락 제스처 임계값**에도 걸린다 — 전부 mm 기준 계산이기 때문. 스크롤을 2배 빠르게 하면 커서도 2배 빨라지고 워크스페이스 스와이프 민감도도 바뀐다.

## 해결

`~/.local/bin/pad-scroll-speed` 스크립트(기존 `scroll-factor`/`term-scroll-speed`와 같은 패턴)로 `AttrResolutionHint` 값을 바꾸고, 터치패드 장치를 unbind/bind로 재생성해 로그아웃 없이 즉시 적용한다(적용 중 1~2초간 터치패드 입력 끊김, 터치스크린으로 복구 가능). 파일 수정에 루트 권한이 필요해 `pkexec`로 실행:

```
pkexec bash ~/.local/bin/pad-scroll-speed 120     # 스톡 속도 (240의 절반 → 2배 빠르게)
pkexec bash ~/.local/bin/pad-scroll-speed 90       # 스톡보다 더 빠르게
pkexec bash ~/.local/bin/pad-scroll-speed reset    # 원래 값(240)으로 되돌림
```

적용 후 커서가 너무 빨라졌으면 터치패드 속도로 보정:
```
gsettings set org.gnome.desktop.peripherals.touchpad speed -0.3
```

## 관련: 스크롤 경로 3종

이 vault에서 확인된 스크롤 관련 경로는 서로 완전히 독립적이다 — 하나를 고쳐도 나머지에 영향 없음:

- **터치패드 전역** (이 페이지): libinput quirks의 `AttrResolutionHint`, `pad-scroll-speed`로 제어. 터치패드를 쓰는 모든 앱에 적용.
- **마우스 휠**: `scroll-boost` 데몬이 특정 USB 마우스 장치를 grab해서 배속. 대상 마우스가 물리적으로 연결돼 있어야 동작(연결 안 돼 있으면 서비스는 떠 있어도 아무 효과 없음 — grab할 장치가 없기 때문).
- **터미널(Ptyxis)**: 패치된 VTE 라이브러리 경로, `term-scroll-speed`로 제어. 자세한 내용은 [[Ptyxis-터치패드-스크롤-속도-패치]].

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-08.
