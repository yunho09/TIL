---
tags: [TypeScript, 타입 가드, unknown, 응답검증, 토빌]
updated: 2026-11-21
---

# 응답을 unknown으로 받고 타입 가드로 검사하기

TypeScript 타입은 **빌드할 때만** 있고 실행 중엔 사라져서, `api.get<TeamTree>()`라고 써도 서버가 필드를 빼 보내면 그대로 들어와 화면 깊은 곳에서 `undefined`로 터진다. 토빌 어드민은 응답을 `unknown`으로 받고 손으로 짠 타입 가드로 검사한 뒤 통과해야 타입을 믿는다. 검사 강도의 적정선은 [[응답-검증-적정선-없음-vs-조회실패-구분]], 에러 보고 경로는 [[TanStack-Query-기본설정-쿼리키-계층과-삭제-시-removeQueries-먼저]] 참고.

## 패턴 (`entities/team/api/teamApi.ts`)
```ts
const { data } = await api.get<unknown>('/team/tree')   // ① 모름으로 받기: 확인 전엔 못 꺼내 씀
if (!isTeamQueryTreeResponse(data)) {                    // ② 직접 검사
  throw new Error('팀 구조 조회 응답 형식이 올바르지 않습니다.')
}
// 여기부터 data는 TeamQueryTreeResponse
```
- 비유: 송장(타입)에 "노트북"이라 적혀 있어도 상자를 열어(타입 가드) 확인하는 것.
- 실패 시 API를 받는 곳에서 바로 throw → 어디가 문제인지 즉시 안다. 이 에러는 `QueryCache.onError`로 Sentry에 가서 **200인데 필드가 빠진 응답**도 운영에서 알 수 있다(이슈 03 수집 기준 조정 때 추가).

## 타입을 바꿔 주는 건 `is`
- `true/false`만 돌려주는 함수는 검사해도 타입이 계속 `unknown`. **반환 타입의 `value is T`**(type predicate)가 "true면 이 값은 T"라고 TypeScript에 약속한다.
- `if (!guard(data)) throw`로 false 경로를 빠져나가면 그 아래에서 `data`가 좁혀진다(**narrowing**).
- 주의 두 가지: ① 실제 값이 바뀌는 게 아니라 컴파일러가 보는 타입만 바뀐다 ② TypeScript는 함수 내부 검사를 확인하지 않고 `is`를 **그냥 믿는다** → 검사를 빼먹으면 틀린 데이터도 통과(정확히 짜는 건 사람 몫).

## 검사 구현
- 먼저 `typeof value !== 'object' || value === null`로 객체인지, 그다음 필드마다 `typeof`·`Number.isInteger`·`Array.isArray`. 중첩 객체·배열은 하위 가드(`isTeamQueryTreeGroup`)를 따로 만들어 한 단계씩 내려가며 `&&`로 잇는다. 하나라도 틀리면 false.
- **zod 같은 스키마 라이브러리 없이 수작업**. 왜 안 썼는지는 기록에 없음 → 지어내지 말고 "직접 짰고, 스키마 라이브러리면 검사와 타입을 한 번에 정의할 수 있다는 건 안다"로 답한다.

## 적용 범위 (develop 코드 세어 본 결과)
- `unknown`으로 받고 검사: **85곳**(거의 모든 조회·등록·수정), 다른 타입 3곳, 타입 없이 호출 3곳(삭제 3개).
- 예외 6곳은 응답 내용을 안 쓰는 곳: 삭제 3개(휴관일·급여 기록 2개, 성공 여부만), 휴관일 등록·공지 등록(본문 안 씀, 공지는 `status`만), 엑셀 다운로드(JSON이 아닌 `Blob`).
- 규칙: "응답 데이터를 꺼내 쓰는 곳은 전부 unknown+검사, 응답을 안 쓰거나 파일인 곳만 예외."
- 말할 때: "에러를 정규화했다"고 하지 않는다 — 인터셉터는 에러를 그대로 넘기고 검사는 API 함수마다 한다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.config/Claude/scratch-workspaces/68eef3b8-da94-4db5-b563-9565e29705a3/4cebf249-3d72-4a77-a55b-e742194ef06a/scratch-2026-11-20-d28dbc) — 2026-11-21
