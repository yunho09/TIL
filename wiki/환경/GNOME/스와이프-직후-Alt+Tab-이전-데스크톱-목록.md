---
tags: [GNOME, Mutter, 확장, 워크스페이스, Alt+Tab]
updated: 2026-10-27
---
# 스와이프 직후 Alt+Tab이 이전 데스크톱 창 목록을 띄우는 문제

3손가락 스와이프로 데스크톱을 넘기자마자 Alt+Tab을 누르면 도착한 데스크톱이 아니라 **떠나는 데스크톱**의 창 목록이 뜨고, 거기서 창을 고르면 이전 데스크톱으로 끌려간다. 원인은 스와이프 종료 후 슬라이드 애니메이션이 끝나야 활성 워크스페이스가 바뀌는 GNOME Shell 구조이며, 확장 `swipe-alttab-fix@yunho.local`로 해결했다(실제 화면 확인은 재로그인 후 대기 중, 미확정).

## 원인
- 3손가락 스와이프는 손을 뗀 뒤 **0.1~0.4초** 더 미끄러지다 멈춘다. 활성 워크스페이스가 실제로 바뀌는 건 이 애니메이션의 onComplete 시점이다.
- 그 전에 Alt+Tab을 누르면 `get_active_workspace()`가 아직 떠나는 데스크톱을 가리켜 목록이 그 기준으로 만들어진다. 선택하면 셸의 onComplete가 다시 원래 목표로 활성화해 끌려가는 것처럼 보인다.
- **키보드 전환(Ctrl+Alt+←→)** 은 활성 워크스페이스가 먼저 바뀐 뒤 애니메이션이 돌아서 문제가 없다.
- 같은 뿌리의 증상: [[GNOME-오버뷰-창-미리보기-사라짐]] (애니메이션 중 제스처 → 스크롤 값이 중간에 남음).

## 해결: `swipe-alttab-fix@yunho.local` 확장
- `SwitcherPopup.prototype._init`(altTab.js의 WindowSwitcher/AppSwitcher/WindowCycler/GroupCycler 공통 부모)를 패치. `_startSwitcher`가 `.bind()`로 미리 묶여 있어 호출별 패치는 불가능하므로 **추상 기반 클래스의 `_init`** 이 올바른 훅이다.
- 워크스페이스 애니메이션의 `begin`/`end` 신호로 진행 중인 스와이프를 기억하고, 팝업 생성 시 아직 미끄러지는 중이면 목표 워크스페이스를 `activate()`한 뒤 `_finishWorkspaceSwitch()`로 전환을 즉시 끝내 도착 데스크톱 기준으로 목록을 만든다. 셸과 같은 순서(activate → finish)라 키보드 슬라이드 재시작은 `_shouldAnimate()`가 막아 준다.
- GNOME 확장은 새로 설치하면 **재로그인해야** 로드된다(`gnome-extensions info`가 "존재하지 않음"으로 나옴).

## 검증 방법
- 헤드리스 격리 셸 + 가상 키보드로 스와이프 직후 Alt+Tab을 재현. 수정 끈 기준선은 증상 그대로, 켜면 도착 데스크톱만 표시.
- 함정: 헤드리스 셸은 기본 키 설정이라 Alt+Tab이 **앱 전환(AppSwitcher)** 에 묶여 있음 → 실제 설정(창 전환)과 맞춰야 재현됨.
- 취소된 스와이프·반대 방향·키보드 전환·확장 끄고 켜기 등 9가지 모두 통과.

## 리뷰에서 나온 교훈
- **취소 후 되돌아오는 중**에 settle하면 공유 workspaces adjustment가 1.6 같은 중간값으로 남고, 나중에 개요를 열 때 `round(1.6)=2` 워크스페이스로 끌려갈 수 있다. 현재는 overview-gesture-fix 확장이 보정해서 가려질 뿐이라, settle 시 그룹 progress를 목표 정수로 스냅하도록 수정했다.
- `workspaces-only-on-primary`일 때 보조 모니터 제스처는 셸이 무시하지만 `begin`은 발생 → 같은 가드를 `begin` 핸들러에도 둬야 함(LOW).
- 다른 확장이 같은 프로토타입을 나중에 감싸면 `disable()` 복원이 체인을 깨뜨릴 수 있어 null 가드가 필요(LOW).

## 남은 한계
- **Ctrl+Alt+Tab**은 `CtrlAltTabManager.popup`이 `new CtrlAltTabPopup()` 전에 목록을 만들기 때문에 이번 수정이 안 먹음. 커버하려면 `Main.ctrlAltTabManager.popup` 인스턴스 메서드도 래핑해야 함.
- 같은 뿌리의 미해결: 스와이프 직후 바로 위로 스와이프해 개요를 열면 떠나는 워크스페이스 기준으로 열림.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
