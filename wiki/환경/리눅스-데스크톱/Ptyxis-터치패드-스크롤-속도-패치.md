---
tags: [리눅스, GNOME, GTK4, VTE, Ptyxis, 터치패드, D-Bus]
updated: 2026-09-07
---

GNOME 기본 터미널 **Ptyxis**(GTK4 + VTE 0.84)는 터치패드 스크롤 속도를 조절하는 설정이 전혀 없다. 원인은 한 겹이 아니라 세 겹으로 겹쳐 있었고, 그중 하나(D-Bus 활성화)는 "래퍼 스크립트로 우회"라는 흔한 해법 자체를 무력화시킨다. VTE를 직접 패치해 해결했다.

## 증상

터치패드로 스크롤하면 셸 스크롤백과 Claude Code 같은 TUI 앱 안 모두 체감상 너무 빠름. `scroll-boost` 데몬(마우스 휠 배속용)은 무관 — 터치패드는 grab 대상에서 제외되어 있었다.

## 원인

Ptyxis/GTK4/VTE 조합이라 흔한 해법들이 순서대로 막힌다:

1. **설정 자체가 없음** — `org.gnome.Ptyxis` gsettings 스키마에 스크롤 속도 항목이 없고, Ptyxis가 `scroll-unit-is-pixels=true`를 하드코딩한다.
2. **LD_PRELOAD로 델타 가로채기 불가** — GTK4가 `-Bsymbolic`으로 빌드되어 있어 심볼 인터포지션이 안 먹힌다. VTE를 직접 패치/재빌드해야 한다.
3. **경로가 두 개로 갈라져 있고 단위가 다르다:**
   - **셸 스크롤백**: Ptyxis가 `enable-fallback-scrolling=false`로 두기 때문에 VTE가 스크롤 이벤트를 자체 처리하지 않고 `return false`하며, 부모 **GtkScrolledWindow**가 원본(무보정) 픽셀 델타로 스크롤하고 관성(kinetic) 감속까지 붙인다. VTE의 `Widget::event_scroll`만 패치해서는 이 경로에 전혀 영향이 없다 — VTE가 애초에 관여하지 않기 때문.
   - **마우스 보고(mouse-reporting) 켜진 앱** (Claude Code, vim, htop 등 TUI): 스크롤이 앱으로 전달되고, 휠 노치 1개 → 앱의 줄 단위 1개로 변환된다. 실측 델타는 이벤트당 5~21px인데 그 안의 매 "1단위"마다 휠 프레스가 1회 나가므로, 스크롤백 대비 체감 속도가 훨씬 빠르고 계단(스텝) 현상도 생긴다.
   - 두 경로는 단위가 완전히 다르므로(스크롤백=픽셀, 앱=휠 프레스 1회) 배율을 하나로 통일하면 반드시 한쪽이 망가진다.
4. **"부드러움"의 원인**: VTE는 스크롤백을 `round(scroll_delta * cell_height)`로 렌더링해 행 이하 픽셀 단위까지 부드럽게 그린다. 반면 앱 경로는 휠 노치→줄 단위로 두 번 양자화되므로 **프로토콜 구조상 부드럽게 만들 수 없다.**
5. **래퍼 스크립트가 로드되지 않던 이유**: Ptyxis는 `.desktop`에 `DBusActivatable=true`가 설정돼 있어, GNOME이 `/usr/share/dbus-1/services/org.gnome.Ptyxis.service`에 적힌 `/usr/bin/ptyxis`를 **D-Bus로 직접 활성화**한다. `~/.local/bin/ptyxis` 같은 PATH 래퍼나 `.desktop`의 `Exec` 오버라이드는 전혀 거치지 않는다. 이 조건에서 커스텀 빌드/래퍼를 적용하려면 **D-Bus 서비스 파일 자체를 `~/.local/share/dbus-1/services/`에 덮어써서** 그 안의 `Exec=`가 래퍼를 가리키게 해야 한다.

## 해결

1. **VTE 소스 패치 + 재빌드** (`vte-0.84.0`): `Widget::event_scroll`을 확장해 스크롤백 케이스에서 VTE가 vadjustment를 직접(배율 적용해) 움직이고 이벤트를 소비하도록 함. 부수 효과로 GtkScrolledWindow의 관성 감속도 이 경로에서 빠진다.
2. **배율 3개로 분리**:
   - `TOUCHPAD_SCROLL_FACTOR` — 셸 스크롤백 속도 (검증된 값: 0.4)
   - `TOUCHPAD_APP_SCROLL_FACTOR` — 대체 화면(alternate screen) 쓰는 앱(vim/less/htop 등, 스크롤백 없음) 속도 (0.05)
   - `TOUCHPAD_BYPASS_APP` — 마우스 보고 켜진 일반 화면 앱(Claude Code 등)에서 터치패드 스크롤을 아예 앱으로 넘기지 않고 터미널 스크롤백(픽셀 단위)으로 돌림. 앱이 출력을 스크롤백에 그대로 남기므로 보이는 내용은 동일하고 스크롤만 완전히 부드러워진다. 마우스 휠은 손대지 않음(휠은 여전히 앱으로 감, scroll-boost 배속 유지).
3. **설정 라이브 리로드**: `~/.config/ptyxis-scroll.conf`를 0.2초 주기로 다시 읽어 재시작 없이 값 변경이 즉시 반영되게 함.
4. **D-Bus 활성화 오버라이드**: `~/.local/share/dbus-1/services/org.gnome.Ptyxis.service`를 만들어 `Exec=`가 패치된 라이브러리를 로드하는 래퍼(`~/.local/bin/ptyxis`)를 가리키도록 함. `.desktop` 오버라이드도 함께 추가.

제어 명령(`term-scroll-speed`, 재시작 불필요 — conf 라이브 리로드 덕분):
```
term-scroll-speed 0.3        # 셸 스크롤백 배율
term-scroll-speed app 0.05   # vim/less/htop 등 대체 화면 앱 배율
term-scroll-speed bypass off # 마우스 보고 앱에 터치패드 스크롤을 다시 넘김 (되돌리기)
```

완전 스톡으로 되돌리려면: `rm ~/.local/share/dbus-1/services/org.gnome.Ptyxis.service` 후 Ptyxis 재시작.

## 한계 / 주의

- **새로 빌드한 라이브러리는 Ptyxis 프로세스 시작 시점에만 로드**된다 — 설정값(배율)은 라이브 리로드되지만, 라이브러리 자체를 바꾼 뒤에는 `pkill -x ptyxis` 후 재시작이 필요하다.
- 현재 터미널에서 `pkill -x ptyxis`를 실행하면 그 터미널 자식 프로세스인 Claude Code 세션도 함께 종료된다.
- vim/less/htop처럼 대체 화면(alternate screen)을 쓰는 앱은 스크롤백이 없으므로 bypass 대상이 아니며 `TOUCHPAD_APP_SCROLL_FACTOR`로만 조절 가능하다.
- 검증은 마우스 보고 상태를 재현한 별도 디버그 창 + 디버그 로그(스크롤 이벤트 수, 델타, 적용된 배율)로 했다 — 실제 터미널을 직접 조작하며 검증하면 그 세션 자체가 끊길 위험이 있기 때문.

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory), 2026-09-07.
