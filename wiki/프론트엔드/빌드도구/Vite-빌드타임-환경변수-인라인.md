---
tags: [vite, frontend, 환경변수, 빌드]
updated: 2026-09-08
---

# Vite 빌드타임 환경변수 인라인

Vite는 `import.meta.env.VITE_*` 값을 런타임에 읽지 않고 **빌드 시점에 번들 코드에 문자열로 치환**한다. `VITE_` 접두사가 붙은 값은 그래서 사실상 공개값이고, 값을 바꾸려면 재빌드·재배포가 필요하다.

## 메커니즘
- `vite.config`(또는 별도 `vite.env.ts`) 안 헬퍼가 `define: { 'import.meta.env.VITE_XXX': JSON.stringify(value) }` 형태로 esbuild/rollup `define` 옵션에 값을 박아 넣는다.
- 빌드된 산출물을 열어보면 값을 참조하던 함수가 상수로 컴파일돼 있다. 예: 키가 비어 있으면 `function qs(){return``}`.
- `.env.local`(로컬)이나 CI/배포 환경변수(예: Portainer)에 값을 넣어도 **다시 빌드**해야 브라우저에 반영된다. 서버를 재시작해도 이미 빌드된 정적 파일은 바뀌지 않는다.

## 시사점
- `VITE_*` 값은 브라우저 devtools에서 누구나 꺼내볼 수 있다 — **비밀값(secret)을 넣으면 안 된다.** 외부 API가 승인키를 URL/도메인 단위로 제한하는 방식([[도로명주소-검색-API-신청]] 참고)과 조합해야 그나마 안전하게 쓸 수 있다.
- 진짜 비밀을 숨기려면 프론트가 직접 호출하지 말고 백엔드에 프록시 엔드포인트를 두고 키를 서버에만 보관해야 한다.
- 배포된 번들에서 값이 실제로 들어갔는지 확인하려면 `dist/` 산출물의 컴파일된 함수를 직접 열어보는 게 가장 확실하다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/Commonly-fe) — `VITE_JUSO_CONFM_KEY`(도로명주소 검색 API 승인키) 인라인 여부를 번들 코드로 직접 검증한 사례
