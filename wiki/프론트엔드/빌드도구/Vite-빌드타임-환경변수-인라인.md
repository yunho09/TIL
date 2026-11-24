---
tags: [vite, frontend, 환경변수, 빌드]
updated: 2026-11-24
---

# Vite 빌드타임 환경변수 인라인

Vite는 `import.meta.env.VITE_*` 값을 런타임에 읽지 않고 **빌드 시점에 번들 코드에 문자열로 치환**한다. `VITE_` 접두사가 붙은 값은 그래서 사실상 공개값이고, 값을 바꾸려면 재빌드·재배포가 필요하다.

## 메커니즘
- `vite.config`(또는 별도 `vite.env.ts`) 안 헬퍼가 `define: { 'import.meta.env.VITE_XXX': JSON.stringify(value) }` 형태로 esbuild/rollup `define` 옵션에 값을 박아 넣는다.
- 빌드된 산출물을 열어보면 값을 참조하던 함수가 상수로 컴파일돼 있다. 예: 키가 비어 있으면 `function qs(){return``}`.
- `.env.local`(로컬)이나 CI/배포 환경변수(예: Portainer)에 값을 넣어도 **다시 빌드**해야 브라우저에 반영된다. 서버를 재시작해도 이미 빌드된 정적 파일은 바뀌지 않는다.

## 시사점
- `VITE_*` 값은 브라우저 devtools에서 누구나 꺼내볼 수 있다 — **비밀값(secret)을 넣으면 안 된다.** 외부 API가 승인키를 URL/도메인 단위로 제한하는 방식([[도로명주소-검색-API-신청]] 참고)과 조합해야 그나마 안전하게 쓸 수 있다.
- 진짜 비밀을 숨기려면 프론트가 직접 호출하지 말고 백엔드에 프록시 엔드포인트를 두고 키를 서버에만 보관해야 한다.
- 배포된 번들에서 값이 실제로 들어갔는지 확인하려면 `dist/` 산출물의 컴파일된 함수를 직접 열어보는 게 가장 확실하다.

## 모드별 env 파일과 공개값 env 파일 커밋 (2026-11-23 추가)
- `vite build`는 production 모드, `--mode staging`이면 `.env` + `.env.staging`을 읽고 **모드 파일이 이긴다**. `.env`는 항상 로드되므로 모드 파일에서 키가 빠지면 조용히 `.env` 값으로 폴백한다 → CI에서 "빌드 결과에 그 환경 주소가 들어 있는지" 가드를 두면 잡힌다. `import.meta.env.PROD`는 staging 빌드에서도 true.
- 일반 원칙: `.env`·`.env.local`·`*.local`과 DB 비밀번호·API secret·private key 파일은 커밋 금지(Vite 문서도 `*.local`을 gitignore 하라고 안내). **`VITE_*` 공개값만 든 `.env.staging`/`.env.production`은 커밋해도 새로 드러나는 정보가 없다.** 단 레포가 public이면 나중에 누가 비밀값을 넣을 수 있으니 파일 맨 위에 "VITE_* 값은 JS에 그대로 노출, 비밀값 금지" 주석을 두고 PR 리뷰에서 확인.
- 안전성 증명법: 운영 중인 사이트의 JS를 내려받아 각 값이 이미 들어 있는지 개수를 센다(xquare: 프론트 주소 1·백엔드 `/api/v1` 33·CLIENT_ID 1·APP_SLUG 1, stag 값과 prod 값이 서로 섞이지 않음). 비어 있던 값은 `gh api /apps/<slug> --jq .client_id`로 채울 수 있고 운영 번들 값과 일치하면 맞는 값.
- 값별 이유: 백엔드 주소는 F12 Network 탭에 다 보이고 보안은 주소 은닉이 아니라 백엔드 인증(로그인 없이 호출 시 401)이 맡는다. OAuth **Client ID**와 App slug는 공개 식별자. 반대로 **Client Secret·Private Key(.pem)·Webhook Secret·JWT 서명 키**는 백엔드 전용이며 프론트 env에 절대 금지.
- 대안: GitHub Variables로 CI가 주입(레포에 파일 없음, 값 변경 시 커밋 불필요)하지만 워크플로 단계가 늘고 로컬 stag/prod 빌드는 값을 따로 구해야 한다. xquare는 커밋 방식 채택 → [[새-서버-이전-runner-자동배포-실행과-도메인-전환-함정]].

## 값이 비면 코드가 통째로 빠지는 원리: 상수 치환 → 죽은 코드 제거 → 트리 셰이킹 (2026-11-24 추가)
```ts
const dsn = import.meta.env.VITE_SENTRY_DSN
if (!dsn) return
Sentry.init({ dsn })        // .env에 값이 없으면 이 줄 이후가 빌드에서 사라짐
```
1. **치환**: 빌드 중 `import.meta.env.VITE_SENTRY_DSN`이 `.env` 값 그대로 박힌다. 값이 없으면 `const dsn = ""; if (!"") return`.
2. **죽은 코드 제거**: 압축(minify) 단계에서 `!""`는 항상 true라 "무조건 return"으로 접히고, `return` 뒤의 `Sentry.init(...)`은 실행될 일이 없어 지워진다.
3. **트리 셰이킹**: `Sentry.init`을 부르던 코드가 없어지니 그걸 위해 import한 Sentry 코드도 아무도 안 쓰는 코드가 돼 같이 빠진다.
- 사람이 뺀 게 아니라 **빌드 도구가 자동으로** 뺀다. 사례(토빌): 로컬 `.env`에 DSN이 없어 번들에 Sentry가 27 kB만 남은 가벼운 빌드(메인 JS 1,031 kB)를 재고 있었고, **가짜 DSN을 넣은 배포용 빌드**는 1,306 kB였다.
- 트리 셰이킹 두 종류 구분: ① 어디서도 import하지 않은 export를 결과물에서 빼기(`utils.ts` 함수 10개 중 1개만 import하면 9개 제외), ② 절대 실행되지 않는 분기 제거(위 경우). 배럴 파일은 ①이 항상 되지 않는 구간이 있다 → [[라우트-코드-스플리팅-React-lazy-Suspense-key와-배럴]].
- 교훈: **성능 수치는 사용자가 받는 것과 같은 조건의 빌드로 잰다.** 환경변수가 번들 내용을 바꾸는 항목은 측정 전에 배포와 같게 맞춘다 → [[어드민-SPA-번들-성능-baseline과-최적화-우선순위]].

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.config/Claude/scratch-workspaces/68eef3b8-da94-4db5-b563-9565e29705a3/4cebf249-3d72-4a77-a55b-e742194ef06a/scratch-2026-11-20-d28dbc) — 트리 셰이킹·죽은 코드 제거 단계 설명 (2026-11-24)
- Claude Code 세션 자동 캡처 (/data/project/Commonly-fe) — `VITE_JUSO_CONFM_KEY`(도로명주소 검색 API 승인키) 인라인 여부를 번들 코드로 직접 검증한 사례
- Claude Code 세션 자동 캡처 (/data/project/xquare-infra-frontend-v3) — stag/prod env 파일 커밋 결정과 운영 번들 대조
