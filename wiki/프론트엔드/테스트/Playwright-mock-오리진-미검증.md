---
tags: [playwright, e2e, testing, mocking]
updated: 2026-09-16
---

# Playwright mock이 경로만 보고 오리진을 안 보면 서버가 바뀌어도 안 걸린다

`page.route()` 매처가 URL의 pathname만 비교하고 origin(스킴+호스트+포트)을 확인하지 않으면, 요청이 의도한 서버가 아닌 **다른 서버**로 가는 회귀가 생겨도 경로 문자열만 같으면 mock이 그대로 잡아 응답해버려 테스트가 계속 초록불이다.

## 구멍이 있던 매처

```ts
await page.route((url) => filePaths.includes(url.pathname), ...)
```

이러면 아래 두 회귀를 못 잡는다.

- 파일 요청이 공용 API axios로 되돌아가 `https://api.example.com/...`(API 서버)로 감 → 경로가 같으니 mock이 그대로 잡아 200 → **테스트 통과**
- 환경변수를 안 읽고 상대 경로로 요청해 앱 자기 오리진(`localhost:5199`)으로 감 → 이것도 통과

"파일 서버에서 받는다"를 검증한다면서 정작 **어느 서버로 갔는지는 확인하지 않은 것**이다.

## 고친 매처

```ts
const fileServerOrigin = new URL(
  process.env.VITE_FILE_BASE_URL ?? 'https://cdn.e2e.invalid',
).origin

await page.route(
  (url) => url.origin === fileServerOrigin && filePaths.includes(url.pathname),
  ...
)
```

기대값(요청 기록)에도 오리진을 함께 넣어 단언한다.

```ts
expect(requests).toEqual([
  { origin: fileServerOrigin, path: `/${encodeURIComponent(fileKey)}`, authorization: undefined },
])
```

이러면 요청이 API 서버나 앱 자신에게 새는 회귀가 나는 즉시 실제 네트워크로 나가 실패한다(그 호스트가 존재하지 않는 `*.invalid`면 확실히 빨간불이 된다).

## 로컬에서만 실패하는 함정과 처리

이 검증에 처음부터 오리진을 넣지 못했던 이유가 있다 — [[Playwright-reuseExistingServer-실서버-유출]]처럼 로컬에서 `yarn dev`를 띄운 채 Playwright를 돌리면 `reuseExistingServer`가 그 서버를 재사용하는데, 이때는 `playwright.config.ts`의 `webServer.env`가 적용되지 않고 실행 중이던 서버의 `.env` 값이 그대로 쓰인다. 기대 오리진을 고정 상수(`cdn.e2e.invalid`)로 박아두면 CI·격리 실행에서는 맞지만 로컬 재사용 환경에서는 값이 어긋나 실패한다.

해결은 기대 오리진을 환경변수에서 읽게 하는 것 — CI·격리 실행은 기본값(`webServer.env`가 심어주는 고정 오리진)을 쓰고, 로컬에서 재사용 서버로 돌릴 땐 실행 서버와 같은 값을 넘겨준다.

```bash
VITE_FILE_BASE_URL=https://cdn.toyvillage.kr yarn playwright test tests/e2e/attachment-download.spec.ts
```

## 관련
- [[네트워크-레벨-모킹]] — route 매칭 정확도 함정 일반(등록 순서, 쿼리스트링 등)에 오리진 검증 항목 추가
- [[Playwright-reuseExistingServer-실서버-유출]] — 같은 재사용 구조가 일으키는 다른 갈래 문제(mock 안 된 요청이 실서버로 나가는 유출)

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop) — [[ToyVillage-Admin-FE/프로젝트-현황]] (#71 첨부 다운로드 e2e, CodeRabbit 리뷰로 발견)
