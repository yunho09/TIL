---
tags: [cloudflare, workers, spa, deploy, wrangler, ci]
updated: 2026-09-17
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

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #134, PR #135)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-3) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (이슈 #133, PR #136 — Workers Builds 즉시 실패 진단법 추가)
