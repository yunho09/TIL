---
tags: [vite, rolldown, 청크, 캐시, 번들, sentry]
updated: 2026-10-25
---

# Vite 청크 그룹으로 자주 안 바뀌는 라이브러리 분리하기 (해시 캐시)

빌드 결과의 **청크** 파일 이름 뒤 해시는 내용 지문이라, 청크 안에 "자주 바뀌는 코드"와 "거의 안 바뀌는 라이브러리"가 섞이면 라이브러리는 그대로인데도 사용자가 매 배포마다 다시 받는다. 청크 그룹(`vite.config.ts`)으로 라이브러리만 따로 떼면 해결되고, 이때 **그룹 순서**를 잘못 두면 첫 로딩이 오히려 커진다. (ToyVillage-Admin-FE 이슈 #267 / PR #268)

## 개념
- **청크**: `yarn build` 시 빌드 도구(Vite 8, 내부는 rolldown)가 수백 개 파일을 합친 묶음 파일(`react-*.js`, `sentry-*.js`, `index-*.js`, 화면별 `LoginPage-*.js` …).
- **해시**(`-CB8Y6S8N`): 청크 내용으로 계산한 지문. 한 글자라도 바뀌면 이름이 바뀌고, 브라우저는 **같은 이름이면 캐시를 재사용**한다. 그래서 배포 때 바뀐 청크만 이름이 바뀌어야 한다.

## 문제: 이름이 연쇄로 바뀐다
- 세션 녹화 청크 `sentryReplay-*.js`(127 kB)에 라이브러리 `@sentry/replay`(약 125 kB)와 우리 설정 코드(약 2 kB)가 함께 있었다.
- 설정 코드가 `index`(앱 공통 청크)의 함수를 import → 청크 코드 안에 `from "./index-C46uAD-_.js"`처럼 **index의 파일 이름이 글자로 박힘**.
- 화면 코드만 고쳐도 `index` 해시 변경 → 녹화 청크 내용 변경 → 녹화 청크 해시 변경 → 125 kB를 매번 재다운로드.

## 해결: 라이브러리만 별도 그룹
```ts
groups: [
  { name: 'react',  test: /node_modules\/react.../, tags: ['$initial'] },
  { name: 'sentry', test: /node_modules\/@sentry\//, tags: ['$initial'] },
  { name: 'vendor', test: /node_modules\//,          tags: ['$initial'] },
  // 맨 뒤에 추가
  { name: 'replay', test: /[\\/]node_modules[\\/]@sentry[\\/]replay[\\/]/ },
],
```
- `test` 정규식의 `[\\/]`는 `/`와 Windows `\` 둘 다 받으려는 것.
- **`tags: ['$initial']`은 "첫 화면에 바로 필요한 것"이라는 뜻.** 녹화는 화면이 뜬 뒤 동적 `import()`(`initSentry.ts`의 `loadReplay`)로 불러오므로 붙이면 안 된다.
- 결과: `sentryReplay` 126.82→0.66 kB(여전히 index를 import해 이름은 바뀌지만 작아서 무방), `replay` 126.19 kB(sentry 청크만 import, index를 안 봄 → 화면 코드가 바뀌어도 이름 유지), `sentry` 172.31 kB 불변, 첫 로딩 목록(index·react·sentry·vendor) 불변. 스테이징 빌드에서도 같은 결과.

## 그룹 순서가 중요한 이유 (함정)
- 그룹 규칙: ① **앞 그룹이 먼저 모듈을 가져간다** ② 가져갈 때 그 모듈이 쓰는 의존 모듈도 같이 가져간다(`includeDependenciesRecursively`, 기본 켜짐).
- `replay`를 `sentry` 앞에 두면 녹화 라이브러리가 쓰는 `@sentry/core`(첫 화면에서도 쓰임)까지 `replay`가 가져가, 첫 화면이 `replay` 청크 전체(약 190 kB)를 바로 받게 됐다. **맨 뒤**로 옮기면 `sentry` 그룹이 공용 코드를 먼저 가져가고 `replay`엔 라이브러리만 남는다. 코드 주석에 이유를 남김.

## 검증 방법
1. **`VITE_SENTRY_DSN`이 비면 녹화 코드가 빌드에서 통째로 빠진다** → 로컬 비교는 가짜 DSN을 넣고 빌드해야 한다. ([[Vite-빌드타임-환경변수-인라인]])
2. 산출물에서 `from "./..."`를 검색해 청크별 import 관계 확인.
3. `dist/index.html`의 첫 로딩 목록에 분리한 청크가 없는지 확인.
4. 앱 코드에 임시 한 줄을 넣고 재빌드 → `index`·설정 청크 이름은 바뀌고 `replay`는 그대로인지 확인 후 되돌림.

## 라이브러리 청크는 4개, 전체 빌드는 87개 (2026-10-25 추가)
- 라이브러리 청크 = 첫 로딩용 3개(`react`: react·react-dom·scheduler·react-router, `sentry`: `@sentry/*` 녹화 제외, `vendor`: 첫 화면용 나머지) + 첫 화면 뒤 1개(`replay`). "3개로 쪼갰다"는 부정확.
- 지연 로딩 화면에서만 쓰는 라이브러리(pdfjs)는 vendor에 넣지 않고 그 화면 청크(`pdf`)에 둔다 — 넣으면 그 화면에 안 가는 사용자도 받음.
- develop(`14721cb`)을 가짜 DSN으로 빌드한 전체 JS 청크 **87개** + pdf 워커 `.mjs` 1개:

| 묶음 | 개수 | 비고 |
|---|---|---|
| 진입 `index` | 1 | 150 kB (gzip 40) |
| 첫 로딩 라이브러리 | 3 | react 287 / sentry 172 / vendor 107 kB |
| 번들러 런타임 `rolldown-runtime` | 1 | 0.69 kB |
| 세션 녹화 | 2 | `replay` 126 + `sentryReplay` 0.66 kB, 첫 화면 뒤 |
| pdfjs | 1+워커 | 431 kB (gzip 129), PDF 화면에서만 |
| 화면(라우트) | 46 | 0.25–38.8 kB |
| 화면 공유 코드 | 33 | 0.3–33 kB |

- 첫 로딩 JS = index+react+sentry+vendor+런타임 ≈ 717 kB(gzip 229). 로그인 화면은 `LoginPage` 10.7 kB가 더해져 728 kB(gzip 233) — 포트폴리오 수치와 일치.
- 공유 청크 이름은 안에 든 모듈 하나에서 따올 뿐이다(예: `ReservationBackLink` 31 kB는 예약 화면 3개가 같이 쓰는 코드 묶음이지 그 컴포넌트 하나가 아님).

## 효과 범위와 설명 방법 (2026-10-25 추가)
- 효과는 **처음 방문이 아니라 배포 뒤 재방문**에 난다(처음 받는 전체 크기는 거의 동일). 첫 로딩 JS를 줄이는 건 라우트 코드 스플리팅의 일.
- 분리 전 측정(9/22, 커밋 `7487430`): 단일 청크 957.66 kB(gzip 260.46), 그중 vendor 351 kB(45.9%). 분리 후 "화면 코드만 바꾼 배포 뒤 재방문 시 새로 받는 JS"는 아직 미측정 — 숫자를 넣으려면 이 값을 전/후로 재야 정확.
- 면접·포트폴리오 표현: "번들 크기를 줄였다"(X) → **"배포 뒤 다시 받는 양을 줄였다"**. "코드 스플리팅"만 말하면 화면별 지연 로딩과 섞이므로 "라이브러리 청크 분리"로 구분.
- 꼬리질문 답: 전부 따로 쪼개지 않은 이유는 파일 수(요청 수)가 늘기 때문이라 바뀌는 주기가 비슷한 것끼리 묶음(React 계열/Sentry/나머지). 관련 커밋 `d236255`(첫 로딩 라이브러리 분리), `9dfb06a`(녹화 분리).

## 기타
- 이슈 본문의 패키지명 `@sentry-internal/replay`는 설치된 Sentry v11에서는 `@sentry/replay`였다(단일 파일). 이슈 속 이름을 그대로 믿지 말고 `node_modules`로 확인.
- 관련: [[Sentry-React-SPA-에러추적-도입]], [[어드민-SPA-번들-성능-baseline과-최적화-우선순위]], [[배포-반영-확인-번들-해시-대조]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-3) — [[ToyVillage-Admin-FE/프로젝트-현황]]
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE, 2026-10-25)
