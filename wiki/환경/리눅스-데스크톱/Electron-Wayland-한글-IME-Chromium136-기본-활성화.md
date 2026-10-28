---
tags: [electron, wayland, ime, chromium, 한글]
updated: 2026-10-28
---

# Electron Wayland 한글 IME: Chromium 136부터 플래그 없이 기본 동작

`--enable-wayland-ime`와 `--wayland-text-input-version=3` 플래그가 하던 일을 Chromium 136(Electron 36)부터는 기본으로 한다. 그래서 Electron 42/44 앱에 이 플래그를 기본값으로 넣는 PR은 효과가 없고, 오히려 사용자가 `disable-wayland-ime`로 끄는 길을 막는다.

## 근거
- Chromium 152 소스: `ui_base_features.cc`의 `kWaylandTextInputV3`가 `FEATURE_ENABLED_BY_DEFAULT`. `IsImeEnabled()`가 이 기능이 켜져 있으면 true, `CreateTextInput()`도 플래그 없이 v3 선택. 변경 커밋은 e48954bdc391, 136.0.7103.48부터(130~134는 DISABLED).
- 직접 실행: 최소 Electron 앱(`<input>` 포커스)을 `WAYLAND_DEBUG=1`로 GNOME 50 Wayland에서 띄우면 44.5.1 플래그 없음/있음, 42.0.1 플래그 없음 모두 `zwp_text_input_manager_v3.get_text_input` 호출 후 포커스 때 `enable()`, `set_content_type`을 보냄 — 플래그 유무와 무관.
- Wayland에서 GTK IME는 쓰지 않음(`gtk_ui_platform_wayland.cc`가 nullptr).

## 기본값으로 넣으면 손해인 이유
- `enable-wayland-ime`를 항상 붙이면 `IsImeEnabled()`에서 enable이 먼저 걸려 사용자가 `disable-wayland-ime`로 끌 수 없다.
- fcitx5 위키 기준 KWin은 v3에 문제가 있어 v1을 권하고 Weston은 v3가 없어, v3 강제는 안전한 기본값이 아니다(위키가 최신이 아닐 수 있음). 필요한 사용자는 Chromium switches 설정에 `enable-wayland-ime` + `wayland-text-input-version=1`.
- 130~135 시절 한글 IME 문제 보고(mutter #3090, Fedora 토론)는 136 이전이라 근거로 쓰지 말 것.

## 진단 순서
1. 앱을 완전히 끄고(켜져 있으면 새 실행이 기존 창으로 넘어감) 런처 플래그 없이 실행해 한글 입력.
2. 안 되면 `WAYLAND_DEBUG=1 <앱> 2>&1 | grep text_input`로 text_input_v3 바인딩 확인.
- 이 건에서 과거 한글 입력 실패의 진짜 원인은 확인하지 못했다(플래그 때문은 아닐 가능성이 큼).

## 교훈
- "플래그가 없어서 안 된다"는 가설은 Chromium 버전을 먼저 확인하고 소스·실측으로 반증한다. 오픈소스 기여 후보 하나가 이렇게 불필요로 판명 → [[오픈소스-기여-후보-선별-기준과-AI정책-함정]].

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
