---
tags: [eslint, flat-config, esm, commonjs]
updated: 2026-10-28
---

# ESLint flat config와 레거시 eslintrc의 ESM/CJS 차이

레거시 eslintrc는 JS 설정 파일을 `require`로 읽어서 CommonJS여야 하고, flat config(`eslint.config.js`)는 `import()`로 읽어서 ESM을 그대로 쓴다. 문서 예시를 고칠 때 이 차이를 모르면 동작하지 않는 예시를 만든다(eslint-plugin-import-x README 사례).

## 규칙
- **flat config(ESLint 9 기본):** `eslint.config.js`를 `import()`로 로드 → `import`/`export default`. 내보내는 값은 객체 하나가 아니라 **배열** (`export default [ { settings: ... } ]`).
- **레거시 eslintrc(ESLint 10에서 제거):** JS 설정을 `require`(importFresh)로 읽음 → `.eslintrc.js`/`.cjs`는 CommonJS(`module.exports`)여야 함.
- **`.eslintrc.mjs`는 아예 읽지 않는다.** `@eslint/eslintrc`의 `configFilenames`에는 `.eslintrc.js/.cjs/.yaml/.yml/.json/.eslintrc/package.json`만 있다. 재현: `ESLINT_USE_FLAT_CONFIG=false`에서 `.eslintrc.mjs`만 두면 "ESLint couldn't find a configuration file". 그래서 그런 예시를 따라 하면 설정이 조용히 무시된다.
- `import-x/resolver` 같은 settings는 두 방식에서 같은 코드가 처리하므로 flat config로 옮겨도 의미가 같다.

## 문서 변환 시 판단 기준
- 설정 예시는 flat으로 바꾸되, 레거시 사용자를 위한 `.eslintrc.cjs`(require)와 YAML/JSON 예시는 남긴다.
- **규칙 문서의 `require`/`module.exports`는 "이 규칙이 잡는 코드" 예시**(no-commonjs 등)이므로 ESM으로 바꾸면 뜻이 달라진다. CHANGELOG의 과거 릴리스 노트도 배포된 기록이라 손대지 않고 메인테이너에게 묻는다.
- 변환 예시에서 `path`를 import 없이 쓰던 곳엔 `import path from 'node:path'` 추가.
- 마크다운 코드 블록까지 린트하는 레포(eslint-plugin-mdx)도 문법이 깨진 블록은 조용히 넘기므로, 블록을 뽑아 `node --check *.mjs`로 따로 확인하고 임시 프로젝트에서 실제로 리졸버가 동작하는지 본다.

## 관련
- [[오픈소스-기여-후보-선별-기준과-AI정책-함정]] — 이 수정이 나온 기여 작업(import-x #216, `Refs`로 연결하고 CHANGELOG는 메인테이너에게 질문).

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
