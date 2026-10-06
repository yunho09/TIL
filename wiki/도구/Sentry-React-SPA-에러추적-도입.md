---
tags: [sentry, react, vite, react-router, observability, error-tracking]
updated: 2026-10-07
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
- ❌ 401/403처럼 **정상적인 인증 재발급 흐름**에서 나는 응답, 400/404/409처럼 **예상 가능한** 응답, CDN/이미지 등 API가 아닌 별도 흐름의 403 (초기 기준 — 400은 #212, 404·405·409는 #225에서 예외로 추가됨, 아래 절 참고)
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

## DSN 형식 함정 — 공개키 누락 시 조용히 꺼짐
- 올바른 DSN: `https://<공개키 32자리>@o<조직번호>.ingest.us.sentry.io/<프로젝트번호>`. 공개키(public key)는 어느 프로젝트로 보낼지 식별하는 값이지 비밀번호가 아니라 번들에 들어가도 된다.
- 실제 사례: Cloudflare Build variables의 `VITE_SENTRY_DSN`에 `https://` 바로 뒤 `공개키@`가 빠진 값을 넣어 배포 → 콘솔에 `Invalid Sentry Dsn`, 초기화 안 됨, Network 탭에 `envelope` 요청이 아예 없음. **번들에 DSN 문자열이 보여도 형식이 틀리면 꺼진 것**이다.
- 값은 직접 조합하지 말고 Sentry → Settings → Projects → 프로젝트 → **SDK Setup → Client Keys (DSN)**의 DSN 한 줄을 통째로 복사한다(Secret Key·Public Key 칸은 쓰지 않는다). `SENTRY_AUTH_TOKEN`(소스맵 업로드용)과는 별개 값.
- `VITE_*`는 빌드 시점 인라인이라 값만 저장하면 반영 안 되고 **Retry build**가 필요하다([[Vite-빌드타임-환경변수-인라인]]). stag/prod가 같은 값을 복사했으면 둘 다 고쳐야 한다.

## 동작 확인 절차
1. 운영 사이트를 열고 F12 → Network 필터 `envelope` → `ingest.us.sentry.io/.../envelope/`가 **200**이면 전송 중(`tracesSampleRate: 1`이면 에러 없이도 페이지 로딩 기록이 나간다). 광고 차단 확장 프로그램이 sentry.io를 막을 수 있으니 안 보이면 **시크릿 창**에서 확인.
2. 테스트 에러: Console에 `setTimeout(() => { throw new Error('sentry 테스트 prod') })`. 콘솔에서 바로 `throw`만 치면 SDK가 못 잡으므로 `setTimeout`으로 감싼다.
3. Sentry Issues에 environment 태그(`prod`)·라우트(`/login` 등)·Replay 1개가 붙어 오는지 본다. 콘솔에서 만든 에러는 스택에 원본 `src/...` 줄이 안 나올 수 있어 소스맵 검증은 실제 에러로 한다. 확인 후 테스트 이슈는 Resolve. 사용자 정보를 안 보내면 Users 0이 정상.
4. 번들에 `.map` 요청이 소스맵이 아니라 일반 SPA HTML을 돌려주면 소스맵이 공개되지 않은 것.

## 성능 화면 위치와 보는 법
- Insights → Frontend: Web Vitals, Network Requests(Outbound API Requests, 엔드포인트별 평균·p95·호출 수·실패율), Frontend Assets(JS·CSS 로딩/크기). 개별 요청은 Explore → Traces에서 `span.op:http.client`(API 호출만), `transaction.op:pageload`(첫 로딩), `navigation`(메뉴 이동)으로 필터. 메뉴 이름·경로는 버전마다 바뀌므로 안 열리면 검색(돋보기)으로 찾는다.
- 기본 대시보드 목록은 대부분 무관하다. 브라우저 React/Vite 앱이면 **Frontend Overview · Web Vitals · Outbound API Requests · Frontend Assets** 4개만 보고, AI/MCP·Backend·Laravel·Next.js·Mobile은 무시. 항상 environment를 `prod`로 걸어 stag와 섞이지 않게 한다. DSN이 고쳐진 시점부터만 쌓이므로 실사용 1~2일 후에 본다.
- Web Vitals 읽기: Performance Score 100점 만점(90+ 좋음). P75 기준 LCP ≤2.5s·INP ≤200ms·CLS ≤0.1·TTFB 빨간 점이면 약점. 표본이 15회 안팎인 페이지의 수치(예: 로그인 화면 INP 422ms는 로그인 API 대기가 섞였을 수 있음)는 며칠 쌓은 뒤 재판단. 값이 정상 범위여도 Pages 표는 로드 횟수가 많은 화면(`/`)부터 본다.

## 에러가 Sentry에 안 남는 빈틈과 보강 (이슈 #212, 2026-10-02)

배포 후 "서버가 200을 주는데 필드 하나가 비어 불러오기가 실패"하는 일이 생겨도 Sentry에 안 뜬 사례에서 정리한 것. axios 인터셉터는 **HTTP 에러 응답만** 보므로, 200인데 응답 형식 검사(런타임 가드)가 실패해서 던지는 에러는 React Query가 잡아 화면에 "불러오기 실패"만 띄우고 기록이 사라진다.

- **보강 위치**: `QueryClient`의 `QueryCache`/`MutationCache` `onError`. 위 "axios·react-query 동시 연결 시 중복" 문제 때문에, **axios 에러는 여기서 제외**하고(인터셉터가 이미 보냄) 취소된 요청도 뺀다. 쿼리 키는 첫 칸만 태그로 남기고 검색어가 섞일 수 있는 나머지는 붙이지 않는다.
- **400을 보낼지**: 이 서버는 400을 거의 "형식 오류" 전용으로 쓰고(중복은 409, 없음은 404) 프론트가 같은 규칙을 미리 막고 있어서, 운영에서 400이 나면 프론트·서버 규칙 불일치 버그일 가능성이 높다. 그래서 위 "예상 가능한 4xx 제외" 기준의 예외로 **400은 보낸다**(API·상태별 이슈로 묶임). 서버의 `description`은 입력값이 섞일 수 있어 제외. 서버가 400을 정상 거절에도 쓰는 곳이면 잡음이 되므로 서버의 상태 코드 사용 방식을 먼저 봐야 한다. 부작용: 화면이 미리 막지 않고 서버 400 문구를 그대로 보여주는 곳(예: 예약 "사전답사일은 방문일보다 늦을 수 없습니다")은 직원이 잘못 입력할 때마다 이슈가 쌓인다 — 화면 검증이 빠졌다는 신호이기도 하다.
- **강제 로그아웃 기록**: 토큰 재발급 실패 후 세션 종료는 에러가 아니라 **경고**로 남긴다. 서버가 401/403을 구분해 주지 않으면 상태 코드 대신 **실패 단계**로 나눈다(저장된 토큰 없음 / 재발급 거절 / 재발급 후에도 403). 동시에 실패한 요청이 여럿이어도 한 번만 보내고, 다른 탭에서 로그아웃해 토큰이 이미 없는 경우는 보내지 않는다.
- **같은 DSN으로 stag/prod 공유**: Sentry 프로젝트는 하나, `environment`(`stag`/`prod`) 태그로만 구분한다. Environment 필터로 같이/따로 볼 수 있지만 **이슈는 프로젝트 단위로 묶여** 같은 에러는 한 이슈이고 Resolved도 두 환경에 같이 적용된다. 알림 규칙은 environment 조건을 걸어 `prod`만 울리게 한다. 실제로 켜져 있는지는 코드가 아니라 Cloudflare Build variables(`VITE_SENTRY_DSN`, `VITE_SENTRY_ENVIRONMENT`, 소스맵용 토큰류)로 정해진다.
- **e2e 검증 방식**: Sentry 전송을 가로채서(실제 Sentry로는 안 나감) 검증하는 전용 e2e와 전용 dev 서버를 둔다. 기본 e2e 서버는 DSN을 비우고, 재사용하는 dev 서버에 DSN이 있으면 테스트 시작 전에 멈추게 해 로컬 `.env`의 DSN으로 테스트 에러가 실서버에 올라가는 사고를 막는다. 고치기 전 코드로 되돌려 테스트가 실제로 실패하는지도 확인했다.
- **그 밖의 후보(보류)**: 배포 버전(`release`) 표시(소스맵 플러그인이 자동으로 붙이는지 Sentry 이슈 화면의 Release로 먼저 확인), 사용자 피드백 버튼(화면 변경이라 상의 필요). 토큰 안의 사용자 ID(`sub`)를 `setUser({ id })`로 붙이면 이름 없이 "어느 직원"을 구분할 수 있다.

## 4xx 수집 범위 재조정 — 실측 분포로 정하기 (#225, 2026-10-07)

"이슈가 거의 없다"를 보고 수집 범위를 점검한 사례. 이슈 18개(14일)는 전부 네트워크 에러·타임아웃·세션 종료 경고였고 **HTTP 상태 코드로 만들어진 이슈는 0개**였다. 수집이 고장난 게 아니라 **수집하기로 정한 코드(400·5xx)가 실제로 거의 안 일어났던** 것.

- **실측 방법**: `tracesSampleRate: 1`이면 모든 요청 span에 `http.response.status_code`가 붙는다. Explore → Traces에서 `span.op:http.client`를 상태 코드로 그룹핑하면 코드를 안 고쳐도 "이슈로 안 올린 코드가 실제로 몇 건인가"를 바로 센다. 14일·11K span: 403 196 / 404 32 / 401 25 / 409 22 / 500 5 / 400 1.
- **업계 관행**: Sentry 기본(`httpClientIntegration`의 `failedRequestStatusCodes`)은 5xx(`[[500, 599]]`)만, 4xx는 안 잡는다. 품질 신경 쓰는 팀은 400(프론트 검증을 통과했는데 400이면 규칙 불일치)까지. 404·409는 "정상적인 업무 결과"라 보통 이슈가 아니라 **카운터/대시보드 위젯**으로 본다.
- **이 프로젝트의 결정**: 트래픽이 하루 100건대(TPM ~0.08)라 노이즈·쿼터 걱정이 없고 "등록 안 됨/삭제 안 됨" 민원이 정확히 409·404라서 이슈로 올리기로 함. 기준은 **"사용자가 실패를 봤는가"**.
  - 5xx·400·**405** → `error` (405=잘못된 HTTP 메서드, 사용자 입력과 무관한 프론트 버그)
  - **404·409** → `warning` (예상된 실패지만 건수를 세고 싶은 것)
  - 401·403·취소·오프라인 → 제외 (401·403은 토큰 재발급 흐름이라 넣으면 HTTP 이벤트의 80%가 재발급 노이즈)
  - 구현은 `reportApiError.ts`의 `shouldReport`(boolean)를 `reportLevel`(level 반환)로 바꾼 것뿐. fingerprint에 상태 코드가 이미 있어 404/409는 자동으로 따로 묶인다. 상태 코드 기준 수집량 6건 → 61건(30일).
- **상태 코드 목록은 명세에서 뽑는다**: API 계약 문서 65개를 전수 집계하니 서버가 주는 코드는 `400·401·403·404·405·409·500` 7종뿐(422·429·410·503 없음). 미수집이던 건 405 하나. 다만 405는 Swagger 보일러플레이트일 가능성이 있고 실제 발생 0건 — 명세가 실제와 다른 전례가 있다. 초안의 "409·422"도 근거 없이 쓴 것이었다.
- **배포 후 검증**: 없는 공지 상세로 들어가 404를 내면 Sentry에 `level: warning`, 경로 `/notice/:id`(숫자 ID가 패턴으로 정규화), 배포 릴리스 태그로 도착. 배포 확인은 콘솔에서 SDK의 `release` 값을 커밋 SHA와 대조.

### 오진 교훈 — Dedupe는 범인이 아니었다
`Dedupe`는 SDK **기본 통합**(직전에 보낸 이벤트와 메시지·스택·fingerprint가 같으면 전송 전에 버림)이고, `integrations`에 **배열**을 넘기면 기본 통합에 추가되고 **함수** `(defaults) => [...]`를 넘겨야 교체·제거된다(`@sentry/core`의 `integration.js`). Usage Stats의 "client-discarded 36건"(Accepted 42건 대비)을 Dedupe 탓으로 추정했고, 콘솔에서 `new Error()`를 5번 보내면 1건만 전송되는 것까지 재현했다. 그러나 **실제 코드 경로(axios → React Query retry → `reportApiError`)로 같은 500을 두 번 내면 Dedupe를 켠 채로도 2건 다 전송**됐다 — 진짜 axios 에러는 스택이 달라 판정에 안 걸린다.
- 교훈: **합성 에러(`new Error()`)로 SDK 동작을 판단하지 말고 실제 경로로 재현**한다. 수정 전/후로 테스트가 실제로 갈리는지(수정을 되돌려 실패하는지) 확인하는 습관이 잘못된 전제를 잡아냈다. 변경·테스트·이슈 문구를 전부 철회.
- client-discarded 36건의 정체는 미확정. `ignoreErrors: ['ResizeObserver loop']` 같은 필터 집계일 가능성이 크지만 확인 못 함.
- **미해명**: 10-02 스테이징 `GET /work-log` 500 5건이 span에는 있는데 Sentry 이벤트(90일, 이벤트 단위 검색)에는 0건. 전송 경로·DSN·tags·fingerprint는 테스트 이벤트로 정상 확인. fingerprint 변경 커밋(`117af92`)은 응답 없는 에러에만 적용돼 무관. 남은 가설: 직전 동일 에러에 의한 dedupe, 페이지 이탈로 전송 전 유실, 이슈 삭제.
- 테스트 이벤트를 실제 프로젝트에 보내면 이슈가 남는다(삭제는 되돌릴 수 없어 승인 필요) — 제목에 식별 가능한 이름을 붙여 보낸다. Sentry가 `fetch`를 네이티브 참조로 캐시해서 `window.fetch` 패치로는 전송 계측이 안 잡히고, SDK 훅(`preprocessEvent`/`beforeEnvelope`)으로 재야 한다.

## 운영 실사례 — 노트북 절전 복귀가 만드는 가짜 네트워크 에러 이슈 (2026-09-30 관측, #209/PR #210)

운영 투입 직후 이슈 15개가 쌓였지만 실제로는 3건(테스트 에러 1 + 사건 2)이었다. 5xx·렌더 에러는 0건.
- **현상**: 대시보드를 켜 둔 채 노트북을 덮고 6~9시간 뒤 열면, 대시보드가 동시에 부르는 API 7개가 **같은 초에** Network Error(Linux) 또는 10초 타임아웃(Mac)으로 실패 → 이슈 7개씩 생성.
- **판별 근거**: breadcrumbs에서 실패 직전까지 수 시간 활동이 없고 그 전 API는 전부 200. 서버가 죽었다면 여러 사용자에게서 활동 중에 났을 것. 같은 시각·같은 화면·여러 API 동시 실패면 서버 장애보다 클라이언트 네트워크 일시 단절을 먼저 의심.
- **원인 3겹**: (1) react-query `refetchOnReconnect` 기본값이 켜져 있어 복귀 순간(와이파이가 덜 붙은 때) 자동 재요청이 나가 실패 (2) 응답 없는 에러를 전부 전송 (3) `fingerprint`가 `api+메서드+경로+상태`라 API마다 이슈가 갈라짐.
- **해결(코드, `reportApiError.ts` 한 파일)**: 응답이 없는 에러는 fingerprint에서 경로를 빼 종류별(`ERR_NETWORK`/`ECONNABORTED`) 이슈 하나로 묶고(경로는 `api.path` 태그로 확인), 응답이 없으면서 `navigator.onLine === false` 또는 `document.visibilityState === 'hidden'`이면 전송 안 함. **5xx는 항상 전송·API별 분리 유지**(원인이 API마다 다르므로).
- **한계**: 복귀 직후 브라우저가 온라인이라 판단하고 화면도 보이는 상태에서 난 실패는 가드로 못 거른다 → 묶기가 안전망(이슈 1개).
- **코드 밖**: Sentry 알림 규칙에서 네트워크 에러·타임아웃은 제외(수집만). breadcrumb로만 남기는 안은 진짜 서버 다운(응답 없음)도 놓쳐 기각.
- Users가 0으로 나오는 건 정상 — 로그인 응답에 사용자 ID가 없어 `role`만 태그로 붙이기 때문.

## 계정·플랜 결정 메모 (2026-09-29~10-02)

- 조직 이름은 개인 이름이 아니라 팀 이름(slug가 `<org>.sentry.io`와 소스맵 `SENTRY_ORG`가 됨). 데이터 리전은 US/EU 중 선택, **가입 후 변경 불가**(한국 리전 없음). 개인 구글 계정으로 만들면 본인이 Owner — 인수인계엔 멤버 초대가 필요한데 무료는 1명뿐이라 그때 유료 전환이 필요할 수 있음.
- 온보딩 화면의 기능 체크박스는 예시 코드만 바꿀 뿐 기능을 잠그지 않는다. 온보딩 예시 코드·"Break the world" 버튼·AI 프롬프트는 환경변수·PII 제거·4xx 제외가 빠져 있어 그대로 쓰지 않는다. 프로젝트 이름은 기본값 `javascript-react`로 생성되므로 `SENTRY_PROJECT`로 쓸 이름으로 바꿔 둔다. 토큰은 Settings → Developer Settings → **Organization Tokens**(`sntrys_...`, 생성 시 한 번만 표시).
- GitHub 연동은 조직에 Sentry 앱(저장소 읽기 권한)을 설치해야 하므로 팀 합의 후에. 에러 수집·소스맵·알림에는 불필요.
- 14일 체험은 카드 미등록 시 자동으로 무료(Developer) 전환, 과금 없음. 무료: 멤버 1명·에러 월 5,000건·Replay 월 50개·보관 30일, 한도 초과분은 그달 버려짐(사이트 영향 없음). Team은 월 $26(연 결제 기준, 무제한 멤버·에러 5만건·Slack 연동 명시), Replay/Tracing 한도는 Team도 동일. 혼자 보는 용도면 무료+이메일 알림으로 충분, 팀 공유·Slack이 필요할 때 Team. 유료가 부담인데 팀이 봐야 하면 Sentry 호환 오픈소스 GlitchTip 자체 호스팅이 대안(서버 관리 부담).
- Cloudflare 등록 시 `SENTRY_AUTH_TOKEN`은 Secret, 나머지는 Text, Workers는 Build 쪽 변수 칸. 다음 빌드부터 적용. DSN이 비면 Sentry가 꺼진 채 동작하므로 환경변수 등록 전에 머지해도 안전.
- 검증: 콘솔 `setTimeout(() => { throw new Error('test') })` → 원본 파일명·줄 번호(소스맵) 확인 → 배포 주소의 `/assets/*.js.map`이 열리지 않는지 확인. 알림은 "새 이슈 발생" 하나를 `prod`에만.

## 관련
- [[Web-Vitals와-모니터링-도구-역할-분담]] — Sentry vs GA4 역할 분담, API 계측 개념
- [[번들-회귀-방지-CI와-ESLint-복잡도-도입]] — Sentry가 못 막는 머지 전 회귀 차단
- [[Vite-빌드타임-환경변수-인라인]]
- [[Cloudflare-Workers-SPA-fallback-404]] — Build variables vs Runtime variables 구분

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #200, PR #202)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-2) — 2026-10-02 (플랜·계정 결정, 운영 이슈 분석, #209/PR #210)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-2) — DSN 공개키 누락 사례, 확인 절차, 성능 화면 안내
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-2) — 이슈 #212, PR #213
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-07 (이슈 #225, PR #226/#227: 4xx 수집 범위 실측·재조정, Dedupe 오진 철회)
