---
tags: [TanStack Query, 캐시, 쿼리키, invalidateQueries, removeQueries, 토빌]
updated: 2026-11-21
---

# TanStack Query 기본 설정·쿼리 키 계층·삭제 시 removeQueries를 먼저

TanStack Query는 서버 데이터를 **쿼리 키(이름표)를 붙여 캐시**하고, 같은 키로 다시 부르면 서버에 묻지 않고 보관본을 보여 주며, "낡았다"고 표시하면 다시 받아 온다. 토빌 어드민(develop 코드 확인)의 기본 설정, 키를 큰 묶음→작은 묶음으로 짜는 이유, 삭제 후 `removeQueries`를 `invalidateQueries`보다 먼저 호출하는 이유를 정리한다. 다른 함정은 [[TanStack-Query-무효화-키-null-vs-undefined]], [[TanStack-Query-mutate-콜백-언마운트-스킵]], 구조 배경은 [[FSD-lite-계층-구조와-배치-기준]].

## 기본 설정 (`shared/config/queryClient.ts`)
- `staleTime: 60_000` — 받은 뒤 60초는 "신선"으로 보고 같은 화면에 다시 들어와도 요청 안 보냄.
- `retry: 1` — 실패하면 한 번만 재시도.
- `refetchOnWindowFocus: false` — 다른 창 갔다 와도 자동 재조회 안 함.
- 데이터 성격에 따라 개별 덮어쓰기: 먹이 급여 기록은 다른 직원이 계속 추가하니 `staleTime: 0`(들어올 때마다 새로), 팀 구조(`/team/tree`)는 느리고(9초대) 잘 안 바뀌니 `staleTime: 30분`(업무 목록에서 미리 받아 둠 — [[포트폴리오-숫자-코드-대조와-면접-주의점-2026-11]]).
- **에러 이중 보고**: `QueryCache`의 `onError`에서 **axios 에러가 아닌 에러만** Sentry로 보낸다. 응답 형식 검사 실패 같은 에러는 TanStack Query가 잡아 화면 에러 상태로만 바꿔 버려 Sentry에 안 남기 때문. axios 에러는 인터셉터가 이미 보냈으므로 제외(중복 방지). 형식 검사는 [[응답을-unknown으로-받고-타입-가드로-검사하기]].

## 쿼리 키는 "큰 묶음 → 작은 묶음" 순서 (`entities/feed/model/queryKeys.ts`)
```ts
all:    ['feeds']
lists:  ['feeds', 'list']
list:   ['feeds', 'list', { date, floor, page }]
detail: ['feeds', 'detail', floor, feedLogId]
```
- 앞쪽 prefix로 무효화하면 하위 키가 전부 걸린다: `['feeds','list']`만 낡음 처리하면 날짜·페이지가 달라도 목록 전체가 한 번에 처리되고 상세는 그대로.
- 4층·5층이 서로 다른 API라 id가 겹칠 수 있어 키에 `floor`를 넣었다.

## invalidateQueries vs removeQueries
| | 동작 | 쓰는 곳 |
|---|---|---|
| `invalidateQueries` | "낡았다" 표시 + 지금 화면에 떠 있는(active) 쿼리는 **바로 재조회** | 삭제 후 목록(그 항목이 빠져야 함), 연관 집계 |
| `removeQueries` | 캐시에서 **아예 삭제**, 재조회 안 함 | 이미 서버에서 지워진 상세 |

업무 삭제(`pages/tasks/TaskDetailPage.tsx`):
```ts
queryClient.removeQueries({ queryKey: ['tasks', id] })       // ① 지운 상세는 버림
await queryClient.invalidateQueries({ queryKey: ['tasks'] }) // ② 목록은 다시 받기
navigate('/tasks')
```
- **순서가 핵심**: `['tasks']`는 prefix라 `['tasks', id]`(상세)까지 같이 낡음 처리된다. 상세를 먼저 지우지 않으면 아직 마운트된 상세가 재조회를 시도해 **404**가 난다.
- 목록 키를 `['tasks']`처럼 넓게 주면 1·2페이지·필터가 달라도 전부 한 번에 처리된다.
- 개체(`SpeciesDetailPage`) 삭제도 같은 방식: 개체 상세는 remove, 종의 마릿수는 개체 수에서 계산되므로 종 쪽도 invalidate.

## 면접용 한 줄
- 키 계층: "도메인부터 큰 묶음 순서로 키를 붙여서 목록만 또는 상세만 골라 다시 받을 수 있게 했고, 급여 기록처럼 계속 바뀌는 건 매번 새로, 느리고 잘 안 바뀌는 팀 구조는 30분 캐시했습니다."
- 삭제: "삭제한 상세는 서버에 없어 다시 받으면 404라 removeQueries로 지우고, 목록은 invalidateQueries로 다시 받게 했습니다. 목록 무효화가 상세까지 건드리므로 상세를 먼저 지웠습니다."

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.config/Claude/scratch-workspaces/68eef3b8-da94-4db5-b563-9565e29705a3/4cebf249-3d72-4a77-a55b-e742194ef06a/scratch-2026-11-20-d28dbc) — 2026-11-21 (토빌 모의면접 중 develop 코드 직접 확인)
