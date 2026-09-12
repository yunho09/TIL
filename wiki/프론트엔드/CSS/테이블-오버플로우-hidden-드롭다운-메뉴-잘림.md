---
tags: [css, overflow, dropdown, 테이블]
updated: 2026-09-12
---

표/카드처럼 둥근 모서리를 위해 컨테이너에 `overflow: hidden`을 걸어두면, 그 안의 행에서 절대 위치로 튀어나오는 드롭다운·케밥(⋮) 메뉴까지 같이 잘린다. 행이 표 아래쪽에 있을수록 심해지고, 행이 하나뿐이면 메뉴 전체가 컨테이너 밖으로 나가 완전히 안 보이게 된다.

## 원인

`Table`(또는 카드) 컨테이너의 `overflow: hidden`은 원래 모서리를 둥글게 보이게 하려고 넣은 것인데, 그 부수효과로 자식의 `position: absolute` 메뉴도 컨테이너 경계에서 잘린다. 메뉴의 `top` 좌표가 컨테이너의 `bottom`보다 크면(예: 마지막 행) 메뉴가 시작하기도 전에 클리핑 영역을 벗어나 아예 렌더되지 않는 것처럼 보인다.

## 해결

컨테이너의 `overflow: hidden`을 제거하고, 둥근 모서리가 실제로 필요한 부분(예: 배경이 있는 헤더)에 직접 `border-radius`를 준다. 배경이 없는 행 영역은 어차피 표 자체의 둥근 모서리 범위 안에 있어서 시각적으로 문제 없다.

```css
/* before */
.Table { overflow: hidden; border-radius: 20px; }

/* after */
.Table { /* overflow 제거 */ }
.TableHeader { border-radius: 20px 20px 0 0; }
```

## 검증 시 함정

이 수정을 지키는 회귀 테스트를 Playwright로 짤 때 `toBeVisible()`만 쓰면 안 잡힌다 — 자세한 이유와 대안은 [[Playwright-toBeVisible-조상-클리핑-미탐지]] 참고.

관련: [[Zaemit-공모전/Zaemit-사이트-제작-트러블슈팅]] (반대로 `overflow:hidden`만으론 절대위치 요소가 안 잘리고 부모 `position:relative`가 같이 필요했던 사례)

## 출처
- [[ToyVillage-Admin-FE/프로젝트-현황]] / Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE), 2026-09-12
