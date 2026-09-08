---
tags: [frontend, vite, monorepo, hmr]
updated: 2026-09-08
---

# Vite 모노레포 워크스페이스 패키지 dev 서버 캐시

Turborepo/bun 같은 모노레포에서 `packages/*` 워크스페이스 패키지를 고쳤는데 dev 서버가 계속 옛날 코드로 동작한다면, 코드 문제가 아니라 **dev 서버가 옛날 변환 캐시를 그대로 서빙**하고 있을 가능성이 높다.

## 증상

- 디스크의 소스 파일은 분명히 수정됐다.
- 그런데 브라우저가 받는 번들(devtools에서 확인 가능)은 수정 전 코드 그대로다.
- 앱 자체(`apps/*`) 코드를 고치면 HMR이 바로 반영되는데, 앱 밖의 워크스페이스 패키지(`packages/*`)를 고치면 반영이 안 된다.

## 원인

`packages/*`는 앱 루트 밖에 있는 워크스페이스 패키지라 Vite dev 서버 입장에서 `/@fs/...` 경로로 서빙된다. 이 경로는 HMR 워처가 앱 루트 기준으로 잡혀 있으면 변경 감지·재변환 대상에서 빠지기 쉽고, 한번 변환된 모듈이 캐시(`node_modules/.vite`)에 그대로 남는다. 그래서 파일은 바뀌었는데 서버가 기동 시점(또는 그 이전) 캐시를 계속 내려준다.

## 확인 방법

브라우저에서 실제로 받는 모듈 소스를 열어(devtools → Sources, 또는 `curl localhost:PORT/@fs/...경로`) 디스크 파일 내용과 비교한다. 서버 기동 시각이 파일 수정 시각보다 이전이면 이 문제를 강하게 의심한다.

## 해결

```
rm -rf apps/*/node_modules/.vite && bun run dev --force
```

재시작 후 브라우저에서도 강력 새로고침(Ctrl+Shift+R)까지 해야 한다. dev 서버를 껐다 켜는 것만으로는 부족하고 `.vite` 캐시 삭제 + `--force`가 필요했다.

## 출처
- [[프로젝트/Commonly-FE/프로젝트-현황]]
- Claude Code 세션 자동 캡처 (/data/project/Commonly-fe)
