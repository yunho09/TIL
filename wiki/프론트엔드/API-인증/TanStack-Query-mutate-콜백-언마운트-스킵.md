---
tags: [frontend, tanstack-query, react-query, mutation]
updated: 2026-09-13
---

# TanStack Query — mutate() 콜백은 언마운트 후 스킵되고, invalidate 완료를 기다리지 않는다

`useMutation()` 훅 정의가 아니라 `mutation.mutate(vars, { onSuccess, onSettled })`처럼 **호출 시점에 넘긴 콜백**(per-call callback)에 캐시 무효화(`invalidateQueries`)를 넣으면, TanStack Query v5는 이 콜백을 두 가지 방식으로 다르게 다룬다.

## 문제

- 컴포넌트가 언마운트되면(`hasListeners()`가 false) per-call 콜백 자체가 **실행되지 않는다**. `useMutation` 훅 정의에 넣은 `onSuccess`/`onSettled`는 언마운트 후에도 실행된다.
- per-call 콜백은 **await되지 않는다** — `mutation.mutate()`가 성공하면 `isPending`은 곧바로 false로 바뀌고, 콜백 안에서 `await invalidateQueries(...)`가 아직 돌고 있어도 기다리지 않는다.

## 실패 시나리오

- **처리 중 화면 이탈**: 승인 버튼을 누르고 요청이 끝나기 전에 뒤로가기/다른 페이지 이동 → 요청은 성공하지만 언마운트돼 콜백의 `invalidateQueries`가 실행되지 않아 목록이 `staleTime`만큼(예: 60초) 갱신 안 된 채로 남는다. 사용자가 같은 항목을 다시 승인할 수 있어 중복 요청 위험도 생긴다.
- **처리 중인데 컨트롤이 풀려 보임**: `isPending`이 `invalidateQueries` 완료를 기다리지 않고 먼저 false가 되므로, 버튼이 다시 활성화되거나 모달의 확인/취소가 풀리는 등 "아직 처리 중"인데 조작 가능해 보이는 창이 생긴다.

## 해결

부수효과(캐시 무효화, `pending` 플래그 해제처럼 **항상 일어나야 하는 것**)는 `useMutation`의 훅 옵션으로 옮긴다. TanStack Query는 이 콜백들을 기다리고, 언마운트 후에도 실행한다.

```ts
useMutation({
  mutationFn,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['task-reports'] }),
  onSettled: () => { reviewingRef.current = false },
});
```

화면에만 필요한 것(토스트, `navigate`, 모달 닫기, 포커스 이동)은 그대로 `mutate()` 호출 시점의 per-call 콜백에 둔다 — 이건 언마운트 후 실행 안 되는 게 오히려 맞는 동작이다.

## 관련 함정 — 넓은 키 무효화가 곧 사라질 좁은 쿼리를 다시 가져옴

상세 페이지가 `['list-key']` prefix를 무효화하면, 아직 마운트돼 있는 `['list-key', id]`(상세 쿼리)도 refetch 대상이 된다. 그 직후 상세 페이지가 `removeQueries`로 그 쿼리를 치워버리면 방금 받은 응답은 그냥 버려진다 — 불필요한 GET 한 번과, 헤더 배지가 잠깐 새 값으로 바뀌었다 사라지는 깜빡임의 원인이 된다. `predicate`로 상세 키를 제외하거나, 제거를 먼저 하고 무효화하는 순서로 피한다.

관련: [[TanStack-Query-무효화-키-null-vs-undefined]] (같은 라이브러리의 다른 캐시 무효화 함정)

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]]
