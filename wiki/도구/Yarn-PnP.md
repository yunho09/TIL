---
tags: [yarn, node, 패키지매니저]
updated: 2026-09-23
---

# Yarn PnP (Plug'n'Play)

Yarn Berry(v2+)가 기본으로 도입한 **의존성 로딩 방식**. 기존 `node_modules` 방식과 대조되는 개념이고, Yarn Berry를 쓴다고 해서 자동으로 PnP인 건 아니다([[Yarn-Berry-silent-플래그-무효-스크립트-거짓FAIL]] 참고 — 이 저장소는 Berry 버전만 쓰고 링커는 옛날 방식).

## node_modules 방식 vs PnP

| | node_modules (기존) | PnP |
|---|---|---|
| 설치 결과물 | `node_modules/` 폴더에 패키지 실제 복사 | `node_modules` 없이 `.pnp.cjs` 파일 + `.yarn/cache`의 zip들 |
| require 해석 | Node.js가 디스크에서 폴더 탐색 (느림, "phantom dependency" 가능) | `.pnp.cjs`가 패키지 위치를 map으로 직접 지정 (빠름, 엄격) |
| 설치 속도 | 상대적으로 느림 | 훨씬 빠름 (심볼릭 링크/복사 불필요) |
| "유령 의존성" 문제 | package.json에 안 적었어도 우연히 require 되면 동작해버림 | package.json에 명시 안 된 패키지는 즉시 에러 — 더 엄격 |
| 에디터(TS/ESLint) | 그냥 동작 | VSCode 등에 별도 SDK 설정 필요 (`yarn dlx @yarnpkg/sdks vscode`) |
| CI 캐시 | `node_modules` 캐시 | `.yarn/cache`(zip) 캐시 — 보통 더 작고 빠름 |
| 생태계 호환성 | 모든 도구가 그냥 됨 | 일부 오래된 툴/스크립트가 `node_modules` 존재를 가정해서 깨질 수 있음 |

## 링커 설정 확인법

`.yarnrc.yml`의 `nodeLinker` 값으로 판별한다.
- `nodeLinker: pnp` (또는 미설정 시 Berry 기본값) → PnP.
- `nodeLinker: node-modules` → PnP 아님, Berry 버전만 쓰고 기존 방식 그대로.

`packageManager: yarn@4.x.x`만 보고 PnP라고 단정하면 안 된다 — 버전과 링커는 별개 설정.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
