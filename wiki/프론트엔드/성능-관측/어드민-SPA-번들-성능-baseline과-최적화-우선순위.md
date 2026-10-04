---
tags: [performance, bundle, code-splitting, vite, lcp, baseline]
updated: 2026-10-04
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
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-04 (세션 내용은 2026-09-22~28 논의)
