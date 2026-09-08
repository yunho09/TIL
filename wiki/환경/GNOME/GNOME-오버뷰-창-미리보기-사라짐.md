---
tags:
  - gnome
  - mutter
  - overview
  - linux
  - extension
updated: 2026-09-08
---

# GNOME 오버뷰: 3손가락 스와이프 후 가끔 창 미리보기 영역이 비는 문제 (원인 미확정)

GNOME Shell 오버뷰(3손가락 위로 스와이프)를 열면 가끔 워크스페이스 썸네일·검색·독은 정상인데 **현재 워크스페이스의 창 미리보기 영역만 통째로 비는** 증상. 재현 빈도가 낮고("가끔") 원인은 아직 확정되지 않았다. 이 페이지는 지금까지 확인된 사실과 반증된 가설, 그리고 설치해 둔 자동 진단 도구의 사용법을 남겨 다음 조사가 처음부터 다시 시작하지 않도록 하기 위한 것이다.

## 반증된 가설 — `_gestureEnd` 예외 (재조사 불필요)

처음에는 로그에 반복적으로 찍히는 아래 예외가 원인이라고 판단했다:

```
JS ERROR: Error: Invalid overview shown transition from HIDDEN to HIDING
  _changeShownState@overview.js:247
  _gestureEnd@overview.js:387
```

`Overview._gestureEnd()`가 `endProgress === 0`일 때 상태 검증 없이 `_changeShownState(HIDING)`을 불러 `HIDDEN→HIDING` 불법 전이로 예외가 나고, 그 뒤에 있는 `controls.gestureEnd()` 호출이 스킵되어 `gestureInProgress`가 영구히 `true`로 고착되면 → 오버뷰가 항상 `transitioning: true`로 판정되어 HIDDEN 상태 기하로 allocate되며 창 미리보기 영역이 빈다는 가설이었다.

**진단 로그로 반증됨.** 익스텐션에 상태값 로깅을 넣고 실제 증상이 재현된 시점을 확인한 결과, 오버뷰가 열릴 때마다 `showing`(진행 중이라 `gestureInProgress=true`가 정상) → `shown`에서 `gestureInProgress=false transitioning=false`로 매번 깨끗하게 종료되고 있었다. `guarded illegal` 가드 발동 기록도 없고, 그 부팅 동안 `Invalid overview` 예외 자체가 0건이었다. 즉 **이 예외는 실재하는 별개 버그이지만 이 증상의 원인이 아니다.**

## 확인된 사실

- 배치 자체는 정상 — 이웃 워크스페이스의 창(예: 스크린샷 속 IntelliJ/VS Code)은 오버뷰 WINDOW_PICKER 배율(화면 높이의 약 70%)로 제대로 축소·배치되어 있었다. **현재 워크스페이스의 미리보기만** 안 그려짐.
- 상단 워크스페이스 썸네일에는 내용이 정상적으로 보임 — 렌더링 파이프라인 전체가 죽은 게 아니라 가운데 window picker 쪽만 문제.
- 증상 발생 시점(스크린샷 기준) 전후로 `journalctl`의 gnome-shell 로그에 관련 예외가 없었음.
- GPU/커널 쪽은 깨끗함 — i915/xe 드라이버 오류나 GPU hang 없음, Mesa 26.0.8 정상.

## 현재 상태 — 자동 진단 익스텐션 설치됨

증상이 워낙 드물게 재현되어("가끔") 수동으로 기다렸다가 덤프를 뜨는 방식은 버리고, **익스텐션이 스스로 감지해서 스냅샷을 남기도록** 바꿨다.

- 위치: `~/.local/share/gnome-shell/extensions/overview-gesture-fix@yunho.local`
- 동작: 오버뷰가 열릴 때마다(`shown` 후 500ms) "활성 워크스페이스에 창이 있는데 실제로 보이는 미리보기가 0개인가"를 자동 검사. 걸리면 (1) 액터 트리 전체를 `~/overview-dumps/`와 journal에 기록, (2) 강제 relayout을 걸어 회복되는지 확인, (3) 회복 여부를 다시 기록. 같은 검사는 20초에 한 번으로 제한(로그 폭주 방지).
- 강제 relayout으로 회복되면 → "미리보기는 만들어졌는데 배치가 다시 안 돌았다"는 뜻이고 그 자체가 자동 복구로도 작동. 회복 안 되면 → 미리보기가 아예 안 만들어졌거나 다른 이유로 숨겨진 것. 두 경우는 원인이 완전히 다르다.
- 확인 명령:
  ```bash
  gnome-extensions info overview-gesture-fix@yunho.local   # ACTIVE 확인
  journalctl -b _COMM=gnome-shell | grep overview-gesture-fix
  overview-dump   # 수동 트리거: 3초 뒤부터 12초간 초당 1회 상태 덤프, ~/overview-dump-HHMMSS.json
  ```
- 다음 조사는 여기서 잡힌 실제 덤프(`~/overview-dumps/` 또는 journal의 `overview-gesture-fix` 태그)부터 봐야 한다. 아직 재현·수집된 덤프 없음.

## 별개로 확인된 GNOME 일반 지식 — 확장 프로그램 핫로드 불가

이 조사 과정에서 익스텐션 코드를 여러 번 고치며 반복 확인된 사실이라 따로 적어둔다: **GNOME Shell은 실행 중에 확장 프로그램 코드 변경을 감지하지 못한다.**

- `extensionSystem.js`에 확장 디렉터리를 감시하는 파일 모니터가 없어서, `gsettings`의 `enabled-extensions`에 새로 등록하거나 기존 파일을 고쳐도 셸이 스스로 다시 읽지 않는다.
- `ReloadExtension` D-Bus 메서드는 deprecated 처리되어 호출해도 거부된다:
  ```
  GDBus.Error:...NotSupported: ReloadExtension is deprecated and does not work
  ```
- 유일한 방법은 **로그아웃 → 로그인**(Wayland 세션은 `Alt+F2` r 재시작이 안 통함). `gnome-extensions info <uuid>`로 ACTIVE 여부를, gnome-shell 프로세스 시작 시각(`ps -o lstart -p $(pgrep -x gnome-shell)`)으로 실제 재로그인이 됐는지 확인할 수 있다 — 로그아웃을 깜빡하면 프로세스 시작 시각이 그대로라 바로 드러난다.

## 별개 버그 (미해결, 이번 증상과 무관)

`workspaceAnimation.js:135`에서 `TypeError: can't access property "clone", record is undefined`가 나며, 같은 계열로 `meta_window_set_stack_position_no_sync: assertion 'window->stack_position >= 0' failed`가 하루에 백 건 넘게 찍힌다. 워크스페이스 **전환 애니메이션 중** 창 하나가 빠지는 증상(오버뷰가 열린 채 창 전체가 안 보이는 이번 증상과는 다름)이라 분리해 둔다. 나중에 "워크스페이스 전환 중 창이 순간적으로 사라진다" 류의 제보가 오면 여기서부터 보면 된다.

## 환경

- GNOME Shell 50.1, Mutter 0ubuntu2.2, 단일 모니터(eDP-1 2880×1800, 배율 1.5)
- gnome-shell/mutter가 apt `hi`(hold) 상태 — 2026-07-13에 `--allow-downgrades`로 의도적으로 되돌린 흔적이 있어 이번 조사에서는 건드리지 않았음. 밀린 업데이트의 changelog에 오버뷰/제스처 관련 수정은 없어 업데이트해도 이 문제 자체는 해결되지 않을 것으로 보임(2026-09-08 기준 미검증).

## 출처

- Claude Code 세션 자동 캡처 (/home/yunho)
