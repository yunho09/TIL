---
tags: [vitest, jsdom, nodejs, localstorage, testing]
updated: 2026-10-01
---

# Node 내장 Web Storage가 vitest jsdom의 `localStorage`를 가린다

최신 Node(22+)가 실험적 Web Storage로 `globalThis.localStorage`를 먼저 차지하는데, `--localstorage-file` 없이는 그 값이 `undefined`(또는 빈 객체, 접근 시 throw)다. vitest의 jsdom 환경은 **이미 global에 있는 키는 jsdom window에서 복사하지 않아** jsdom의 진짜 Storage가 주입되지 못해, `localStorage.getItem`을 쓰는 코드가 수집 단계에서 줄줄이 터진다.

## 증상과 진단
- `yarn test:unit` → 46개 파일 중 37개가 수집 단계 실패, `localStorage.getItem` 이 undefined(`useTheme.ts` 등 모듈 로드 시 접근).
- 함께 뜨는 `ExperimentalWarning: localStorage is not available because --localstorage-file was not provided`가 **Node 자체 메시지**라는 게 단서. jsdom을 단독 실행하면 `setItem/getItem` 왕복이 정상 → jsdom 문제가 아니라 환경 충돌.
- 관측된 값은 Node 버전마다 다르다: 26.8.2에서는 `undefined`, 25는 빈 객체, 26 문서 기준 `--localstorage-file` 없이 접근하면 `DOMException` throw. 프로젝트 `.nvmrc`(23.11.0)에서의 재현 여부는 해당 런타임이 없어 미확인.
- CI에서 테스트를 안 돌리는 레포라면 로컬에서 돌리는 사람만 막히고, 코드 변경과 무관하게 기존부터 깨져 있다는 점으로 원인을 분리할 수 있다.

## 해결: 셋업 파일 폴리필
- `vitest.unit.setup.ts`에서 **사용 가능 여부를 try/catch로 판정**하고, 쓸 수 없을 때만 Map 기반 Storage를 `Object.defineProperty(globalThis, "localStorage", ...)`로 넣는다.
```ts
const hasUsableLocalStorage = () => {
  try { return typeof globalThis.localStorage?.getItem === "function"; }
  catch { return false; }
};
```
- 옵셔널 체이닝(`?.`)만으로는 **접근 시 throw하는 getter**를 못 막는다 → try/catch 필수. 세 경우(undefined/빈 객체/throw) 모두 커버된다.
- 적용 후 46파일 272테스트 전부 통과.
- 대안 `execArgv: ['--no-webstorage']`(vitest#10867)는 vitest 3.1.0에서 옵션 위치가 다르고 Node 버전 가드가 필요해 폴리필을 유지하고 vitest 4 업그레이드 때 재검토하기로 했다.

## 같은 세션의 오탐 판별 메모 (코드 리뷰 봇 지적 검증)
- "vitest `playwright` provider가 없다" → `@vitest/browser@3.1.0`에 내장(`dist/providers.js`에 `PlaywrightBrowserProvider`). 분리 패키지는 Vitest 4부터.
- "tsconfig alias가 안 먹는다" → composite project reference라 TS가 `dist/*.d.ts`로 리다이렉트. dist·tsbuildinfo를 모두 지운 클린 상태에서 `tsc --build` 통과로 반증.
- "`.env.development`를 `doctor.ts`가 못 읽는다" → `loadEnv`가 `.env.[mode]`를 읽는 것을 실제로 재현해 확인.
- 봇 지적은 근거를 직접 재현해 확인한 뒤 코드 변경 없이 근거만 남기고 resolve하는 판단이 필요하다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — [[JOBIS-FE-V2/프로젝트-현황]]
