---
tags: [cloudflare, workers, monorepo, nx, wrangler, deploy]
updated: 2026-10-01
---

# Cloudflare Workers Builds — 모노레포는 앱마다 Worker 하나

Nx 모노레포(앱 3개)를 Cloudflare Workers Builds에 연결하면 기본 빌드 명령 `yarn run build`가 인자 없는 `nx build`로 터진다. 해법은 **앱마다 Worker를 하나씩** 만들고(같은 저장소·같은 브랜치), Build command와 Deploy command의 `--config`로 앱을 갈라주는 것. 운영(main)과 스테이징(develop)은 Worker 세트를 따로 둔다. 정적 호스팅·SPA fallback 기본은 [[Cloudflare-Workers-SPA-fallback-404]] 참고.

## 빌드가 터진 원인
- 루트 `package.json`의 `"build": "nx build"`는 `nx build <프로젝트>` 형태여야 하는데 Cloudflare가 인자 없이 호출 → `NX Both project and target have to be specified`. 로컬 `npx nx build`로 그대로 재현된다. 의존성 설치는 정상, 빌드 명령 한 줄에서 10초 만에 실패.
- 수정: `build` → `nx run-many -t build --parallel=3`(전체), `build:app` → `nx build`(앱 단독, `yarn build:app @jobis/student`). 프로젝트명은 `student`가 아니라 `@jobis/student`.
- 스크린샷 URL이 `/workers/services/...`면 Pages가 아니라 **Worker(Workers Builds)**. 이 경우 `wrangler.jsonc`가 없으면 빌드를 고쳐도 배포 단계에서 막힌다.

## "서비스 하나 = Worker 하나 = 앱 하나"
- 모노레포는 "저장소가 하나"일 뿐 "배포가 하나"가 아니다. Worker 하나에 3개 앱을 넣으려면 `/`·`/admin`·`/company` 경로 분리라서 Vite `base`·`<base href>`·라우터 `basename`을 전부 바꾸고, SPA 폴백을 경로별로 나누는 Worker 스크립트까지 필요 → 비권장. Worker 3개가 코드 수정 없이 가장 싸다.
- 3개 Worker 모두 같은 저장소·같은 브랜치, **Root directory는 `/`**. 달라지는 건 두 명령뿐:

| 항목 | 값 (앱별로 치환) |
|---|---|
| Build command | `yarn build:app @jobis/student` |
| Deploy command | `npx wrangler deploy --config apps/student/wrangler.jsonc` |
| Version(프리뷰) command | `npx wrangler versions upload --config apps/student/wrangler.jsonc` |

- `wrangler.jsonc`(앱마다 `apps/<app>/`): `name`(= Worker 이름), `assets.directory: "./dist"`, `not_found_handling: "single-page-application"`. wrangler는 `assets.directory`를 **설정 파일 기준 상대경로**로 풀기 때문에 루트에서 `--config`로 돌려도 된다(없는 경로를 주면 `apps/student/NOPE`로 에러 나는 것으로 확인).
- **Worker 이름은 대시보드와 `wrangler.jsonc`의 `name`이 정확히 같아야** 한다. 다르면 `wrangler deploy`가 설정 파일 이름으로 새 Worker를 따로 만든다.
- 기본 프리뷰 명령 `npx wrangler preview`는 wrangler 4.24.3에 없는 명령이라 `versions upload`로 바꿔야 한다.
- GitHub Actions 배포 워크플로가 있다면 같은 방식(`yarn build:app @jobis/<app>` + `deploy --config ...`)으로 맞춰야 한다. 예전 Pages 배포 워크플로는 계정에 없는 Pages 프로젝트로 배포하고 `nx build admin`(실제 프로젝트명 `@jobis/admin`)을 불러 원래부터 실패 상태였다.

## 빌드 감시 경로(Build watch paths)
- Include `*` 그대로, **Exclude에 다른 앱 폴더만** 넣는다(`apps/admin/**`, `apps/company/**` 식). 그러면 한 앱만 바뀐 커밋은 그 앱만 빌드되고, `packages/`나 루트 설정이 바뀌면 3개 전부 빌드된다.
- 안 나누면 학생 앱만 고쳐도 3개가 전부 다시 빌드·배포된다. 푸시 한 번에 Worker마다 따로 빌드가 돌고, PR 체크도 `Workers Builds: student-v2`처럼 Worker별로 붙는다.

## 환경변수는 Worker마다, 빌드 변수 칸에
- `.env`가 저장소에 없으면 환경변수를 안 넣었을 때 **빌드는 성공하는데 번들에 `baseUrl: undefined`가 박혀** API가 전부 깨진다(Vite 인라인 원리는 [[Vite-빌드타임-환경변수-인라인]]). 로컬 `.env` 있을 땐 `baseUrl:"https://stag..."`, 없을 땐 `undefined`로 실제 확인.
- `loadEnv(mode, DIR_NAME, "")`처럼 prefix를 빈 문자열로 주면 `process.env`도 같이 읽히므로 `MODE` 없이도 `BASE_URL`은 주입된다. 다만 모드 라벨이 `development`로 남으니 Sentry 환경 구분용으로 `MODE=production`은 넣는 편이 낫다.
- 변수는 **Worker마다 따로**(공유 안 됨), 런타임 Variables가 아니라 **Builds → Variables and secrets**. Sentry DSN은 앱 이름으로 `SENTRY_<앱>_DSN`을 골라 읽는 구조라면 앱마다 자기 것만 넣는다. `SENTRY_AUTH_TOKEN`은 없어도 빌드는 되고 소스맵 업로드만 빠진다.
- 대시보드 입력창에 새로 추가한 필드는 첫 글자가 먹히는 증상이 있었다 → 입력 후 실제 값을 확인할 것.

## 운영/스테이징 Worker 분리
- 운영: `student-v2`/`admin-v2`/`company-v2`, 브랜치 `main`. 스테이징: `student-stag-v2` 등, 브랜치 `develop`, `MODE=staging`, Sentry 변수 없이.
- Worker 생성 시 첫 빌드는 브랜치 설정 전이라 기본 브랜치(develop)로 돌아 `build:app`이 없는 코드면 실패한다 — 정상. 생성 후 Settings에서 Branch control을 바꾼다. 스크립트가 아직 main에 없으면 main 연결 후에도 같은 에러가 난다(코드가 머지돼야 해소).
- 스테이징은 `wrangler.jsonc`에 `env.staging`을 추가하고 Deploy command에 `--env staging`. Workers Builds는 연결된 Worker 이름으로 배포를 강제(`WRANGLER_CI_OVERRIDE_NAME`)하므로 설정 PR이 머지되기 전에도 운영 Worker를 덮어쓰지 않는다(다만 이름 불일치 경고가 뜬다).
- **무료 플랜은 빌드를 한 번에 하나씩**만 돌린다. 운영 Worker들이 develop도 "non-production branch 프리뷰"로 빌드하면 stag 빌드가 그 뒤에 줄을 서 반영이 5~10분 늦는다. 개선안: 운영 Worker의 non-production 빌드를 끄고, 프리뷰는 stag Worker에서 켠다(프리뷰가 운영 환경변수로 도는 것도 함께 해소). 프리뷰가 운영 서버 데이터를 보기 때문에 develop 머지 직후 "prod에 반영된 것처럼" 보이는 착시가 생겼으나 실제 prod는 main 기준이라 그대로였다.

## 도메인·URL
- 커스텀 도메인: Worker → Settings → **Domains & Routes**(또는 Domains 탭) → Custom domain에서 `student-v2.jobis-dsm.kr` 식으로 등록하면 DNS 레코드(Worker 타입)와 인증서는 자동 생성·수정 불가. 목록 화면에서 Worker 이름 밑 회색 글씨로 보인다.
- `<Worker>.<계정서브도메인>.workers.dev`(Production)와 `*-<Worker>.<계정서브도메인>.workers.dev`(Preview URLs, main 외 브랜치 빌드마다)는 Cloudflare가 자동 부여. **둘 다 공개 주소**. Production workers.dev는 운영과 같은 앱이고, Preview는 PR 확인용. 끄려면 Domains & Routes에서 Disable(끄면 Preview 주소도 같이 사라진다). 성공한 프리뷰 결과에 Version ID만 있고 URL이 없으면 workers.dev가 꺼져 있어서.
- 이 URL들은 백엔드 CORS 허용 목록에 별도로 등록해야 API가 열린다(허용 Origin 문제는 [[에러-응답-CORS-헤더-누락-상태코드-차단]]). 허용 목록 검증은 `OPTIONS` preflight를 해당 Origin으로 보내 200/403을 보면 된다.
- 로그 수집(Logs/Invocation logs) 창은 정적 자산만 있는 Worker엔 불필요("Logpush cannot be added to a Worker that only has static assets"). 대시보드에서만 바꾼 설정은 다음 배포에서 `wrangler.jsonc`로 덮일 수 있으니 필요하면 `observability`를 파일에 넣는다.
- 빌드 기록은 Worker → Deployments(Recent builds), 롤백은 Version History의 `⋯` → Deploy. Active deployment의 Traffic이 `0%`로 찍혀도 표시 오류일 수 있다.

## 지운 Worker가 남기는 유령 체크
- `jobis-fe-v2` Worker를 삭제해도 GitHub 연결(빌드 트리거)이 Cloudflare 쪽에 남아 **푸시마다 `Workers Builds: jobis-fe-v2 failure — Worker does not exist`** 체크가 붙는다. 시작·종료가 같은 초라 빌드를 돌린 게 아니라 즉시 실패 처리된 것(관련 진단: [[Cloudflare-Workers-SPA-fallback-404]]).
- 필수 상태 체크가 없는 룰셋이면 머지를 막지 않는다. 같은 이름으로 Hello World Worker를 다시 만들어도 연결이 자동으로 붙지 않았고(대시보드엔 `Connect` 버튼만 보임) 오히려 공개 Worker만 하나 더 생겼다. 확실한 제거는 Cloudflare 지원 문의뿐 — **지울 거면 연결(Settings → Builds → Disconnect)을 먼저 끊고 삭제**.
- Workers Builds는 **푸시가 있을 때만** 빌드한다. Worker를 새로 만든 뒤 기존 PR 브랜치를 프리뷰 빌드하려면 새 푸시(빈 커밋 포함)가 필요하고, 대시보드의 재시도는 원래 브랜치(develop)를 다시 빌드할 뿐이다.

## 비운영 브랜치 푸시는 `versions upload`만 — stag 목록에 main 줄이 뜨는 이유
(ToyVillage-Admin-FE 사례, 단일 앱 stag/prod Worker 2개 구성. prod는 `wrangler.jsonc`의 `env.prod`를 `npx wrangler deploy --env prod`로 배포)
- Worker가 저장소 전체 push를 보므로 **배포 대상이 아닌 브랜치가 push돼도 빌드가 돈다.** Production branch가 아닌 브랜치는 Deploy command가 `npx wrangler versions upload`(버전만 올리고 트래픽은 안 돌림)로 실행된다. 진행 단계에 "Deploying"이라고 떠도 마지막 단계의 고정 이름일 뿐이다.
- 그래서 stag(Production branch `develop`)의 Versions 목록에 main 줄이 생겨도 **현재 배포(파란 막대)는 develop 버전**에 그대로 있다. 반대로 prod에는 `#200/sentry` 같은 다른 브랜치 빌드가 실패 줄로 뜰 수 있다 — 배포는 안 되지만 `--env prod` 업로드 대상이 운영 Worker이므로, 운영 Worker의 non-production branch builds를 끄거나 main만 빌드하도록 제한하는 게 안전하다.
- 어느 쪽이 실제 배포 중인지는 Versions의 파란 막대 + Active deployment의 커밋 해시, 그리고 실제 사이트(`curl`로 해당 커밋에만 있는 변경, 예: 파비콘 태그)로 교차 확인한다.
- force push·머지로 같은 내용이 두 번 푸시되면 빌드·버전 줄도 두 번 생긴다(원래 커밋 해시와 재작성 후 해시).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — [[JOBIS-FE-V2/프로젝트-현황]]
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-2) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]]
