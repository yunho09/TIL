---
tags:
  - gnome
  - wayland
  - mutter
  - wl-clipboard
  - claude-code
  - linux
updated: 2026-09-06
---

# GNOME/Wayland: 터미널 Claude에 이미지 붙여넣을 때마다 뜨는 "wl-clipboard 앱이 준비되었습니다" 토스트

GNOME Wayland(Mutter 50) 환경에서 터미널의 Claude Code CLI에 이미지를 붙여넣거나 복사할 때마다 출처를 알 수 없는 "«앱»이 준비되었습니다" 알림이 뜨는 문제. 원인은 Mutter 50에 `wl-clipboard`가 기대하는 클립보드 프로토콜이 아예 없어 폴백 경로를 타는 것과, `focus-new-windows`가 `strict`로 박혀 있던 설정이 겹친 것이었다.

## 원인 체인

1. **Mutter 50에 `data-control` 계열 프로토콜이 없다** — `libmutter-18.so.0`에 `zwlr_data_control` / `ext_data_control` 문자열이 0개. 이 프로토콜이 있어야 `wl-copy`/`wl-paste`가 백그라운드에서 조용히 클립보드를 읽고 쓸 수 있다.
2. **그래서 `wl-clipboard`가 폴백 경로를 탄다** — 보이지 않는 `xdg_toplevel`을 만들고 `set_app_id("wl-clipboard")`를 호출한 뒤 `xdg_activation_v1`로 포커스를 요청한다(바이너리 `strings`로 확인 가능).
3. **`org.gnome.desktop.wm.preferences focus-new-windows`가 `strict`였다** — 스키마 기본값은 `smart`인데 dconf에 `strict`로 직접 박혀 있었다. `strict`면 Mutter가 이 포커스 요청을 거부하고, 대신 GNOME Shell이 "«앱»이 준비되었습니다" 토스트를 띄운다. 요청한 앱에 `.desktop` 파일이 없어서 알림에 출처가 "알 수 없음"으로 뜨는 것도 이 메커니즘과 일치한다.

**누가 호출하는가** — 이 머신에서 `wl-copy`/`wl-paste`를 실행하는 프로세스는 Claude Code CLI(`~/.local/share/claude/versions/<버전>/`)뿐이었다(`wl-paste -l`, `wl-paste --type image/png`, `wl-copy` 문자열 확인). Claude Desktop·Clawd·ptyxis에는 해당 호출이 없다. 즉 **터미널 Claude에 이미지를 붙여넣거나 복사할 때마다** 한 번씩 뜨는 것이었다.

## 해결

`focus-new-windows`를 기본값 `smart`로 되돌리면 된다. 설정은 즉시 적용되며 재로그인 불필요.

```bash
gsettings set org.gnome.desktop.wm.preferences focus-new-windows smart
```

검증은 `wl-paste -l`을 실행해 동일한 폴백 경로(숨은 `xdg_toplevel` + `xdg_activation_v1` 포커스 요청)를 인위적으로 트리거해보는 방식으로 했다(읽기 전용 나열이라 클립보드 내용은 바뀌지 않는다). `smart`로 되돌린 뒤에는 토스트가 뜨지 않는 것을 확인했다.

재발 시(예: [[claude-desktop-focus-steal|Claude Desktop이 워크스페이스를 끌고 가는 문제]]로 다시 `strict`가 필요해지면) 되돌리는 한 줄:

```bash
gsettings set org.gnome.desktop.wm.preferences focus-new-windows strict
```

## 주의점

`strict`는 과거 Claude Desktop이 포커스를 훔쳐 워크스페이스를 끌고 가는 문제 때문에 넣었던 것으로 보이나, 그 문제는 최종적으로 Claude Desktop 실행 옵션에 `--ozone-platform=x11`을 붙이는 것으로 해결되었고 이미 적용되어 있었다. 즉 `strict`는 이미 다른 방법으로 해결된 문제의 잔재 설정이었고, 지우고 나서 부작용이 없었다.

## 관련 시행착오: ws-monitor.sh 좀비 프로세스

같은 세션에서, 과거 포커스 디버깅용으로 띄워둔 `ws-monitor.sh`(0.1초 간격으로 `wmctrl -d` 폴링, `/tmp/claude-*/.../scratchpad/` 아래) 스크립트가 3일 22시간째 돌고 있던 것을 발견해 종료했다. 디버깅용 임시 스크립트를 백그라운드로 띄울 때는 조사가 끝나면 반드시 종료 여부를 확인해야 한다는 교훈.

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-06.
