---
tags: [playwright, e2e, 테스트, overflow]
updated: 2026-09-12
---

Playwright의 `expect(locator).toBeVisible()`은 요소 자신의 `display`/`visibility`/`opacity`/크기만 확인하고, **조상 요소의 `overflow: hidden`이 그 요소를 화면 밖으로 잘라내고 있는지는 검사하지 않는다.** 그래서 컨테이너 클리핑 때문에 실제로는 안 보이는 요소에 대해서도 `toBeVisible()`이 통과해버릴 수 있다.

## 문제 상황

표 컨테이너의 `overflow: hidden`이 행의 케밥 메뉴를 잘라 `수정`/`삭제` 버튼이 안 보이는 버그를 고치면서([[테이블-오버플로우-hidden-드롭다운-메뉴-잘림]]), "다시 `overflow: hidden`이 걸리면 실패하는" 회귀 테스트가 필요했다. 하지만 메뉴 요소 자체에 대고 `toBeVisible()`을 검사하면, `overflow: hidden`이 있든 없든 요소는 DOM상 정상 크기로 존재하므로 **항상 통과**한다 — 정작 잡아야 할 버그를 못 잡는 테스트가 된다.

## 해결

요소가 실제로 클릭 가능한 지점에서 눈에 보이는지를 **hit-test**로 확인한다. 예: 메뉴 항목의 중심 좌표에서 `document.elementFromPoint(x, y)`를 호출해 그 결과가 실제로 대상 요소(또는 그 자손)인지 비교한다. 클리핑된 상태라면 그 좌표엔 다른 요소(또는 배경)가 잡히므로 실패로 드러난다.

## 일반화

조상의 `overflow`/`clip-path`/`position` 등으로 인한 시각적 가림을 검증해야 하는 회귀 테스트는 `toBeVisible()` 계열 단언만으로 부족하다 — 실제 렌더 결과(hit-test, 스크린샷 비교, bounding box 교차 검사)까지 확인해야 한다.

## 출처
- [[테이블-오버플로우-hidden-드롭다운-메뉴-잘림]] / Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE), 2026-09-12
