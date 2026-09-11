---
tags: [testing, playwright, javascript, module-system]
updated: 2026-09-11
---

# 배럴 Import 함정

폴더의 `index.ts`가 내부 모듈을 모아 재수출하는 것을 **배럴(barrel)**이라 부른다. 앱 코드에서는 편리하지만, **Node가 직접 모듈을 로드하는 테스트 파일**에서는 하나만 가져와도 배럴이 재수출하는 나머지 전부가 실행되어 예상치 못한 크래시를 일으킬 수 있다.

## 왜 문제가 되나

`export ... from '...'` 문법은 재수출 대상 모듈을 **실제로 실행**시킨다. 트리셰이킹이 안 되는 구간이다.

```ts
// src/entities/close-schedule/index.ts  (배럴)
export { closeScheduleStorageKey } from './model/mock'        // 필요한 것
export { createCloseSchedule } from './api/closeScheduleApi'  // 안 써도 실행됨
  // → import { api } from '@/shared/api/axios' → axios → form-data → ...
```

앱 실행 시에는 Vite가 번들링하면서 안 쓰는 걸 트리셰이킹하므로 문제없다. 하지만 Playwright가 테스트 파일을 **Node**에서 로드할 때는 이 가공을 거치지 않아, 배럴이 딸려온 axios의 CJS 의존 체인(`form-data` → `es-set-tostringtag` → `get-intrinsic`)이 링크 단계에서 깨질 수 있다.

```
$ yarn playwright test tests/e2e/close-schedule-edit.spec.ts --list
Error: request for './index.js' is from a module not been linked
    at node_modules/get-intrinsic/index.js:154:24
Total: 0 tests in 0 files
```

## 가장 나쁜 점 — 조용히 사라진다

에러가 나면 "그 테스트가 실패"할 것 같지만 실제로는 **파일 자체가 수집되지 않는다.** `Total: 0 tests in 0 files`로 끝나고, 실패 집계에도 잡히지 않는다. 그래서 이 문제로 파일 몇 개가 통째로 안 돌고 있어도 아무도 눈치채지 못할 수 있다.

## 진단 방법

의심되는 import를 단독으로 실험해 좁힌다.

```
테스트 파일에서 import axios from 'axios'        → 크래시 (재현)
테스트 파일에서 배럴 하위 모듈만 직접 import      → 정상
```

## 해결

테스트 파일에서는 배럴을 우회하고 필요한 하위 모듈을 직접 가리킨다.

```diff
- import { closeScheduleStorageKey } from '@/entities/close-schedule'
+ import { closeScheduleStorageKey } from '@/entities/close-schedule/model/mock'
```

가져오는 값은 동일하다(배럴도 결국 같은 파일에서 꺼내 재수출하던 것). 재발 방지로 `tests/**`에서 엔티티 배럴 직접 import를 금지하는 ESLint `no-restricted-imports` 규칙을 검토할 수 있다.

앱 코드(`src/**`)는 배럴을 그대로 쓰는 게 맞다 — 문제는 Node 로더를 직접 타는 테스트 파일에서만 발생한다.

## 집계 스크립트를 짤 때의 부수 함정

이 문제를 진단하던 중, 스펙 전체를 자동으로 훑는 스크립트가 `"Total:"` 문자열 포함 여부로 성공을 판정했는데, **크래시로 수집 자체가 실패한 경우도 `Total: 0 tests in 0 files`를 출력**해 같은 문자열이 찍히는 바람에 깨진 파일 3개가 전부 "정상"으로 오판됐다. 대량 스펙을 자동으로 훑을 때는 `passed`/`failed` 숫자 자체를 파싱하거나 exit code를 봐야지, 특정 로그 문자열의 존재 여부만으로 판정하면 안 된다. `tail -1`로 마지막 줄만 보는 방식도 같은 이유로 위험하다 — `failed` 줄이 `passed` 줄보다 먼저 출력되면 실패를 놓친다.

## 다른 메커니즘: 순환 참조로 top-level 코드가 `undefined`를 받음

Node 로더 문제와는 별개로, 배럴을 거치는 **순환 참조** 자체가 문제가 되는 경우도 있다. 여러 모듈이 배럴 경유로 서로를 참조하는 순환 고리 안에 있으면, 그중 한 모듈이 **모듈 최상위(top-level)에서** 다른 모듈의 값을 사용할 때 초기화가 덜 끝난 `undefined`를 받을 수 있다.

실사례: `renderWithTheme → themes → GlobalStyles → hooks → useToast → Toast → @/components 배럴 → compound → Modal` 순서로 이어지는 순환에서, 다른 컴포넌트들은 함수 본문 안에서만 값을 쓰는데 **`Modal`만 모듈 최상위에서 `styled(Text)`를 평가**해서 그 시점에 `Text`가 아직 `undefined`라 크래시가 났다(`Cannot read properties of undefined`류). 증상만 보면 원인이 `Modal`이 아니라 그 시점에 렌더되던 다른 컴포넌트처럼 보이기 쉬워서, 추측 대신 **실제 import 그래프를 순서대로 따라가야** 진짜 고리를 찾을 수 있었다.

해결은 순환에 걸리는 배럴 경유를 끊는 것 — 문제가 된 모듈이 배럴(`@/components`) 대신 실제 모듈 경로를 직접 가리키게 바꿨다. 최상위 배럴을 거치는 한 다른 조합으로 순환이 재발할 수 있어, **모듈 최상위에서 다른 모듈의 값을 즉시 평가하는 코드(`styled(X)` 등)가 있으면 배럴 대신 직접 경로를 쓰는 편이 안전**하다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] (`close-schedule`/`operating-hours` 스펙 3개가 이 문제로 조용히 안 돌고 있던 사례)
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — [[프로젝트/JOBIS-FE-V2/프로젝트-현황]] (`Modal` 순환 참조로 `styled(Text)`가 `undefined`를 받던 사례)
