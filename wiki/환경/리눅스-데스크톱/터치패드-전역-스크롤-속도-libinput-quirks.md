---
tags: [리눅스, GNOME, libinput, 터치패드, Wayland]
updated: 2026-10-15
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

## Chrome 전용 스크롤 배율 (2026-10-12 세션)

전역 값(`AttrResolutionHint`)을 건드리면 커서·제스처까지 변하므로, **Chrome만** 배율을 따로 조절하는 방법.

- Chrome에는 자체 스크롤 속도 설정이 없다. 기존 선택지는 1배(기본)와 1.5배(터치패드 이동량 고정 해제)뿐이었다.
- 그래서 **content script 확장** `~/.local/share/chrome-scroll-boost`(이름 *Touchpad Scroll Boost*)를 만들어 `content.js`의 `FACTOR`로 배율을 곱한다. 터치패드 휠 이벤트에만 적용되고 USB 마우스 휠(scroll-boost)과는 겹치지 않으며, 핀치줌·두 손가락 뒤/앞으로 스와이프는 Chrome 기본 그대로. 편집기·지도처럼 스크롤을 직접 처리하는 사이트는 건너뛴다.
- **설치**: `chrome://extensions` → 개발자 모드 → 압축해제된 확장 프로그램 로드. 외부(CLI·브라우저 도구)에서 `chrome://` 페이지를 열 수 없어 사용자가 직접 해야 한다(주소를 클립보드에 넣어 Ctrl+T, Ctrl+V). 최신 Chrome은 CLI로 확장 로드도 막는다.
- **배율 변경**: `FACTOR` 수정 후 확장 카드의 새로고침(↻) + 열린 탭 F5.
- 검증: 위키백과에서 스크롤 40을 보내 50 이동 = 정확히 1.25배. 사용자 체감으로 **1.35배**로 확정.

### 뒤로가기 스와이프를 덜 밀어도 되게
- 원인: Chrome의 '터치패드 이동량 1배 고정' 옵션이 켜져 있어 스와이프 거리 요구가 큼. 이 옵션을 끄면 필요한 이동량이 약 2/3로 줄지만 **세로 스크롤도 1.5배** 빨라진다.
- 그래서 확장 `FACTOR`를 1.35 → **0.9**로 낮춰 체감 속도를 유지(0.9×1.5=1.35). 옵션은 Chrome 실행 플래그(`.desktop` 파일)로 바꿨고 Chrome 완전 재시작 후 적용. 효과는 사용자 확인 전.
- 스크롤을 직접 처리하는 사이트는 확장이 개입하지 않아 1.5배 그대로.
- 그래도 빡빡하면 대안: 확장이 직접 history back 실행(적게 밀어도 되지만 화살표 애니메이션 사라짐).
- 되돌리기: 백업 `~/.local/share/applications/google-chrome.desktop.bak-20261012-unscaled`를 덮어쓰고 `FACTOR`를 1.35로 복구.

## 관련: 스크롤 경로 4종

이 vault에서 확인된 스크롤 관련 경로는 서로 완전히 독립적이다 — 하나를 고쳐도 나머지에 영향 없음:

- **터치패드 전역** (이 페이지): libinput quirks의 `AttrResolutionHint`, `pad-scroll-speed`로 제어. 터치패드를 쓰는 모든 앱에 적용.
- **마우스 휠**: `scroll-boost` 데몬이 특정 USB 마우스 장치를 grab해서 배속. 대상 마우스가 물리적으로 연결돼 있어야 동작(연결 안 돼 있으면 서비스는 떠 있어도 아무 효과 없음 — grab할 장치가 없기 때문).
- **Chrome 전용 확장**: 위 "Chrome 전용 스크롤 배율" 절. Chrome에만 곱해짐.
- **터미널(Ptyxis)**: 패치된 VTE 라이브러리 경로, `term-scroll-speed`로 제어. 자세한 내용은 [[Ptyxis-터치패드-스크롤-속도-패치]].

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-08.
Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory), 2026-10-15.
