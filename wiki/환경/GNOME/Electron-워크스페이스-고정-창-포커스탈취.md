---
tags: [electron, linux, x11, wayland, gnome, window-management]
updated: 2026-09-16
---

# Electron 창 전체 워크스페이스 고정과 포커스 탈취

Electron의 `setVisibleOnAllWorkspaces(true)`로 창을 모든 워크스페이스에 띄우는 기능은 리눅스에서 X11/XWayland의 `_NET_WM_STATE_STICKY`를 이용하는 것이라 네이티브 Wayland 세션에서는 아무 효과가 없다. 또한 창을 전체 워크스페이스에 고정하면 워크스페이스를 전환할 때 키보드 포커스가 그 창으로 넘어가는 부작용이 있다. `clawd-on-desk`(데스크톱 펫 앱) PR #980 작업 중 확인됨 — [[프로젝트/clawd-on-desk/프로젝트-현황]] 참고.

## X11/XWayland vs 네이티브 Wayland
- `setVisibleOnAllWorkspaces(true)`는 리눅스에서 `_NET_WM_STATE_STICKY` EWMH 힌트를 세팅하는 방식으로 동작하며, 이는 X11/XWayland 세션에서만 유효하다.
- Electron 앱을 네이티브 Wayland로 띄우면(예: `--ozone-platform=wayland`) 이 힌트를 읽는 윈도우 매니저가 없어 호출이 조용히 아무 일도 하지 않는다. X11/XWayland가 기본 실행 경로라면 평소엔 드러나지 않는 함정.
- Electron 타입 정의상 `setVisibleOnAllWorkspaces`와 관련 옵션들은 darwin/linux 전용이고(Windows는 API 자체가 없음), `visibleOnFullScreen` 등 옵션 일부는 darwin 전용이라 리눅스에 넘겨도 의미가 없다.

## override-redirect 창은 원래도 전체 워크스페이스에 있음
- 입력만 받는 히트 테스트용 창처럼 윈도우 매니저가 관리하지 않는(override-redirect) 창은 애초에 특정 워크스페이스에 속하지 않아 항상 모든 워크스페이스에서 보인다.
- 반면 WM이 관리하는 일반 창(그림을 그리는 렌더 창)은 워크스페이스 전환 시 사라지므로, 이 렌더 창에만 `setVisibleOnAllWorkspaces`를 걸어도 실질적인 문제(펫이 사라짐)는 해결된다. 두 창을 동일하게 다루려고 히트 창에도 호출을 거는 것은 선택 사항.

## macOS와의 차이 — 1회 설정 vs 매 show마다 재적용
- macOS는 창을 보여줄 때(hide→show 등)마다 sticky 상태를 다시 잃을 수 있어, `reapplyMacVisibility()`류 함수를 매번 호출해야 한다.
- 리눅스 X11에서는 생성 시 한 번 `_NET_WM_STATE_STICKY`를 걸면 hide/show를 거쳐도 유지되는 것으로 관찰됨(Chromium 구현 기준). 다만 워크스페이스 전환·미니 모드 복귀까지 포함한 완전한 검증은 아니라 재확인 여지가 있음.

## 부작용: 워크스페이스 전환 시 포커스 탈취
- 창이 모든 워크스페이스에 고정돼 있으면, 사용자가 워크스페이스를 전환할 때 GNOME(Mutter)이 키보드 포커스를 그 창으로 넘기는 현상이 관찰됨.
- A/B로 확인: 이 기능이 없는 브랜치에서는 워크스페이스 1→2→1로 이동해도 포커스가 원래 작업 중이던 앱(예: VS Code)으로 돌아왔지만, `setVisibleOnAllWorkspaces`를 적용한 브랜치에서는 포커스가 그 창에 남았다.
- 증상: 워크스페이스 전환 직후 곧바로 타이핑해도 입력이 안 먹힘. 원인이 이 포커스 탈취라는 걸 모르면 다른 데서 헤매기 쉽다.
- 해결책은 아직 없음 — 창을 포커스 불가(`focusable:false`)로 만들어도 막히는지는 미검증.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
