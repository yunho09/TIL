---
tags: [nextjs, turborepo, monorepo, npm-workspaces, emotion]
updated: 2026-10-29
---

# Next.js 16 + Turborepo + npm workspaces 모노레포 세팅

앱 2개와 공유 패키지 1~2개 규모에서는 Nx보다 **Turborepo + npm workspaces**가 가볍다. Next 16(Turbopack)은 workspace 패키지를 자동 변환해 주므로 `transpilePackages`가 필요 없다. 적용 사례: [[어드민-도입-모노레포-결정과-Figma-디자인시스템-정리]].

## Turborepo vs Nx
- Turborepo: `turbo.json` 하나, 기존 npm 스크립트 그대로, Next와 궁합 좋음. 하는 일은 `turbo dev`(앱 동시 실행), `--filter=admin`(하나만 빌드), 빌드·린트 캐시.
- Nx: `nx.json`·플러그인·프로젝트별 설정, 큰 조직·다언어·수십 프로젝트용. `@nx/next`가 새 Next를 늦게 따라올 수 있음.
- 패키지 매니저는 pnpm이 엄격·빠르지만 팀 전원 전환 비용이 있어 처음엔 **기존 npm 유지**.

## 구조와 turbo.json
```
apps/openapi, apps/admin
packages/ui          (colors.ts, button, input, modal, sidebar)
package.json         "workspaces": ["apps/*", "packages/*"], devDeps: turbo
turbo.json, package-lock.json (루트에 하나만)
```
```json
{ "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "!.next/cache/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {} } }
```

## Next 16에서 주의할 점 (설치된 Next 문서 기준)
- **공유 패키지**: Turbopack이 workspace 패키지를 자동 변환 → `packages/ui`는 빌드 없이 `.tsx`를 export 가능.
- **루트 인식**: lockfile 위치로 루트를 찾으니 lockfile을 루트에 하나만 두면 추가 설정 없음.
- **Docker `output: "standalone"`**: 각 앱 `next.config`에 `outputFileTracingRoot: path.join(__dirname, "../../")`를 넣어야 `packages/ui`가 결과물에 포함됨.
- **Emotion**: 공유 패키지에서는 `peerDependencies`로만 둔다. 두 벌 설치되면 앱의 `CacheProvider`를 못 찾아 스타일이 깨짐. `EmotionRegistry`는 앱마다 하나.

## 이전 순서(PR 분리)
1. 세팅만: `git mv`로 기존 앱을 `apps/openapi`로(이력 유지), workspaces·turbo 추가, lockfile 재생성, 빌드·dev 동일 확인(기능 변경 없음).
2. 공유 UI 패키지로 `colors.ts`와 공통 컴포넌트 이동.
3. 새 앱(admin) 추가.

배포처(Vercel vs 자체 Docker)에 따라 1단계 설정이 달라지는데 레포에 배포 설정이 없어 미확인이었다.

## 1단계 실제 적용 결과 (OPN-6, PR #20, 2026-10-28)
- `src`·`next.config.ts`·`tsconfig.json`·`eslint.config.mjs`·`package.json`을 `git mv`로 `apps/openapi`에 이동(파일 92개, 이력 유지). 앱 패키지 이름 `openapi` → `turbo run dev --filter=openapi`로 한 앱만 실행.
- **`AGENTS.md`도 앱 폴더로 같이 이동**: `next dev`가 실행되는 앱 폴더에 이 파일을 다시 쓰기 때문.
- 루트 `package.json`에 `workspaces`, `turbo`, `packageManager: npm@x`. `.gitignore`는 `node_modules`·`.next` 패턴이 하위 폴더에도 걸리게 바꾸고 `.turbo` 추가.
- `package-lock.json`은 지우지 않고 그 위에 재설치 → 추가된 건 turbo와 workspace 연결뿐, 기존 패키지 버전 불변.
- 검증: `npm run build` 동일한 27페이지 생성, 두 번째 빌드는 Turbo 캐시로 8ms, lint 통과, dev 서버 주요 경로 200. 3000번 포트가 다른 Next 서버에 점유돼 3001로 뜸(기존 서버는 건드리지 않음).
- 커밋을 둘로 나눔: "이동만"(내용 변경 없음 → 이력 유지 확인 쉬움) / "workspaces·Turborepo 설정". PR 본문 첫 줄 `Closes OPN-6`로 Linear 연동, 팀원은 브랜치 전환 후 루트에서 `npm install` 1회 필요.
- 배포 전이라 `outputFileTracingRoot`는 넣지 않음. 나중에 Vercel이면 앱마다 프로젝트를 만들고 Root Directory를 `apps/<app>`로, 자체 Docker면 `output: "standalone"` + `outputFileTracingRoot`.

## admin 앱 추가와 앱 이름 정리 (같은 PR #20, 2026-10-29)
- `apps/admin`을 빈 앱(임시 홈, `dev`·`start` 포트 **3001**)으로 같은 PR에 추가. Next·React·Emotion 버전과 React 컴파일러 설정을 기존 앱과 **똑같이** 맞춤 → 나중에 컴포넌트를 공유할 때 버전 차이로 깨지는 것을 방지. `EmotionRegistry`·`globals.css`·폰트는 일단 복사하고 `packages/ui` 단계에서 한 곳으로 합칠 계획.
- 앱 이름을 `apps/openapi` → **`apps/user`**로 `git mv` 개명(Figma 페이지명 user/admin과 일치). 이후 `--filter=user`, `npm install <패키지> -w user`. lockfile은 workspace 연결 이름만 바뀜.
- `npm run dev`는 두 앱을 동시에 띄우므로 포트를 나눠 둬야 하고(user 3000 / admin 3001), 팀 안내에도 "admin은 3001" 한 줄이 필요하다.

## Turbo 설정에서 CodeRabbit 리뷰로 잡은 함정 (PR #20)
- **`start`에 `dependsOn: ["build"]`를 걸면 안 된다.** Turbo 캐시는 실행한 머신에만 있고 git에 안 올라간다. 새 서버·Docker·CI처럼 캐시가 없는 환경에서는 `.next`가 이미 있어도 `next build`를 다시 돌려, ① 시작이 느려지고 ② 실행 서버에 소스·devDependencies가 없으면 빌드 실패로 서버가 안 뜨며 ③ `next/font/google`(Roboto Mono)은 **빌드 때 Google에서 폰트를 받으므로** 외부 접속이 막힌 서버면 실패한다. 빌드와 실행은 분리하는 게 정석 → `dependsOn` 제거, `build` → `start` 순서는 사람이 지킴. 빌드 없이 `next start`하면 "빌드 결과 없음" 에러로 멈춤. `dev`는 빌드 불필요.
- **루트 `npm start` 누락**: 루트 scripts에 `start`가 없어 안 됨 → `turbo run start` 추가(앱이 둘이라 둘 다 실행).
- **빌드 outputs에 `.next/dev` 혼입**: Next 16은 개발 서버 캐시를 `.next/dev`에 두므로 `outputs`에 `"!.next/dev/**"`도 제외.
- **`PORT=4555 npm run dev`가 무시됨**: Turbo는 `turbo.json`에 선언하지 않은 환경변수를 걸러낸다 → dev·start 작업에 `passThroughEnv: ["PORT"]`. 직접 실행해 무시되는 것을 확인한 뒤 고쳤다.

## 도구 역할 요약
- **npm workspaces**(바탕): 루트 `package.json`의 `"workspaces": ["apps/*", "packages/*"]` 한 줄. 루트에서 `npm install` 1회로 공용 `node_modules`, 사내 패키지를 npm 게시 없이 `import { Button } from "@xquare/ui"`처럼 사용.
- **Turborepo**(그 위): 여러 앱에 명령 일괄 실행(`--filter`로 선택), 결과 캐시(6초 → 8ms), 순서 보장. `dependsOn: ["^build"]`의 `^`는 "내가 쓰는 패키지들 먼저".

## 관련
- [[Vite-모노레포-워크스페이스-패키지-dev서버-캐시]]
- [[Cloudflare-Workers-Builds-모노레포-앱별-Worker]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/xquare-openapi-frontend/node_modules/next/dist/docs)
