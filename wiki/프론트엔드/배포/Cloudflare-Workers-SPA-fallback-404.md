---
tags: [cloudflare, workers, spa, deploy, wrangler, ci]
updated: 2026-09-29
---

# Cloudflare Workers SPA 새로고침 404

SPA를 Cloudflare Workers(정적 assets)로 배포했을 때 `/`는 정상인데 `/login`처럼 라우터가 만든 하위 경로를 직접 열거나 새로고침하면 404가 나는 문제의 원인과 해결법.

## 원인

- 빌드 결과물(`dist/`)에는 `index.html` 하나만 있고 `login`이라는 실제 파일은 없다.
- `/`로 들어가면 `index.html`이 오고 React Router가 클라이언트에서 `/login`으로 내부 이동시키므로 정상으로 보인다.
- 반대로 `/login`을 직접 열거나 새로고침하면 서버가 `login` 파일을 찾다가 없어서 404를 낸다. `/login`뿐 아니라 라우터가 만드는 모든 하위 경로에서 동일하게 재현된다.
- **React 코드/라우터 문제가 아니라 배포 플랫폼 설정 문제**다. 앱 코드를 고쳐도 해결되지 않는다.
- 본문 없는(body-less) 404와 `x-content-type-options` 등 Pages 전용 헤더 부재로 Workers 배포임을 curl 헤더만으로 간접 판별할 수 있었다. Cloudflare **Pages**는 보통 `index.html`로 자동 fallback한다.

## 해결 (Workers인 경우)

저장소 루트에 `wrangler.jsonc`를 추가하고 `not_found_handling: "single-page-application"`을 켠다.

```jsonc
{
  "name": "<Cloudflare 대시보드의 Worker 이름과 동일해야 함>",
  "compatibility_date": "YYYY-MM-DD",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  }
}
```

- `name`이 대시보드 Worker 이름과 다르면 새 Worker가 별도로 생성된다.
- 대시보드에서 자동 빌드·배포한다면 **Settings → Build**의 배포 명령이 `npx wrangler deploy`인지 확인해야 이 설정 파일이 실제로 반영된다.
- 확인: `curl -sI https://<도메인>/<하위경로>`가 `HTTP/2 200`을 반환하면 해결.
- 로컬 검증은 빌드 후 `wrangler dev`로 하위 경로가 200을 주는지, `wrangler deploy --dry-run`이 통과하는지로 한다.

## 해결 (Pages인 경우)

`public/_redirects` 파일에 한 줄을 추가한다.

```
/* /index.html 200
```

Vite는 `public/` 안의 파일을 그대로 `dist/`로 복사하므로 배포에 같이 들어간다.

## `wrangler.jsonc` 자체가 없으면 Workers Builds 체크가 즉시 실패

SPA fallback 404와는 별개로, 저장소 루트에 `wrangler.jsonc`가 아예 없는 브랜치는 PR의 Cloudflare Workers Builds 체크 자체가 실패할 수 있다(로컬 `yarn build`는 정상 성공).

- **판별법**: 체크 로그의 시작·종료 타임스탬프가 완전히 같은 초(예: `14:09:21` = `14:09:21`)면 빌드가 돌기도 전에 떨어진 것이다. 실제 빌드 오류라면 최소 몇십 초는 걸린다. 같은 저장소의 다른 PR이 같은 체크를 통과하는지도 대조하면 이 브랜치만의 설정 누락인지 판단할 수 있다.
- 원인은 대개 `wrangler.jsonc`가 그 브랜치가 갈라진 시점 이후에 다른 PR로 추가됐고, 이 브랜치는 그 전에 갈라져서 없는 경우다. develop과 병합하면 먼저 들어간 버전과 충돌이 날 수 있는데(예: trailing comma 차이), 내용이 같다면 develop 버전을 그대로 채택하면 된다.
- 실제 로그(빌드 명령 실행 여부 등)는 Cloudflare 대시보드에만 있고, wrangler 로그인이 안 된 환경에서는 CLI로 못 본다 — 위 타임스탬프 비교가 코드 안에서 할 수 있는 간접 진단이다.

## 원인 소재와 책임 분담

고치는 파일(`wrangler.jsonc`)은 프론트 저장소에 들어가지만, 실제 배포(빌드 명령, Worker 연결)는 Cloudflare 대시보드 쪽 설정과 맞아야 하므로 배포 권한이 있는 사람과 같이 확인해야 한다. 저장소에 배포 설정 파일이 전혀 없다면 대시보드에서 누군가 수동으로 연결해 둔 것일 가능성이 높다 — 이 경우 "SPA fallback이 꺼져 있다"고 전달하면 대시보드에서 바로 켜는 것으로도 해결된다.

## Worker와 Workers Builds는 다른 것

- **Worker**: 빌드 결과물(`dist/`)을 사용자에게 내려주는 정적 파일 호스팅. `wrangler.jsonc`의 `assets.directory`가 가리키는 폴더를 그대로 서빙할 뿐, Worker 안에서 실행되는 별도 서버 코드는 없다(정적 호스팅 용도로만 쓰는 경우).
- **Workers Builds**: GitHub 푸시에 반응해 코드를 받아 `yarn install` → 빌드 명령 → `dist` 배포까지 자동 실행하는 CI/CD 기능. PR에 뜨는 "Workers Builds: ✅/❌" 체크는 이 **배포 파이프라인**의 결과이지, 코드 품질 검사(lint, e2e)가 아니다. 빌드 명령이 `tsc -b && vite build`라면 이 단계에서 타입 체크까지는 같이 걸린다.
- 그래서 PR에 ✅가 떠도 "기능이 맞게 동작한다"는 뜻은 아니고, ❌가 떠도 "코드가 잘못됐다"는 뜻은 아닐 수 있다 — 아래처럼 배포 단계 자체(미리보기 이름, Worker 이름 불일치, GitHub 연결)에서 실패하는 경우가 있다.

## PR 브랜치 이름의 `#`이 미리보기 배포를 깨뜨림

Workers Builds는 PR 브랜치를 미리보기로 배포할 때 **브랜치 이름으로 미리보기 이름을 만드는데**, 미리보기 이름은 영숫자와 `- _ / . + =`만 허용한다. `#200/sentry`처럼 이슈 번호를 `#`으로 붙이는 브랜치 컨벤션과 충돌해 `Invalid preview name` 에러로 실패한다.

- 코드·빌드 자체는 문제없다. 로컬 `yarn build`는 정상 성공한다.
- 미리보기가 필요 없는 Worker(예: PR 브랜치까지 빌드할 필요 없는 운영 Worker)라면, 그 Worker 설정에서 **"Builds for non-production branches"를 끄면** PR·develop 등 production branch가 아닌 브랜치에서는 아예 돌지 않아 이 문제가 사라진다.
- 관련: [[Git-브랜치명-샵-이스케이프]] — 같은 `#`이 로컬 셸에서는 주석으로 잘리는 다른 이유로 문제된다.

## stag/prod를 Worker 두 개로 나눌 때: `wrangler.jsonc`의 `name` 불일치

저장소 하나로 스테이징·운영 두 Worker를 운영하는 구조(예: develop→stag Worker, main→운영 Worker)에서는 `wrangler.jsonc`의 `name`이 배포 대상을 고정하는 값이라는 점이 함정이 된다.

- `name`이 한쪽 Worker 이름(예: `toyvillage-admin-fe-stag`)으로 고정돼 있으면, 다른 쪽 Worker(`toyvillage-admin-fe`)로 배포될 때 이름이 안 맞아 배포가 거부될 수 있다.
- **production branch 지정과 "non-production branches 빌드 여부"는 별개 설정이다.** production branch를 `main`으로 지정해도 "Builds for non-production branches"가 켜져 있으면 `develop`이나 PR 브랜치까지 그 운영 Worker가 따라 빌드하다가(그리고 위 이름 불일치로) 계속 실패를 남길 수 있다.
- 운영 브랜치(`main`)가 develop보다 수백~수천 커밋 뒤처져 있고 `wrangler.jsonc`조차 없다면, 그 운영 Worker는 이 저장소로 정상 배포된 적이 한 번도 없었을 가능성이 있다 — merge 전에 이름 분기(wrangler 환경 설정 등)부터 정리해야 나중에 배포 단계에서 또 실패하지 않는다.

## Build variables vs Runtime variables

Worker 설정에는 이름이 비슷한 두 변수 칸이 따로 있다.
- **Variables and Secrets** (런타임): Worker가 요청을 처리할 때 참조하는 값.
- **Settings → Build 안의 Build variables** (빌드 시점): `yarn build`를 실행할 때만 참조되는 값.

`VITE_*` 값은 Vite가 **빌드 시점에 번들 문자열로 그대로 박아 넣으므로**([[Vite-빌드타임-환경변수-인라인]]) Build variables 쪽에 넣어야 적용된다. 빌드 도구가 쓰는 토큰(예: 소스맵 업로드용 인증 토큰)도 마찬가지다. 런타임 변수 칸에 넣으면 빌드에 반영되지 않고 조용히 무시된다.

## GitHub App 연결 경고 "Error fetching GitHub User or Organization details"

Worker의 Build 설정 화면에 이 경고가 떠 있으면 Cloudflare가 GitHub 조직 정보를 못 읽고 있다는 뜻이고, 이 상태에서는 PR 미리보기뿐 아니라 **정상 브랜치(develop 등) 배포까지 실패할 수 있다**(밀려 있던 빌드가 몇 분 뒤 뒤늦게 시작되며 저절로 정상화되기도 한다).

- 흔한 원인: GitHub 조직에 설치된 "Cloudflare Workers and Pages" App의 저장소 접근 권한이 빠졌거나, 새 권한 요청이 조직 승인 대기 중이거나, 앱을 설치한 계정이 조직 권한을 잃은 경우.
- 해결: GitHub 조직 Settings → GitHub Apps → Cloudflare Workers and Pages → Configure에서 저장소 접근·대기 중인 권한 요청 확인. 그래도 안 풀리면 연결을 끊고 저장소를 다시 연결한다(재연결 후 빌드 설정을 다시 확인해야 할 수 있음).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #134, PR #135)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-3) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #133, PR #136 — Workers Builds 즉시 실패 진단법 추가)
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #200, PR #202 — Worker/Workers Builds 개념, PR 브랜치 `#` 미리보기 실패, stag/prod 이중 Worker 이름 불일치, Build/Runtime 변수 구분, GitHub App 연결 경고 진단)
