---
tags: [npm, package-json, exports, esm, umd]
updated: 2026-10-28
---

# package.json `"type":"module"` + UMD 진입점 → import/require가 빈 객체

`"type":"module"`인 패키지의 `main`이 UMD `.js`를 가리키면 Node가 UMD를 ESM으로 읽어 export가 하나도 없는 빈 객체가 된다. 에러도 안 뜬다(yorkie-js-sdk의 `@yorkie-js/schema` #1490 사례). `exports` 조건 맵과 `.mjs` 확장자로 해결한다.

## 원인
- Node는 확장자와 가장 가까운 `package.json`의 `"type"`으로 CJS/ESM을 정한다. `"type":"module"` 안의 `.js`는 무조건 ESM.
- UMD는 "CJS면 `module.exports`, 아니면 전역 변수에 등록"하는 파일이다. ESM으로 읽히면 `module`이 없어 전역에만 넣고 export는 없다. → `import`도 `require`도 `[]`/빈 객체. esbuild로 번들해도 `has no exports` 경고 후 실행 시 `undefined` TypeError.

## 해결 (같은 레포 sdk 패키지 #1431 방식)
- `"type":"module"` 제거 → UMD `.js`가 다시 CJS.
- ES 번들은 `.es.mjs`(확장자로 ESM 표시), 타입은 `.d.mts` 쌍.
- `exports` 조건 맵: `node`+`import` → CJS를 재export하는 얇은 `.node.mjs` 래퍼(코드 두 벌 로드 방지, import/require가 같은 인스턴스), `node`+`require` → UMD, `import` → `.es.mjs`, `require`/`default` → UMD, `./package.json`.
- `main`은 UMD 그대로 두어 unpkg/jsDelivr CDN 사용자 영향 없음.
- 검증: `pnpm pack` tarball에서 이슈 재현 명령(고치기 전 `[]` → 후 `[ 'buildRuleset', 'validate' ]`), Node 20/22/24, ATTW(are-the-types-wrong), 수정한 검증 스크립트를 수정 전 빌드에 돌려 실패하는지(반대 확인).

## Breaking 안내
- `exports`가 생기면 `.`/`./package.json` 외 하위 경로(deep import) 차단, 이전에 직접 import하던 `dist/*.es.js` 이름이 `.es.mjs`로 바뀜. 커밋 본문에 `Breaking:` 줄, PR 본문에 호환성 노트를 넣는다(선례 #1431). 다운스트림 사용처(같은 팀 dashboard)도 언급.
- 리뷰 지적: 검증 스크립트가 결과 배열 순서에 의존하면 패키징과 무관하게 깨지므로 `.some(...)`으로 순서 독립으로.

## 같이 겪은 UMD 전역 이름 불일치 (#1489)
- `<script>` 태그용 prosemirror UMD가 `global.YorkieSdk`를 찾는데 SDK UMD는 `window["yorkie-js-sdk"]`로 등록 → 로드 즉시 에러. 해결은 `(global.YorkieSdk || global["yorkie-js-sdk"])`로 둘 다 지원. Rollup `output.globals`는 이름 하나만 받으므로 빌드 결과 문자열을 고치는 `renderChunk` 플러그인을 쓰고, 치환 지점이 정확히 1개가 아니면 빌드를 실패시키는 안전장치를 건다. 메인테이너가 "기존에 `YorkieSdk`를 직접 설정한 페이지가 계속 동작해야 한다"고 요구했으므로 호환 경로는 필수.

## 관련
- [[오픈소스-기여-후보-선별-기준과-AI정책-함정]] (yorkie는 CLA 필요, 두 브랜치를 함께 머지하면 `docs/tasks/README.md`만 충돌 → `bash scripts/tasks-index.sh`로 재생성)

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
