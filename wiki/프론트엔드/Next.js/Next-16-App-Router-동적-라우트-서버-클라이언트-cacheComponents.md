---
tags: [nextjs, app-router, cacheComponents, suspense, emotion, 서버컴포넌트]
updated: 2026-11-11
---

# Next.js 16 App Router: 동적 라우트·서버/클라이언트 컴포넌트·cacheComponents

Next.js를 처음 쓰는 사람이 공지 상세 페이지(`/notice/[id]`)를 만들며 정리한 기본 개념과, `cacheComponents: true` 환경에서 실제 빌드로 확인한 함정 모음. 핵심은 "파일·함수 이름이 곧 약속"이라는 것과, `params`처럼 요청 때 정해지는 값은 `Suspense` 안에서 읽어야 한다는 것이다. 적용 사례: [[xquare-openapi-frontend/프로젝트-현황]].

## 1. 폴더 = 주소
- `src/app` 아래 폴더 구조가 URL이 된다. **파일 이름이 `page.tsx`일 때만** 페이지가 생기고, 같은 폴더의 `notice-detail.tsx`·`page-style.ts`는 주소가 안 생기는 부품 파일이다(그래서 `[id]` 폴더 안에 같이 둘 수 있다).
- `layout.tsx`는 여러 페이지를 감싸는 틀. `children` 자리에 현재 주소의 `page.tsx`가 들어간다.
- 폴더명 `[id]`는 동적 경로. 폴더 이름이 곧 키 이름(`[slug]`면 `params.slug`)이고 **값은 항상 문자열**이다. 데이터 id를 숫자로 저장하면 `notice.id === id` 비교가 항상 실패한다.
- 정적 경로(`/forgot-password`)는 루트의 `[category]` 같은 동적 경로보다 우선해서 충돌하지 않는다.

## 2. 서버 컴포넌트 vs 클라이언트 컴포넌트
- 기본은 서버 컴포넌트, 파일 맨 위 `"use client"`가 있으면 클라이언트 컴포넌트.
- 서버: `async/await`로 데이터 찾기, `metadata` export. 클라이언트: `useState`·`onClick`·**Emotion `styled`**. `async` 컴포넌트는 서버만 가능.
- **Emotion은 내부에서 React Context를 쓰므로 클라이언트에서만 동작**한다. 이 프로젝트에선 "서버 `page.tsx`가 데이터 찾기·404 판단 → props로 클라이언트 파일에 넘겨 Emotion으로 그리기"로 나눈다. `metadata`도 서버에서만 쓸 수 있어 폼(클라이언트)과 `page.tsx`를 나눈다.
- 임시 파일로 직접 빌드해 얻은 실제 에러 메시지(어기면 이렇게 나온다):
  - 서버에서 Emotion `styled` 사용 → `TypeError: g.createContext is not a function` (**메시지가 불친절**, "`use client` 빠졌나?"부터 의심)
  - 서버에서 `useState` → "You're importing a module that depends on `useState` into a React Server Component module…"(고치는 법까지 안내)
  - 서버→클라이언트로 함수 prop → "Event handlers cannot be passed to Client Component props."
- 서버→클라이언트 props는 **직렬화 가능한 값**(문자열·숫자·일반 객체·배열)만. 함수는 클라이언트 안에서 만든다.
- 클라이언트 파일이 서버 컴포넌트를 쓰려면 **import가 아니라 `children`으로 받아야** 한다(import하면 그 컴포넌트도 클라이언트가 됨). 예: 클라이언트 `NoticeDetailFrame`이 서버 `NoticeContent`를 `children`으로 받으면 껍데기는 `Suspense` 바깥, 본문은 안쪽에 둘 수 있다.

## 3. `params`는 Promise (Next 15+)
- 서버: `const { id } = await params;` (`params: Promise<{ id: string }>`), 클라이언트: React `use(params)`. 옛 방식 `params.id`는 값이 안 나온다.
- 이유: 주소 값을 몰라도 그릴 수 있는 부분(헤더 등)을 먼저 그리려고, 진짜 값이 필요한 순간까지 기다림을 미루기 위해서.

## 4. `generateStaticParams`
- 동적 경로의 빌드 시점 사전 생성 목록. 함수 이름이 정확히 이것이어야 Next가 찾는다. 예: `NOTICES.map(n => ({ id: n.id }))` → 빌드 로그에 `/notice/5`, `/notice/4` 등이 정적(○)으로 찍힌다.
- **`cacheComponents` + 사이드바 `usePathname()` 구조에서는 사실상 필수**: `generateStaticParams` 없이 동적 경로를 만들어 빌드하면 `Route "/x/[id]": Next.js encountered URL data usePathname() in a Client Component outside of <Suspense>` (사이드바에서 발생)로 실패한다. 그래서 `[무언가]` 폴더를 만들면 항상 같이 쓴다.

## 5. `cacheComponents`와 `Suspense`
- 켜면 페이지를 **정적 껍데기**(누가 언제 와도 같은 헤더·구분선)와 **나중에 채울 부분**(`params`·쿠키처럼 요청이 와야 아는 값)으로 본다. 경계가 `<Suspense>`.
- 규칙: `params` 등 요청 값을 쓰는 부분은 `Suspense` 안에서 읽는다. 안 그러면 dev에서 "navigation is blocking" 경고. 패턴: `<Frame><Suspense><AsyncContent params={params}/></Suspense></Frame>`.
- `Suspense`에 `fallback`이 없으면 기다리는 동안 그 자리는 빈칸(헤더가 바깥이라 대개 문제없음).
- `partialPrefetching: true`는 `Link`가 화면에 보이면 껍데기를 미리 받아 클릭 시 거의 즉시 이동하게 한다.
- **방문한 라우트는 사라지지 않고 숨겨진 채 남는다**(Activity). 폼 입력값·열린 모달·삭제 대상 state가 그대로 남으므로, 페이지가 숨겨질 때 `useLayoutEffect` cleanup으로 초기화한다(`useLayoutEffect(() => () => setX(null), [])`).
  - 네이티브 `<dialog>`를 `showModal()`로 연 채 브라우저 뒤로→앞으로 가면, 숨김 동안 Chrome이 `:modal` 상태를 버려 **backdrop 없이 떠 있고 Esc로도 안 닫히며 뒤 페이지가 클릭된다**(재현 확인). 모달 effect cleanup에서 `dialog.close()`를 부르는 건 금물(`close` 이벤트가 나중에 발생해 dev StrictMode에서 열자마자 `onClose`가 불림) — 부모가 state를 reset하는 쪽이 맞다.
  - reset 훅 deps를 `[initialValue]`로 두면 값이 바뀔 때 cleanup이 **이전 값으로** 되돌린다. `[]`+ref 또는 `key`로 리마운트.

## 6. `notFound()`와 HTTP 상태 코드
- `next/navigation`의 `notFound()`는 그 자리에서 렌더링을 멈추고 404 화면을 보인다.
- **`Suspense` 안에서 부르면 첫 요청 상태 코드는 200**(껍데기를 이미 스트리밍한 뒤라 못 되돌림). 화면은 404. 프로덕션은 반복 요청부터 404(캐시), dev는 매번 200. 이 프로젝트의 `[category]`·admin edit 라우트도 같다. 이 동작은 Next 쪽에 피드백 초안(`agentFeedback`)으로 정리했으나 제출은 사용자 몫.
- **`export const dynamicParams = false`는 `cacheComponents`와 함께 못 쓴다**(`Route segment config "dynamicParams" is not compatible with nextConfig.cacheComponents`로 빌드 실패). Next 16 문서는 그 export를 지우고 `notFound()`를 쓰라고 한다. CodeRabbit이 이 옵션을 제안했을 때 실제 빌드로 반박했고, 봇이 스스로 철회하며 repo "learning"으로 저장했다. → 리뷰 봇 제안은 **우리 설정에서 실제 빌드되는지·공식 문서가 뭐라는지** 확인하고 적용.

## 7. `<Link>`와 기타
- `<a href>`는 전체 재요청, `next/link`의 `Link`는 바뀐 부분만 교체(사이드바 유지, 클라이언트 내비게이션). 이동 후에도 페이지에 심어 둔 표시가 남으면 새로고침 없이 이동한 것. `styled(Link)`로 Emotion 스타일 가능. 주소에 변수는 `` href={`/notice/${id}`} ``.
- 사이드바 활성 판정이 `pathname.startsWith("/notice")`면 상세에서도 자동 활성.
- 목록 `key`는 제목이 아니라 id(겹치지 않음).
- 이전/다음 글은 목록이 최신순이면 이전=`index+1`, 다음=`index-1`; 범위 밖은 `undefined`라 `{prev && …}`로 숨겨진다.
- 본문 줄바꿈은 `white-space: pre-wrap`, 한글 단어 보존은 `word-break: keep-all`.
- 피그마의 gap 박스·divider 레이어는 CSS의 `margin`/`padding`/`border`로 옮긴다(의미 없는 태그를 늘리지 않음).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/xquare-openapi-frontend, 2026-10-22, 2026-11-01)
