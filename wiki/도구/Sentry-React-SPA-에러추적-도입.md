---
tags: [sentry, react, vite, react-router, observability, error-tracking]
updated: 2026-09-29
---

# Sentry — React+Vite SPA에 에러·성능 추적 붙이기

`@sentry/react` + `@sentry/vite-plugin`로 React(react-router v7)+Vite SPA에 에러 추적과 성능(Web Vitals) 추적을 붙이는 방법과, 실전에서 걸리기 쉬운 지점들. 기본 `Sentry.init()` 하나로는 부족하고 라우터·axios·react-query 세 곳을 따로 연결해야 한다.

## 기본 초기화

앱 렌더링보다 먼저(라우터 생성 전) `Sentry.init()`을 한 번 호출한다. `dsn`이 비어 있으면 SDK가 아예 동작하지 않게 만들어 두면 로컬·e2e에서 자동으로 꺼진다.

주요 옵션:
- `dsn`: `VITE_*` 환경변수로 빌드에 인라인됨 ([[Vite-빌드타임-환경변수-인라인]])
- `environment`: `local`/`stag`/`prod` 구분
- `release`: 배포 버전(커밋 해시). 소스맵과 매칭할 때 쓴다
- `tracesSampleRate`: 성능 추적 샘플링 비율. 사용자가 적은 내부 도구(직원용 어드민 등)는 낮게 잡으면(0.1) 데이터가 거의 안 쌓이므로 `1`(전부)로 두는 게 낫다
- Replay(`replayIntegration`)는 선택. 개인정보가 보일 수 있는 화면이면 텍스트·입력값 마스킹을 기본으로 하고, 에러가 난 세션만 녹화하도록 제한

## 기본 설정만으로는 안 잡히는 3곳

1. **라우터**: `createBrowserRouter`(react-router v7)를 `Sentry.wrapCreateBrowserRouterV7`로 감싸야 화면 경로(`/tasks/:id`)가 함께 기록된다. 라우트의 `errorElement`에서 잡힌 렌더 에러는 React가 삼켜서 별도 전송 없이는 Sentry에 안 남는다 — 에러 화면 컴포넌트 안에서 `Sentry.captureException`을 직접 호출해야 한다. 라우터 밖(Provider 등)에서 나는 에러는 React 19 root의 `onUncaughtError` 훅으로 잡는다.
2. **axios**: API 에러가 가장 많이 쌓이는 지점. 공용 axios 인스턴스의 응답 인터셉터에서 보낼 에러를 직접 고른다.
3. **react-query**: `QueryClient`의 `QueryCache`/`MutationCache`에 `onError`를 걸면 쿼리 에러를 한곳에서 보낼 수 있다.

axios 인터셉터와 react-query `onError`를 **둘 다** 연결하면 같은 API 에러가 두 번 전송된다 — 한쪽만 고른다.

## API 에러 필터링

전부 보내면 노이즈가 커지고 무료 플랜 한도도 빨리 찬다. 판단 기준:
- ✅ 5xx, 네트워크 에러, 타임아웃 — 실제로 뭔가 깨진 신호
- ❌ 401/403처럼 **정상적인 인증 재발급 흐름**에서 나는 응답, 400/404/409처럼 **예상 가능한** 응답, CDN/이미지 등 API가 아닌 별도 흐름의 403
- 이슈가 API 경로+상태 코드 단위로 나뉘도록 묶어야 한다. 안 그러면 axios 에러가 전부 한 이슈로 뭉친다.
- 스테이징이 없는 경로에도 500을 돌려주는 서버라면, 미배포 API 호출까지 5xx로 잡혀 스테이징 알림이 시끄러워진다 — 스테이징 알림은 끄거나 따로 둔다.

## 개인정보 보호

- **Authorization 헤더만 지우는 걸로는 부족하다.** 토큰 재발급 요청 본문에 `refresh_token`이 그대로 들어가고, axios 에러 객체는 `config.data`(요청 바디)까지 들고 있다. `beforeSend`에서 요청 헤더와 본문(data)을 **통째로** 제거하는 편이 안전하다.
- **Sentry v11부터 `sendDefaultPii` 옵션이 `dataCollection`으로 이름이 바뀌었다.** 기본값이 헤더·쿠키·요청 본문까지 전부 수집하는 쪽이라 명시적으로 꺼야 한다.
- `dataCollection`의 `urlQueryParams: false` 같은 옵션은 **에러 이벤트뿐 아니라 성능(트레이싱) 데이터의 URL에도 동일하게 적용된다.** 목록 화면 검색어(`?keyword=...`)처럼 URL에 개인정보가 실리는 경우를 이걸로 같이 막을 수 있다.
- 사용자 식별은 `Sentry.setUser({ id })`처럼 id만 넣는다. 이름·이메일은 넣지 않는다. **단, 로그인 응답에 사용자 ID 자체가 없으면(이름·역할만 오는 API) 이 방법을 못 쓴다** — `role` 같은 태그로 대체할 수는 있지만 "몇 명에게 영향이 있었는지" 집계는 포기해야 한다.

## 소스맵 업로드

- `vite.config`에 `sentryVitePlugin`을 추가하고 `build.sourcemap: true`로 둔다. 없으면 스택이 압축된 코드 위치(`index-abc123.js:1:48213`)로만 보인다.
- 업로드 후 `filesToDeleteAfterUpload` 옵션으로 `.map` 파일을 dist에서 지운다 — 배포 서버에 원본 소스 위치 정보가 공개되지 않게 하기 위해서다.
- `SENTRY_AUTH_TOKEN`, `SENTRY_ORG`, `SENTRY_PROJECT`는 **빌드 환경변수**로만 넣는다. `VITE_` 접두사를 붙이면 번들에 그대로 박혀 공개되므로 절대 붙이면 안 된다(Cloudflare Pages/Workers처럼 빌드-런타임 변수가 분리된 플랫폼에서는 "Build variables" 칸에 넣어야 한다 — [[Cloudflare-Workers-SPA-fallback-404]] 참고).
- Auth Token이 잘못돼도 소스맵 업로드만 실패하고 빌드 자체는 성공한다(이 경우 `.map` 파일이 dist에 안 남는지도 확인해야 함).

## 성능(Web Vitals) 추적

`reactRouterV7BrowserTracingIntegration`을 켜면 아래가 자동으로 기록된다:
- 페이지 로드·화면 이동 시간(라우트 패턴별)
- 그 로드·이동 구간 중에 나간 API 요청의 URL·상태 코드·소요 시간
- Web Vitals: LCP, CLS, INP, FCP, TTFB

한계:
- SPA 특성상 **LCP·CLS는 첫 진입(하드 리로드) 때만** 측정된다. 메뉴를 눌러 화면을 옮길 때는 안 잰다. INP는 계속 측정된다.
- API 응답 시간은 "페이지 로드나 화면 이동" 구간 안에 나간 요청만 잡힌다. 화면이 안정된 뒤 저장·삭제 버튼으로 보내는 요청은 이 구간 밖이라 빠질 수 있다 — 필요하면 axios에서 요청마다 직접 시간을 재는 보강이 필요하다.
- v11부터 성능 데이터는 예전 방식(transaction)이 아니라 span 단위로 전송된다. Sentry 화면에서는 동일하게 Insights 메뉴에서 보인다.
- 백엔드가 다른 도메인이고 CORS로 추적 헤더를 안 열어 두면, 프론트에서 본 응답 시간까지만 보이고 서버 내부 어디가 느렸는지는 안 보인다. 이어 보려면 백엔드에도 Sentry를 붙이고 CORS에 추적 헤더를 허용해야 한다.

## 관련
- [[Vite-빌드타임-환경변수-인라인]]
- [[Cloudflare-Workers-SPA-fallback-404]] — Build variables vs Runtime variables 구분

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #200, PR #202)
