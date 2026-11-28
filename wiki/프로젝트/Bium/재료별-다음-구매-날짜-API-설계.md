---
tags: [Bium, 백엔드, API설계, 구매주기, Spring]
updated: 2026-11-23
---

# 재료별 다음 구매 날짜 API (bium-be #66 / PR #68)

냉장고 재료의 구매 이력으로 "다음에 살 날"을 계산해 주는 BE API. FE 달력(bium-fe #83)과 짝이 되는 기능이며, 제안서의 "구매 주기로 다시 살 날 추천"([[Bium/프로젝트-현황]])을 실제로 구현한 것이다. 계산 규칙과 응답에서 빠지는 조건이 헷갈리기 쉬워 기록한다.

## API
- `GET /api/fridges/{fridgeId}/purchase-schedule`
- 응답: `ApiResponse.data.items[]`, 각 항목은 `name`, `emoji`, `lastPurchasedAt`, `purchaseCount`, `cycleDays`, `nextPurchaseDate`.
- 냉장고 멤버가 아니면 **404 FRIDGE_NOT_FOUND**(존재 노출 방지, [[냉장고-공유-API-설계와-리뷰-반영]]과 같은 관례), 토큰이 없으면 401.
- 코드: `domain/shopping`의 `PurchaseScheduleController`·`PurchaseScheduleService`·`PurchaseScheduleResponse`, 조회는 `FridgeItemRepository.findAllPurchasedWithCatalogByFridgeId`(소프트 삭제 행 포함, 도감 left join fetch).

## 계산 규칙
- 같은 냉장고의 `fridge_items` **전체(지운 재료 포함)** 중 `purchasedAt`이 있는 행을 **도감 이름(없으면 재료 이름 trim)** 으로 묶는다. 소프트 삭제 행을 넣는 이유는 다 먹고 지운 재료가 곧 구매 이력이기 때문.
- 서로 다른 구매일이 **2개 이상**: 주기 = round((마지막 − 처음) / (n − 1))일, 최소 1일.
- 구매일이 **1개**: 가장 최근 행의 `purchaseCycleDays`를 주기로 쓴다. 그 값도 없으면 **그 재료는 응답에서 뺀다**.
- 다음 구매일 = 마지막 구매일 + 주기. 정렬은 다음 구매일, 이름 순.

## 주의: "한 번만 산 재료는 빠진다"는 틀린 표현
- 한 번만 샀어도 `purchaseCycleDays`가 있으면 응답에 들어간다. 빠지는 건 **한 번만 샀고 주기 값도 없는 재료**뿐이다. PR 본문 검증 문구가 이 점에서 명세와 어긋나 있어 고쳤다. 실측에서 돼지고기·토마토·표고버섯·사과가 주기 값이 비어 빠졌고, 우유는 주기 7일에 다음 구매일 11-27로 나왔다.

## 상충하는 정보
- 위 실측은 우유에 주기 7일이 있었다고 기록한다. 2026-11-28 점검은 **도감 76개 중 `purchaseCycleDays`가 있는 재료가 0개**였고, 기본 재료 넣기·사용자 재료 추가 어느 쪽도 값을 넣지 않으며 고치는 API도 없다고 확인했다(점검용 로컬 서버 기준). 우유의 값이 어디서 왔는지는 확인되지 않았다. 후속으로 재료별 기본값 추가를 맡겼으나 완료는 미확인([[Bium/FE-BE-전체-기능-점검과-병렬-수정-2026-11-28]]).

## 협업 관례(이 작업에서 확인된 것)
- 브랜치명은 `<이슈번호>-feat-...` 형태(`66-feat-purchase-schedule`), PR은 develop으로, 본문에 `Closes #66`, AI 표기 줄은 넣지 않는다.
- FE 세션이 BE 기능을 별도 worktree에서 만들어 PR까지 열 수 있다. 이때 BE 본 폴더의 다른 작업·원격 브랜치와 같은 파일(`FridgeItemRepository` 등)을 건드리는지 먼저 대조했다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/bium-be)
