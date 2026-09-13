---
tags: [tanstack-query, react, ui-bug]
updated: 2026-09-14
---

목록 화면에서 탭/필터/페이지를 바꿀 때 `placeholderData: (prev) => prev`로 새 데이터가 도착할 때까지 이전 결과를 계속 보여주면, 화면이 매끄러워지는 대신 **행별 액션(승인·반려·삭제 등 버튼·메뉴)이 그 이전 데이터를 대상으로 그대로 활성 상태**로 남는다.

## 실패 시나리오

목록에 `상태` 탭이 있고 각 행에 승인/반려 케밥 메뉴가 있는 화면에서, 사용자가 `완료` 탭에서 `반려` 탭으로 전환한다. 새 데이터가 도착하기 전까지 화면엔 여전히 `완료` 탭의 행들이 보이는데, 그 케밥 메뉴는 계속 열려 있다. 사용자가 그 상태에서 `반려하기`를 누르면, 실제로는 **이미 승인된 항목에 반려 요청**이 나간다. 백엔드가 상태 전이를 막아주면 409로 끝나지만, 막아주지 않으면 조용히 잘못된 상태로 바뀐다.

## 왜 생기나

쿼리 키가 바뀌어 새 요청은 나가지만, `placeholderData`가 이전 키의 데이터를 그대로 붙여줘서 `isLoading`은 `false`, `data`도 채워진 것처럼 보인다. 화면 코드가 `isLoading`만 보고 액션 버튼 활성 여부를 판단하면 이 틈을 놓친다.

## 해결

`useQuery`가 반환하는 `isPlaceholderData`를 확인해, 그게 `true`인 동안은 행 액션(메뉴 열기, 버튼 클릭 등)을 막는다.

```ts
open={!pending && !isPlaceholderData && openMenuId === row.id}
```

또는 탭/필터가 실제로 바뀌었을 때만 이전 데이터를 유지하고, 그 외엔 `placeholderData`를 쓰지 않는 방식으로 범위를 좁힌다.

## 관련

같은 화면 계열에서 발견된 다른 TanStack Query 함정은 [[TanStack-Query-mutate-콜백-언마운트-스킵]], [[TanStack-Query-무효화-키-null-vs-undefined]] 참고.

## 출처
- [[ToyVillage-Admin-FE/프로젝트-현황]] — 2026-09-14: 업무보고 목록 API 연동 코드 리뷰에서 발견, `isPlaceholderData` 가드로 수정.
