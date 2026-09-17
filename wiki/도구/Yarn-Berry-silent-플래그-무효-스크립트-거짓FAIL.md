---
tags: [yarn, node, 트러블슈팅]
updated: 2026-09-17
---

# Yarn Berry에서 `-s`(silent) 플래그가 무효 — 검증 스크립트 거짓 FAIL

Yarn classic에 있던 `-s`(silent) 플래그가 **Yarn berry에서는 지원되지 않는다.** `yarn -s <script>`로 실행하면 스크립트 자체가 정상 실행되지 않고 FAIL로 끝나는데, 같은 스크립트를 `node scripts/xxx.mjs`로 직접 실행하면 문제없이 통과한다.

## 증상과 함정

검증/승인 스크립트(`yarn harness:api:validate`, `yarn harness:api:approve` 같은 것들)가 원인 불명으로 실패하면, 먼저 "코드가 잘못됐나" 또는 "환경이 깨졌나"를 의심하게 되지만 실제 원인이 **Yarn berry가 인식 못 하는 CLI 플래그** 하나인 경우가 있다. 스크립트 로직은 멀쩡한데 실행 경로 자체가 깨져서 FAIL이 나오므로 디버깅이 엉뚱한 방향으로 새기 쉽다.

## 대응

- Yarn berry 프로젝트에서 스크립트가 원인 불명으로 실패하면 `-s` 같은 classic-yarn 전용 플래그부터 의심한다.
- `node scripts/xxx.mjs`처럼 스크립트를 node로 직접 실행해 결과를 교차 검증하면 Yarn 래퍼 문제인지 스크립트 자체 문제인지 빠르게 구분된다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
