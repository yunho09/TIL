---
tags: [nextjs, turborepo, monorepo, npm-workspaces, emotion]
updated: 2026-10-28
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

## 관련
- [[Vite-모노레포-워크스페이스-패키지-dev서버-캐시]]
- [[Cloudflare-Workers-Builds-모노레포-앱별-Worker]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/xquare-openapi-frontend/node_modules/next/dist/docs)
