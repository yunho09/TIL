---
tags: [TanStack Query, 캐시, staleTime, gcTime, 면접]
updated: 2026-11-26
---

# TanStack Query 캐시 내부 동작 — QueryCache·Observer

TanStack Query는 `queryKey`를 해시 문자열로 바꿔 **QueryCache(Map)**에 Query 객체를 저장하고, `useQuery`마다 **QueryObserver**가 그 Query를 구독한다. 캐시에 데이터가 있으면 바로 보여 주고, 오래됐으면 백그라운드에서 다시 받는다. 설정값과 무효화 실전 사용은 [[TanStack-Query-기본설정-쿼리키-계층과-삭제-시-removeQueries-먼저]].

## 구조

```
QueryClient
 └ QueryCache: Map<queryHash, Query>
     └ Query: { data, dataUpdatedAt, status, error, observers[], promise }
          ↑ 구독
     QueryObserver  ← useQuery 하나당 하나
```

## 동작 순서

- **키 → 해시**: `['tasks', { page: 1 }]` 같은 키를 **객체 키 순서를 정렬한 JSON 문자열**(`queryHash`)로 만든다. 객체를 새로 만들어도 내용이 같으면 같은 캐시
- **useQuery 실행**
  1. 해시로 Query를 찾고 없으면 생성
  2. 데이터가 있으면 바로 돌려줌 (캐시 히트)
  3. `현재 시각 − dataUpdatedAt > staleTime`이면 stale
  4. stale이고 마운트·창 포커스·재연결 같은 계기가 있으면 백그라운드 재조회
- **중복 요청 제거**: Query가 진행 중인 `promise`를 들고 있어, 같은 키의 두 번째 요청은 그것을 기다린다
- **무효화**: `invalidateQueries`는 stale 표시 후 사용 중인 쿼리만 바로 재조회, `removeQueries`는 캐시에서 삭제
- **structural sharing**: 새 응답과 이전 데이터를 깊게 비교해 안 바뀐 부분은 **이전 객체 참조를 재사용** → 불필요한 렌더 감소

## staleTime vs gcTime

| | 언제부터 세나 | 지나면 |
|---|---|---|
| `staleTime` | 데이터를 받은 순간 | stale → 계기가 생기면 다시 받음 (데이터는 계속 보여 줌) |
| `gcTime` | 그 키를 쓰는 컴포넌트가 **0개**가 된 순간 | 캐시에서 삭제 (`setTimeout`) |

- gcTime 안에 다시 쓰는 컴포넌트가 생기면 타이머 취소

## ToyVillage 적용

- 기본값: `staleTime` 60초, `retry` 1, 창 포커스 재조회 끔
- 팀 구조: `staleTime` 30분·`gcTime` 1시간. 업무 목록에서 `prefetchQuery`로 미리 넣고 업무 등록이 같은 키로 꺼내 씀 ([[포트폴리오-숫자-코드-대조와-면접-주의점-2026-11]])
- 삭제한 상세는 `removeQueries`, 목록은 prefix로 `invalidateQueries`

## 면접에서 말할 문장

> TanStack Query는 queryKey를 정렬된 JSON 해시로 바꿔 QueryCache라는 Map에 Query 객체로 저장합니다. useQuery는 캐시에 데이터가 있으면 바로 보여 주고, staleTime이 지났으면 백그라운드에서 다시 받습니다. 같은 키의 동시 요청은 promise 하나를 공유하고, 쓰는 컴포넌트가 없어지면 gcTime 뒤에 캐시에서 지웁니다.

## 꼬리 질문

- **staleTime이 지나면 로딩 화면이 되나** — 아니다. 이전 데이터를 보여 주며 뒤에서 다시 받는다. `isPending`은 캐시에 데이터가 아예 없을 때
- **Zustand와 왜 나눴나** — 서버 데이터는 캐시·재조회·무효화가 필요해 TanStack Query, 사이드바 열림처럼 서버와 무관한 화면 상태만 Zustand ([[Zustand-내부-동작-클로저-구독자-useSyncExternalStore]])

## 출처

- Claude Code 세션 자동 캡처 (/work/code/project/ToyVillage-Admin-FE) — 2026-11-26. 원리를 줄인 설명이며 라이브러리 소스·공식 문서 대조는 하지 않았다(내부 동작은 버전마다 다를 수 있음)
