---
tags: [performance, bundle, code-splitting, vite, lcp, baseline]
updated: 2026-10-28
---

# 어드민 SPA 번들 성능 — baseline 실측과 최적화 우선순위

Vite+React 어드민(ToyVillage-Admin-FE)의 개선 전 수치를 재현 가능한 조건으로 고정해 두고, 효과 큰 순서로 최적화를 나열한 정리. 핵심은 "before 숫자를 먼저 찍어두지 않으면 나중에 얼마나 줄였는지 말할 근거가 사라진다"는 것과, 소스 구조를 안 건드리는 싼 항목이 스플리팅보다 먼저라는 판단이다.

## 측정 방법(재현 조건)
- `yarn build` → `npx vite preview --port 4199 --strictPort` → Playwright + CDP: `Network.emulateNetworkConditions`(Slow 4G 1.6Mbps/150ms RTT) + `Emulation.setCPUThrottlingRate`(4x), navigation/paint/LCP 수집, **5회 중앙값**. Chrome 확장이 연결 안 될 때는 설치된 Playwright로 대체.
- 번들 구성은 **소스맵을 디코드해 패키지별 기여도**를 산출. 수치에는 항상 조건을 같이 적는다(조건이 다르면 before/after 비교가 성립 안 함).
- 랩으로 못 재는 것은 INP·에러율·API p95 → [[Web-Vitals와-모니터링-도구-역할-분담]].

## baseline
| 항목 | 2026-09-22 (`7487430`) | 2026-09-28 (`29812dc`) |
|---|---|---|
| 메인 JS | 957.66 kB (gzip 260.46) 청크 1개 | 984.84 kB (gzip 266.65) |
| 로그인 LCP(=FCP) | 2,088 ms | 2,124 ms |
| JS 전송 | 252.5 kB / 1,464 ms | 258.6 kB / 1,486 ms |
| 로고 PNG | 139.4 kB / 839 ms | 동일 |
| vendor | 351 kB (45.9%) | 44.7% |

- vendor 구성: react-dom 174 kB(22.8%), react-router 93 kB, axios 44 kB, @tanstack/query-core 32 kB. 앱 코드 한 줄만 바꿔도 이 351kB가 통째로 재다운로드된다(매일 쓰는 내부 도구라 재방문 캐시가 중요).
- 규모: 모듈 557 / 소스 377 / 라우트 43. `lazy(` 0건, `prefetchQuery` 0건, `React.memo` 0건. 로그인 화면 하나에 개체관리·업무일지·대시보드 코드를 전부 받는다. Vite가 `chunks are larger than 500 kB` 경고를 낸다.
- 서버 상태: `useQuery` 56 / `useMutation` 33, 전역 `staleTime` 60초, `placeholderData` 6개 화면. 업무일지 쪽은 `predicate`로 목록만 무효화하고, 급여/피드는 다른 직원이 계속 추가하므로 `staleTime: 0`으로 둔 판단이 주석으로 남아 있다.
- 이미 잘 된 것: `pdfjs-dist`를 `await import()`로 동적 로드해 PDF 청크 431kB + 워커 1.27MB가 초기 로드에서 빠져 있다.

## 2026-10-14 재측정 — 로컬 빌드는 Sentry가 빠져 실제보다 작게 나온다
- 같은 조건(프로덕션 빌드, Slow 4G, CPU 4x, 캐시 끔, 5회 중앙값, develop `2613909`)으로 다시 쟀다. 9/22 이후 기능이 늘어 메인 JS 957→**1,031 kB**(+7.7%, 로컬 빌드 기준)로 커졌고, PDF 청크(431 kB)는 이미 분리돼 있어 9/22 값과 구성이 다르므로 **스플리팅 직전에 before를 다시 재야** 한다.
- **함정**: 로컬 `.env`에는 `VITE_SENTRY_DSN`이 없어 `initSentry`가 빌드에서 통째로 제거(dead-code elimination)된다. DSN을 넣은 **배포 빌드**로 다시 재면 메인 JS **1,306 kB**(gzip 372), 로그인 초기 JS 전송 **369.8 kB**, 로그인 LCP **2,528 ms**(로컬 빌드 280.4 kB·1,968 ms)로 훨씬 크다. 앞으로 before·after 모두 **가짜 DSN을 넣은 배포 빌드**로 재고, 측정 중 sentry.io·GA 요청은 막는다. 환경변수가 번들 내용을 바꾸는 항목([[Vite-빌드타임-환경변수-인라인]])은 측정 전에 배포와 같게 맞춘다.
- **번들 구성(배포 빌드 1,306 kB)**: react-dom 179, **Sentry 전체 298(23%)**(replay 125.3·core 94.5·browser 27.4·browser-utils 25.5·react 15.3·web-vitals 8.7), react-router 96, axios 45(fetch 전환은 재발급 흐름까지 바뀌어 비추천), 기타 라이브러리 ~75, **인라인 SVG 69개 ~63 kB**+아이콘 파일 26 kB(Figma 추출 SVG라 불필요한 속성이 많아 압축 여지), 나머지 앱 코드 ~340. 코드 스플리팅은 **전체 크기는 거의 그대로**고 첫 화면 전송량만 줄인다 — 전체를 줄이려면 replay 지연 로딩(−125 kB)+SVG 정리(−30~40 kB 추정)가 먼저(약 −13%).
- **로그인 LCP 요소는 로고가 아니라 왼쪽 패널의 인라인 SVG**: 위 우선순위 2번(로고 포맷 전환)을 해도 LCP는 거의 안 준다. 로고 작업의 after는 "전송 143 kB→?"로 쓰는 게 정확하다(위 baseline의 "로고가 LCP 요소"라는 추정을 이 측정이 정정).
- **"느린 화면=JS"라는 가정을 뒤집은 사례**: 업무 만들기 화면 이동 p95 9.24s를 코드 스플리팅 대상으로 봤으나 운영에서 9번 연 기록 모두 화면 이동 시간이 `/team/tree` 응답 시간과 거의 같았다(예: 9,236 ms 중 9,172 ms; `/team/tree` p95 8.53s). 담당자 선택 트리가 이 API를 기다리는 **API 대기**이므로 스플리팅으로는 안 줄고, 팀 구조는 자주 안 바뀌어 프론트에서 캐시·미리 불러오기로 대기를 가리는 게 가능하다(서버 응답 자체는 백엔드 영역이라 "프론트가 대기를 가린 몫"으로만 서술). 10/6 이후 운영 사용 0회라 Sentry로 after를 못 재고 지연을 건 mock 실험실 측정을 쓴다.
- 운영 Web Vitals(Sentry p75): LCP 502 ms·INP 48 ms·CLS 0·TTFB 542 ms는 개선 숫자가 아니라 "운영에서 재고 있다"는 증거(관리자 1명+개발자 테스트가 섞인 값이라 기간·환경을 붙인다). 번들 예산 CI 없이 3주에 +7.7%가 늘었다는 사실이 size-limit의 근거 → [[번들-회귀-방지-CI와-ESLint-복잡도-도입]].

## 2026-10-20 — 이미지 최적화(#262/PR #265) 결과: 파일 −90%, LCP는 그대로
- 한 일: 로그인 로고(390×328, 143 kB)를 **화면 표시 크기(136×114)의 2배**인 272×228로 줄여 WebP(품질 90)로 변환, 사이드바 프로필(64×64 표시)은 128×128 WebP로. import 경로만 `.webp`로 바꾸면 CSS가 크기를 고정하고 있어 화면은 그대로. 변환은 Pillow로 했고 새 패키지는 없음. 쓰이지 않던 `public/assets/login-logo.png` 사본(위 우선순위 1번의 중복)은 삭제. 하네스 문서(명세·컴포넌트 맵)의 로고 파일명도 함께 갱신.
- 용량이 줄어든 몫은 두 가지가 겹친 것: **표시 크기에 맞춘 픽셀 수 축소** + **PNG→WebP**. 로고에 투명 배경이 있어 JPG는 불가, WebP는 투명 지원(PNG식 무손실/JPG식 손실 압축 둘 다 가능, 이번엔 손실 q90). 브라우저 지원은 Chrome·Edge·Firefox·Safari 14+로 별도 대비 불필요.
- 머지 전후(`9d31b34` vs `10fced2`) 실측(배포 빌드, Slow 4G+CPU 4x, 5회 중앙값, 전후 번갈아 2회 반복):

| 항목 | 전 | 후 | 변화 |
|---|---|---|---|
| 로그인 로고 파일 | 143 kB | 13.8 kB | −90% |
| 로고를 다 받은 시각 | 2,749 ms | 2,135 ms | −22% |
| 로그인 화면 전체 전송량 | 431 kB | 302 kB | −30% |
| 사이드바 프로필 파일 | 64.0 kB | 8.3 kB | −87% |
| 로그인 LCP | 1,704 ms | 1,706 ms | 변화 없음 |

- **LCP가 그대로인 이유**: 첫 큰 페인트가 로고가 아니라 왼쪽 패널 그림이고 로고는 그 뒤에 받기 시작 → 위 10-14 절의 예측대로. 그래서 성과 문구는 "LCP 개선"이 아니라 "**전송량·로고 표시 시각 개선**"으로 쓴다. 프로필은 로그인 후 화면에만 나와 브라우저 측정 없이 파일 크기만 비교. 지운 사본(142.7 kB)은 아무도 받지 않던 파일이라 사용자 전송량 영향 없음(배포 산출물에서만 빠짐).
- **남은 후보 — 파비콘**: `public/favicon.png`(192×192, 64 kB)가 변환 전 프로필 이미지와 동일 파일(`favicon.ico` 10.5 kB, `apple-touch-icon.png` 57.9 kB). 파비콘이 "안 되어 있다"는 뜻이 아니라 "아이콘치고 무겁다"는 뜻. 낮은 우선순위로 받고 캐시되며 LCP를 안 늦추므로 영향은 작고, 크롬이 `.ico`/`.png` 중 하나만 고르는지는 미확인.

## 2026-10-20 — 최종 재측정: 개선 전(`6dd3985`) vs 전부 반영(`14721cb`)
스플리팅(#252)·라이브러리 청크(#257/#261)·replay 지연·팀 구조 캐시(#253/#260)·이미지(#262)·replay 라이브러리 분리(#267/#268)가 다 들어간 develop을 **같은 컴퓨터에서 전후 번갈아 2번** 쟀다. 조건은 배포 빌드(가짜 DSN), Slow 4G(1.6 Mbps/150 ms)+CPU 4x, 캐시 끔, 5회 중앙값, sentry.io·GA 요청 차단.

| 항목 | 개선 전 | 지금 | 변화 |
|---|---|---|---|
| 로그인 화면 JS | 1,307 kB (gzip 368) | 728 kB (gzip 231) | −44% |
| 화면이 뜨기 전 JS 전송량 | 370 kB | 233 kB | −37% |
| 로그인 LCP | 2,526 ms (5회 2,512~2,588) | 1,708 ms (1,696~1,724) | −32% |
| 로그인 화면 전체 전송량 | 527 kB | 303 kB | −43% |
| 로고를 다 받은 시각 | 3,346 ms | 2,149 ms | −36% |
| 로그인 load 이벤트 | 3,350 ms | 1,726 ms | −48% |
| 요청 수 | 10 | 17 | 파일이 나뉜 대가 |
| 앱 코드만 바꾼 배포 뒤 다시 받는 라이브러리 | 693 kB | 0 kB | (중간: #261 후 127 kB) |

- **비교 기준은 항상 가장 나빴던 원래 before**: 중간 상태(예: 이미지 최적화 직전)와 비교하면 "이미 줄인 위에서 또 줄인" 값이 되어 효과가 과소/과대로 읽힌다. 처음엔 로고 전후 값(431→302 kB, 2,749→2,135 ms)을 그대로 쓰다가 "수정한 값에서 또 수정한 값과 비교하면 어떡하냐"는 지적으로 전부 원래 before 기준(527→303, 3,346→2,149)으로 바꿨다. 중간 값은 "이미지 변환만의 효과" 참고용으로만.
- LCP는 10/19 측정(2,536→1,706)과 오차 범위 안에서 같고, 최종 코드로 잰 값으로 카드에 썼다. 파일이 나뉘면 요청이 10→17개로 늘지만 둘 다 페이지 로드 뒤에 받는 replay 청크(40.5→41.3 kB, 2개)라 첫 화면에 영향 없음 — 그래서 효과 확인을 용량이 아니라 **LCP**로 한다.
- **앱 코드만 바꾼 배포 시뮬레이션**: 임시 worktree(`git worktree add --detach`)에 `node_modules` 심볼릭 링크+`.env` 복사 후 대시보드 제목 한 줄만 `sed`로 바꿔 다시 빌드하고 파일명 비교. react 287·sentry 172·vendor 107·replay 126·pdf 431 kB는 이름 유지, 바뀐 건 `sentryReplay` 설정 코드(0.7 kB)와 화면 코드(91개 중 74개). 원리는 [[Vite-청크-그룹으로-라이브러리-분리와-해시-캐시]].
- **업무 만들기 대기(실험실)**: 운영 사용이 적어 mock으로 `/team/tree` 응답을 운영 수치(9,172 ms)만큼 늦추고, **업무 목록에서 업무 만들기로 넘어갈 때 담당자 트리가 뜨는 시간**을 3회씩 쟀다(값이 거의 같음).

  | 목록에 머문 시간 | 개선 전 | 개선 후 |
  |---|---|---|
  | 1초 | 9.25 s | 8.18 s |
  | 3초 | 9.25 s | 6.17 s |
  | 10초 | 9.25 s | 0.37 s |

  구현은 `teamTreeQueryOptions`(staleTime 30분·gcTime 60분)를 `TaskListPage`·`TaskDetailPage`에서 `prefetchQuery`로 미리 받는 방식이라 **목록에 오래 머물수록 대기가 사라지고 바로 넘어가면 효과가 작다.** 성과에 쓸 땐 "실험실, 업무 목록에 10초 머문 뒤" 조건을 같이 적어야 정확하다(바뀌기 전에도 5분 안 재방문은 전역 staleTime 60초+gcTime으로 캐시돼 있어, 실제 차이는 5분~1시간 사이 재방문이며 이건 안 쟀다). 캐시를 길게 두면 직원 계정을 새로 발급해도 담당자 목록에 안 뜰 수 있어 직원 계정 발급·수정 시 `invalidate`가 체크리스트에 포함됐다.
- **측정 중 걸린 함정들**
  - 비교용 개선 전 worktree에 `.env`가 없으면 앱이 `VITE_API_BASE_URL 환경 변수가 필요합니다`를 던져 화면이 안 그려지고 LCP가 0으로 나온다 → `.env` 복사 후 다시 빌드.
  - 로컬 `.env`에 DSN이 없는 첫 빌드는 Sentry/replay가 통째로 빠져 숫자가 틀린다(10/14 절). 가짜 DSN(`VITE_SENTRY_DSN=https://public@o0.ingest.sentry.io/0 VITE_SENTRY_ENVIRONMENT=prod VITE_GA_MEASUREMENT_ID=G-TEST`)으로 빌드.
  - **완료 신호를 잘못 고르면 이상한 결과**: 업무 만들기 실험에서 "체크박스가 보이면 끝"으로 잡았더니 체크박스는 데이터 도착 전에 그려져 약 80 ms가 나왔다. 담당자 수가 찍힌 `aria-label` 개수(`/\/[1-9]/`)로 바꾸고, 폴링 간격(약 830 ms)이 오차가 되지 않게 `waitForFunction(..., {polling:'raf'})`+`performance.now()`를 썼다.
  - `pkill -f`로 서버를 끄면 자기 셸까지 죽는다(exit 144) → `ss -ltnp`로 포트의 PID를 찾아 그 PID만 종료.
- **같은 코드도 빌드에 따라 LCP가 560 ms 달라진다**(로컬 1,968 vs 배포 2,528) → 숫자보다 측정 조건을 먼저 맞춰야 한다는 회고 소재.

## 최적화 우선순위
1. **로고 중복 제거**(10분, −139kB): `public/assets/login-logo.png`와 `src/shared/assets/toyvillage-logo.png`가 **md5까지 동일**한데 `public/`과 `src/` 양쪽에 있어 Vite가 둘 다 번들에 넣는다.
2. **로고 포맷 전환**: 390×328에 139kB는 과하고 이게 **로그인 LCP 요소**. WebP 10~20kB, 로고라면 SVG 2~5kB. `profile-admin.png`(192×192, 62kB)도 같이.
3. **brotli**: 252.5→198.3 kB(−21%), 서버 설정만. Cloudflare Workers는 기본 지원이니 켜져 있는지만 확인.
4. **폰트 preconnect**: 폰트가 jsdelivr CDN에서 오는데 `preconnect`가 없어 DNS+TLS(~385ms)가 렌더 경로에 들어간다(`index.html`). `<link rel="preconnect" href="https://cdn.jsdelivr.net" crossorigin>` 한 줄, 더 확실히는 자체 호스팅+`font-display: swap`.
5. **라우트별 `React.lazy`+`Suspense` + vendor 청크 분리**(1일, 가장 큰 효과): 초기 JS 절반 이하 예상. 직후에 [[번들-회귀-방지-CI와-ESLint-복잡도-도입]]의 size-limit.
6. **목록→상세 prefetch(`prefetchQuery` on hover) + 캐시 seed(`setQueryData`)**: 상세 진입 요청 1회→0회.
7. **대시보드 7개 API**: 이미 병렬. 카드별 독립 로딩, 계측으로 병목 확인 후 필요 시 백엔드 집계 API 요청.
8. 컴포넌트 분리(복잡도 상위 `IndividualDetailPage` 32, `DataTable` 30·854줄): 렌더 성능보다 유지보수 목적.
- 1~4번은 코드 구조를 안 건드리고 LCP를 1초 이상 줄일 수 있어 **스플리팅보다 먼저**.

## 하지 않기로 한 것(근거)
- **리스트 가상화**: 서버 페이지네이션이라 한 화면의 행이 적어 이득 없이 복잡도만 증가.
- **Emotion→Tailwind 전환**: `styled` 178곳, 병목은 런타임 스타일이 아니라 초기 번들.
- **성급한 `memo`/`useCallback`**: 측정(Sentry INP) 후 실제 느린 곳만.
- **Lighthouse CI 먼저**: 스플리팅 전엔 기준선을 못 정함.

## 기술 스택 선택 이유 설명 시 주의(포트폴리오·면접 대비)
- **React**: "SEO 필요 없어서 React"가 아니라 "로그인 뒤 어드민이라 SSR·SEO 이득이 없어 Next.js 대신 Vite+React CSR". CRA는 2025년 공식 지원 종료.
- **axios**: "로딩 처리"는 axios가 아니라 TanStack Query(`isPending`) 담당. axios의 실제 이점은 fetch와 달리 4xx/5xx를 에러로 던짐, 인터셉터, 기본 타임아웃(`timeout: 10_000`), JSON 자동 변환.
- **Yarn Berry**: 이 저장소는 PnP를 끄고 `.yarnrc.yml`에 `nodeLinker: node-modules`로 쓴다 — "berry인데 PnP는?" 질문에 답할 수 있어야 한다.
- **react-router**: 선택 이유보다 쓰면서 한 판단이 답이 된다 — 조회 조건(탭·날짜·페이지)을 `searchParams`로 URL이 소유해 상세에 다녀와도 유지, `RequireAuth` 라우트 가드.
- **emotion**: styled-components는 2025년 3월 유지보수 모드(새 기능 중단, 보안 패치만). zustand는 서버 데이터를 TanStack Query가 맡아 전역 상태가 인증·UI 정도라 Redux는 과함. playwright는 멀티 브라우저·무료 병렬·자동 대기로 flaky 적음(e2e 시나리오 1,068개).

## 관련
- [[Vite-빌드타임-환경변수-인라인]]
- [[ToyVillage-Admin-FE/프로젝트-현황]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-19~20 (최종 재측정 `6dd3985` vs `14721cb`, 업무 만들기 실험실 측정, 측정 함정)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-3) — 2026-10-20 (#262 이미지 WebP 변환 결과, WebP 설명, 파비콘 후보)
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-14 (before 재측정, 배포 빌드 DSN 함정, 번들 구성, 업무 만들기=API 대기)
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-04 (세션 내용은 2026-09-22~28 논의)
