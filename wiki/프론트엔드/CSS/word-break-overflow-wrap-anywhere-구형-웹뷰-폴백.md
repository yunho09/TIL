---
tags: [css, word-break, overflow-wrap, 웹뷰, 폴백]
updated: 2026-10-28
---

# 영어 단어가 줄 끝에서 잘릴 때: word-break vs overflow-wrap과 `anywhere` 폴백

`word-break: break-all`은 모든 글자 사이 줄바꿈을 허용해 영어 단어가 중간에서 잘린다. `word-break: normal` + `overflow-wrap`으로 바꾸면 긴 URL만 끊고 단어는 지킨다. 다만 `overflow-wrap: anywhere`는 Chrome 80/Safari 15.4부터라 구형 웹뷰에는 배열 폴백이 필요하다(당근 seed-design Dialog 사례 분석).

## 개념
- **`word-break`**: 어디서 끊어도 되는지. `break-all`은 영어 단어 중간도 끊음. 한글은 `normal`에서도 음절 사이에서 끊기므로 `normal`로 바꿔도 한국어 줄바꿈은 그대로. `keep-all`은 한국어를 어절 단위로 바꾸는 디자인 결정이라 버그 수정 범위에서 제외.
- **`overflow-wrap`**: 한 단어가 한 줄보다 길 때만 그 안을 끊음 → 긴 URL이 컨테이너 밖으로 넘치는 것을 막는 역할은 유지.
- **`anywhere` vs `break-word`**: `anywhere`는 끊는 자리를 최소 너비(min-content) 계산에도 반영해 flex 행 안의 URL도 줄어든다. `break-word`는 반영하지 않아 그 경우 넘친다.

## 구형 웹뷰 폴백
- 지원 하한이 Chrome 77/Safari 14라면 `anywhere` 하나만 쓰면 그 사이 버전에서 선언이 무시돼 긴 URL 넘침 회귀가 생긴다.
- CSS-in-JS 레시피(qvism)에서는 배열 값 `overflowWrap: ["break-word", "anywhere"]`로 선언 두 번을 순서대로 내보낸다(같은 패턴이 `maxHeight: [vh, dvh]`에 이미 있음). 파서가 생성한 CSS는 커밋돼 있어 `bun qvism:generate`로 재생성.
- 검증은 한국어/영어/긴 URL/한영 혼합/flex 행 안 URL 다섯 케이스를 수정 전후 스크린샷으로 비교.
- 참고: 같은 `word-break: keep-all` 사용은 [[CSS-fieldset-legend-flex-패딩-무시]].

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
