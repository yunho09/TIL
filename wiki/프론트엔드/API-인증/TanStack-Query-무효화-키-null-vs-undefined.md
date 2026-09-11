---
tags: [frontend, tanstack-query, react-query, caching]
updated: 2026-09-11
---

# TanStack Query 캐시 무효화 — `null`은 와일드카드가 아니다

쿼리 키에 파라미터가 없을 때 `?? null`로 채우면, 그 키로 무효화(`invalidateQueries`)해도 파라미터가 있는 다른 키들이 지워지지 않는다. **TanStack Query의 prefix 매칭은 키 배열에서 `undefined`만 와일드카드로 취급하고, `null`은 그 자체로 구체적인 값으로 본다.**

## 증상

```ts
const noticesKeys = {
  noticeList: (params?: { page?: number }) => ['notice-list', params?.page ?? null] as const,
}
```

`noticeList()`(인자 없이 호출) → `['notice-list', null]`.
`noticeList({ page: 2 })` → `['notice-list', 2]`.

공지 작성 후 `invalidateQueries({ queryKey: noticesKeys.noticeList() })`를 호출하면 키가 `['notice-list', null]`이 되는데, 이건 `['notice-list', 2]`의 prefix가 아니라서 **캐시가 하나도 안 지워진다.** 화면은 에러 없이 조용히 낡은 목록을 계속 보여준다.

## 해결

파라미터가 없을 때 `null`로 채우지 말고 **그냥 `undefined`가 되도록 둔다**(또는 파라미터 자체를 키에서 생략).

```ts
noticeList: (params?: { page?: number }) => ['notice-list', params] as const,
```

`noticeList()` → `['notice-list', undefined]`. 무효화 로직이 키에서 `undefined` 세그먼트를 걸러내도록 돼 있으면(직접 만든 `invalidate` 헬퍼든, `queryKey: ['notice-list']`처럼 짧게 넘기든) 결과적으로 `['notice-list']`가 되어 `page` 값에 상관없이 전부 prefix 매칭된다.

**핵심은 "값 없음"을 표현할 때 `null`과 `undefined`를 구분해서 써야 한다는 것** — 둘 다 "값이 없다"는 의미로 섞어 쓰기 쉽지만, 캐시 키 설계에서는 완전히 다르게 취급된다. `??`/`||` 자체의 함정은 별개, [[Nullish-Coalescing-빈문자열-함정]] 참고.

## 언제 의심할지

- 생성/수정/삭제 API 요청 자체는 성공하는데, 목록 화면이 갱신 안 될 때.
- 무효화 대상 쿼리 키를 만드는 함수에 `?? null`, `|| null`처럼 명시적으로 `null` 폴백이 들어가 있을 때.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — [[프로젝트/JOBIS-FE-V2/프로젝트-현황]]
