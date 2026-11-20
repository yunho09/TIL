---
tags: [코드스플리팅, React.lazy, Suspense, React Router, 배럴, Vite]
updated: 2026-11-20
---

# 라우트 코드 스플리팅: React.lazy + Suspense `key`, 그리고 배럴 import 금지

화면(라우트)마다 JS를 별도 청크로 나눠 "처음 들어갈 때만" 받게 하는 방법. React Router의 라우트 `lazy` 대신 **`React.lazy` + `Suspense`**를 쓴 이유(주소가 늦게 바뀜), `Suspense`에 `key`를 줘야 하는 이유, 묶음 파일(배럴)로 import하면 분할이 깨지는 이유를 정리한다. 라이브러리 청크·해시 캐시는 [[Vite-청크-그룹으로-라이브러리-분리와-해시-캐시]], 배포 직후 옛 청크 문제는 [[배포-후-열린-탭-옛-청크-요청이-index-html로-돌아오는-문제]] 참고. 적용 프로젝트: [[ToyVillage-Admin-FE/FSD-lite-계층-구조와-배치-기준]]의 어드민(PR #254).

## 구현 (`src/app/App.tsx` `lazyPage`)
```tsx
function lazyPage(load, name, fallback) {
  const Page = lazy(async () => ({ default: (await load())[name] }))
  return <Suspense key={name} fallback={fallback}><Page /></Suspense>
}
{ path: '/tasks', element: lazyPage(() => import('@/pages/tasks/TaskListPage'), 'TaskListPage') }
```
- `import('…')`: 번들러에게 "여기서 파일을 나눠라" 신호 → 화면마다 `XxxPage-[hash].js`가 생기고 그 화면을 처음 렌더할 때 받는다. 도입 당시 화면 43~44개, 현재 라우트 52곳.
- `lazy(...)`: React가 처음 그릴 때 import 실행. `Suspense fallback`: 받는 동안 보여 줄 것(로그인은 `LoginPageSkeleton`, 다른 화면은 잠깐 빈 본문).
- 검증: e2e(`route-code-splitting.spec.ts`)가 **로그인 화면에서 다른 화면 모듈을 요청하지 않는지** 매번 확인.

## 1. 왜 React Router `lazy`가 아니라 `React.lazy`인가
| | Router `lazy` | `React.lazy`+`Suspense` |
|---|---|---|
| 메뉴 클릭 | 청크를 다 받을 때까지 **주소가 안 바뀌고 이전 화면 유지** | 주소가 바로 바뀌고 받는 동안만 본문이 빔 |
- 측정: `/tasks` 이동 시 주소가 바뀌기까지 **267 ms → 80 ms**. 처음엔 Router `lazy`로 나눴다가 "눌렀는데 반응이 없다"는 문제로 교체(`bbe8c285` 도입 → `10782901` 수정).

## 2. `Suspense`에 `key={화면이름}`을 주는 이유
- React는 같은 자리의 같은 `Suspense`를 재사용하고, 새 화면을 받는 동안 **이미 보이던 내용을 유지**하려 한다.
- `key`가 없으면 `React.lazy`가 Router `lazy`보다 더 헷갈리는 상태가 된다: **주소는 `/tasks`로 바뀌었는데 화면은 아직 대시보드**.
- `key`를 주면 "다른 Suspense"가 되어 이전 화면을 버리고 새로 기다린다 → 주소와 화면이 같이 바뀐다.

## 3. 배럴(`index.ts`)로 import하면 안 되는 이유
- 배럴 = 폴더 안 여러 파일을 한 번에 내보내는 `index.ts`(`src/pages/tasks/index.ts`가 4개 화면 export). 폴더 경로만 쓰면 `index.ts`를 자동으로 읽으므로 `import { TaskListPage, CreateTaskPage } from '@/pages/tasks'`처럼 짧게 쓸 수 있다.
- `import('@/pages/tasks')`는 `index.ts`가 4개 파일을 전부 import하므로 **4개 화면이 한 청크로 합쳐진다** → 목록만 들어가도 등록·상세·수정 코드까지 받는다. 그래서 `import('@/pages/tasks/TaskListPage')`처럼 **화면 파일을 직접** 가리킨다.
- 로그인 스켈레톤(`LoginPageSkeleton`)은 `lazy` 없이 첫 파일에 들어가야 하는데, 배럴(`@/pages/login`)로 가져오면 `LoginPage`까지 첫 번들에 딸려 들어간다 → `App.tsx` 상단 주석과 함께 파일 직접 import.
- FSD는 다른 슬라이스를 공개 API(index)로 import하라고 하지만, 코드 분할을 위해 **일부러 예외**를 둔 것.
- 테스트 쪽 배럴 함정(재수출 전체가 실행됨)은 [[배럴-Import-함정]].

## 4. 이어지는 세부 (같은 PR 묶음)
- `$initial` 태그: Rolldown `codeSplitting.groups`에서 "엔트리에서 정적 import로 닿는 모듈만" 잡는다 → 지연 로딩 화면에서만 쓰는 pdfjs는 vendor 그룹으로 끌려오지 않는다. 그룹은 앞에서부터 모듈을 가져가므로 react → sentry → vendor 순(자세한 내용은 위 Vite 청크 그룹 페이지).
- **측정 시 주의**: 코드 스플리팅 뒤 대시보드 **재방문** LCP는 552 → 818 ms로 오히려 늘었다(면접 "단점은 없었나요?"용 트레이드오프).

## 5. 대가와 재방문이 느려진 구조 (2026-11-20 보충)
- 로그인 화면 요청 10 → 17개, JS 파일 2 → 81개, 전체 JS 합계 +1.8%(파일마다 붙는 공통 코드). 로그인 JS는 압축 전 1,307 → 728 kB(실제 전송 370 → 229 kB). `/tasks`로 갈 때 주소가 바뀌기까지 router `lazy` 267 ms → `React.lazy` 80 ms.
- **재방문이 느려진 이유**: 재방문하면 브라우저가 캐시한 파일마다 "바뀌었나?"를 서버에 묻고 안 바뀌었으면 **304**를 받는다. 파일이 2 → 81개로 늘면서 이 304 요청이 **2 → 13번**으로 늘어 본문은 다시 안 받아도 확인 시간이 쌓였다. 원인을 더 보니 **해시가 붙은 파일에도 `max-age=0`(매번 확인)** 이 걸려 있었다 → 해시 파일은 내용이 바뀌면 이름이 바뀌므로 오래 캐시하게 서버 설정을 바꾸면 질문 자체가 사라진다(다음 과제, [[Cloudflare-Workers-SPA-fallback-404]]처럼 `_headers` 설정 영역).
- **화면별 분리와 라이브러리 분리는 목적이 다르다**: 화면별은 "처음에 덜 받기"(소스 `import()`+`React.lazy` 수정, 처음 방문에 이득), 라이브러리는 "배포 뒤 다시 안 받기"(빌드 설정만 수정, 재방문에 이득, 오히려 파일이 늘어 첫 방문 전체량은 줄지 않음). 세션 녹화는 둘 다 써서 `load` 뒤 `import()`(늦게 받기) + `replay` 그룹(라이브러리만 분리).
- **배럴 보충**: `@`는 `tsconfig.app.json`의 `paths`와 `vite.config.ts`의 `alias`가 둘 다 `src`로 지정한 별칭이고, 폴더 경로만 import하면 도구가 `index.ts`를 자동으로 읽는다. 로그인 스켈레톤을 배럴(`@/pages/login`)로 가져오면 `LoginPage`까지 첫 번들에 들어간다(트리 셰이킹이 항상 빼 주지 않아 실제로 들어갔음). FSD는 보통 index(공개 API) import를 권하므로 여기서는 코드 분할을 위한 **의도적 예외**. `route-code-splitting.spec.ts`가 개발 서버가 파일 단위로 보낸다는 점을 이용해 로그인에서 `/src/pages/` 중 로그인 폴더 밖이 요청되면 실패시키고, 응답을 붙잡아 스켈레톤이 보이는지도 확인한다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-11-20 (면접 대비 설명 세션)
