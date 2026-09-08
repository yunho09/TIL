---
tags: [css, layout, forms]
updated: 2026-09-08
---

# `<legend>`가 flex fieldset에서 카드 padding을 무시하는 문제

`fieldset`에 `display: flex`를 주면 브라우저가 `<legend>`를 border-box 최상단에 특수 배치한다. 그 결과 `fieldset`의 `padding`을 무시하고 라벨이 카드 테두리에 딱 붙어버리고, 대신 카드 아래쪽에 그만큼의 빈 공간이 생긴다.

## 증상

- 디자인 스펙은 라벨이 카드 안쪽 padding 위치(예: `@40,40`)에 있어야 하는데, 실측하면 `legendInsetTop=0`으로 카드 최상단에 붙어 있다.
- 원인은 `fieldset`을 flex로 만들어 내부를 정렬하려던 것인데, `legend`도 flex item 취급을 받으면서 일반 흐름과 다른 위치로 렌더된다.

## 해결

`fieldset`을 `display: block`으로 되돌리고, `legend`에 `float: left; width: 100%`를 줘서 일반 문서 흐름에 편입시킨 뒤, 뒤따르는 콘텐츠에 `clear: both`를 줘서 float을 해제한다. `fieldset`/`legend` + radio 시맨틱은 그대로 유지되어 접근성 손실이 없다.

```css
fieldset { display: block; padding: 40px; }
legend { float: left; width: 100%; margin: 0 0 20px; }
.content { clear: both; }
```

## 관련: 같은 디버깅에서 함께 정리한 반응형 오버플로 기법

- **고정 px 그리드가 좁은 폭에서 카드를 넘칠 때**: `grid-template-columns`를 고정 px(`360px 320px 320px 1fr`) 대신 같은 비율의 `fr`(`360fr 320fr 320fr 320fr`)로 바꾸면 컨테이너 축소에 비례해서 같이 줄어든다. 줄바꿈되면 안 되는 라벨·값 셀에는 `white-space: nowrap`을 같이 준다.
- **한글 텍스트가 어절 중간에서 줄바꿈될 때**: `word-break: keep-all`을 주면 공백(어절) 단위로만 줄바꿈된다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
