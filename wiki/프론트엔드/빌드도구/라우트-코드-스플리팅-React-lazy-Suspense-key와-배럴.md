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

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-11-20 (면접 대비 설명 세션)
