---
tags: [javascript, typescript, nullish-coalescing]
updated: 2026-09-10
---

`??`(nullish coalescing)는 왼쪽 값이 `null`/`undefined`일 때만 오른쪽으로 폴백한다. `""`(빈 문자열)·`0`·`false`처럼 "값은 있지만 falsy"한 경우엔 폴백이 발동하지 않는다. `||`(logical OR)는 falsy 전체에 반응해 폴백한다는 점에서 다르다.

## 함정이 드러나는 패턴

문자열 파싱 결과가 **빈 문자열로 정상 반환**되는 경우(에러가 아님) `??`로 fallback을 짜면 빈 문자열이 그대로 새어 나간다.

```js
function getFileName(url) {
  return new URL(url).pathname.split("/").pop() ?? url;
}

getFileName("https://example.com");
// new URL(...).pathname === ""  (경로 세그먼트가 없음)
// "".split("/").pop()           === ""
// "" ?? url                     === ""  ← 폴백 안 됨, 빈 문자열 그대로 반환
```

`pathname`이 없는 URL(호스트만 있고 경로가 없는 값 등)에서 `split("/").pop()`은 예외 없이 조용히 `""`를 반환한다. `??`는 `""`을 "값이 있다"고 보기 때문에 원래 의도한 "파싱 실패 시 원본 URL로 대체" 동작이 빠진다.

## 판별 기준

- 폴백 대상이 "값이 없을 수도 있는" 경우(`null`/`undefined`만 가능)면 `??`가 맞다.
- 폴백 대상이 "빈 문자열/0처럼 falsy하지만 유효한 값이 나올 수 있는" 경우면 `??` 대신 `||`(또는 명시적으로 빈 문자열을 체크하는 삼항)를 써야 한다.

## 실사례 — 같은 로직인데 한쪽만 `??`로 남아 있던 divergence

JOBIS-FE-V2에서 어드민 공지 상세(`apps/admin/.../notice/detail/page.tsx`)는 `getFileName`을 `fileName || url`로 구현해 이미 이 문제를 피해가고 있었는데, 학생 앱 공지 상세(`apps/student/.../notice-detail/page.tsx`)만 `?? url`로 남아 있어서 더미 데이터(`url: "https://~~~"`, 경로 세그먼트 없음)를 가진 공지에서 첨부파일 링크 텍스트가 빈 문자열로 사라지고 다운로드 아이콘만 남는 버그가 있었다. 학생 쪽을 `|| url`로 맞춰서 해결. 같은 헬퍼 로직을 여러 앱에 복붙할 때 `??`/`||` 선택이 갈라지지 않았는지 대조할 필요가 있다.

관련: [[JOBIS-FE-V2/프로젝트-현황]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
