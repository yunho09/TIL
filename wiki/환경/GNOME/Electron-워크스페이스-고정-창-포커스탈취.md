---
tags: [electron, linux, x11, wayland, gnome, window-management]
updated: 2026-10-22
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
- ~~해결책은 아직 없음~~ → 아래 "원인과 해결"에서 원인 특정·해결 (2026-10 후속).

## 원인과 해결: `WM_HINTS input=False` (2026-10 후속)
- **원인 (mutter 50.1 소스 기준):** Electron/Chromium 렌더 창에는 `WM_HINTS`가 없어 mutter가 "포커스 가능한 창"으로 취급한다(`is_focusable() = input || wm_take_focus`, Chromium은 WM_TAKE_FOCUS를 쓰지 않음). 여기에 sticky까지 붙으면 한 번 포커스를 받은 순간 **모든 워크스페이스의 MRU 맨 앞**에 올라가고, 이후 워크스페이스를 전환할 때마다 mutter가 MRU 첫 번째 포커스 가능 창(=이 창)에 포커스를 준다.
- **해결:** 출력 전용 창이면 X11 `WM_HINTS`의 InputHint 플래그를 세우고 `input=False`로 써서 포커스 후보에서 뺀다(ICCCM `InputHint = 1L<<0`, 9개 CARD32, format 32, 속성·타입 모두 atom 35=WM_HINTS, Replace 모드). mutter는 PropertyNotify로 WM_HINTS를 다시 읽고 기본 포커스를 MRU에서 실시간 계산하므로 창이 뜬 뒤에 써도 전환 시 포커스 문제는 해결된다.
- **Electron에는 이 API가 없다.** `setFocusable(false)`는 리눅스에서 CanActivate만 바꿀 뿐 WM_HINTS를 건드리지 않는다. 그래서 koffi로 libxcb를 직접 호출하는 작은 모듈로 구현: `xcb_change_property_checked` + `xcb_request_check`(checked 요청이라 Xlib의 에러 시 프로세스 종료 핸들러를 피함), 창 ID는 네이티브 핸들에서 읽어 29비트로 제한.
- **게이트:** `--ozone-platform`이 정확히 `x11`일 때만 활성. 네이티브 Wayland 핸들은 작은 위젯 ID라 XID 범위 검사를 통과해 버리므로 이 게이트가 유일한 방어선이다. Windows/macOS는 koffi 로드 전에 단락.
- **A/B 검증(워크스페이스 1→3→1 후 포커스 위치, 자동):** main = 에디터 3/3, sticky만 적용한 수정 전 = **펫 3/3(문제 재현)**, 수정 후 = 에디터 3/3. 수동으로도 전환 후 클릭 없이 타이핑·드래그·숨기기/보이기·미니 모드·재시작 정상.
- **남은 한계(수정 안 함, main도 동일):** 창이 처음 뜰 때 한 번은 포커스를 잡는다. Electron 생성자가 `show:true` 기본값으로 창을 매핑하고 활성화를 요청한 뒤에야 힌트가 써지고, mutter는 매핑 시점(`window_state_on_map`)에 input=TRUE로 본다. 클릭하거나 워크스페이스를 한 번 옮기면 에디터로 넘어간다. 근본 해결은 Linux에서 `show:false`로 만들어 매핑 전에 sticky/WM_HINTS가 있게 하는 것인데 실제 GNOME/KDE 검증이 필요해 미적용.
- **Replace 모드 주의:** WM_HINTS의 기존 필드(urgency, group leader)를 지운다. Chromium은 생성 시 WM_HINTS를 쓰지 않고 `SetFlashFrameHint`만 읽고-쓰므로 지금은 안전.
- **부수 효과:** 컨텍스트 메뉴 같은 transient 창이 닫힐 때 포커스가 펫 대신 MRU 앱으로 돌아간다(mutter `find_focusable_ancestor`가 건너뜀).
- **로컬 우회책(`wmctrl` 감시자)은 이 수정이 정식 릴리스에 들어가기 전까지 계속 필요.**

## 외부에서 `input=False`를 붙일 때: 창 범위 함정 (2026-10-22)
- 같은 앱의 질문 말풍선 같은 보조 창도 펫 본체와 같은 종류의 창이라, 감시 스크립트가 종류만 보고 `input=False`를 붙이면 말풍선 입력칸이 먹통이 된다(입력을 받아야 하는 창까지 포커스 후보에서 빠짐). 창 제목으로 출력 전용 본체("Clawd")에만 적용해야 한다.
- 수정 후 말풍선은 워크스페이스 고정만 되고 입력 불가 표시는 안 붙으며, 전환 시 포커스가 해당 워크스페이스 앱으로 정상 복귀함을 로그로 확인.
- PR #980은 2026-10-22 머지됨(v1.3.0엔 미포함, 다음 릴리스부터) — [[프로젝트/clawd-on-desk/프로젝트-현황]].

## koffi로 네이티브 호출 테스트할 때의 함정
- koffi는 **JS 인자 개수 초과를 조용히 무시**하고 `void *`에 숫자도 받는다(`labs(-5, 99)`가 5를 반환). 선언에서 파라미터를 빠뜨려도 이름만 resolve하는 테스트는 통과하고, 실연결에서는 엉뚱한 포인터로 libxcb가 복사해 크래시할 수 있다.
- 대처: 함수의 `fn.info.arguments`/`fn.info.result`를 C 프로토타입과 대조(koffi는 `str`을 `char *`로 표시), 에러 연결(`:65535` 같은 잘못된 디스플레이)에서도 I/O 없이 호출 가능한 경로로 값 전달 마샬링을 실행.
- 상수 테스트는 내보낸 상수와 비교하면 값이 0이어도 통과하므로 **와이어 리터럴**(`[1,0,0,0,0,0,0,0,0]`)로 고정해야 변이가 잡힌다.
- 스킵 조건이 "koffi 로드 실패"와 "libxcb 없음"을 섞으면 npm allowScripts 정책으로 koffi 설치 스크립트가 막힌 환경에서 유일한 실바인딩 검증이 조용히 건너뛰어진다 — 리눅스에서는 `require("koffi")` 실패를 실패로 처리.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
