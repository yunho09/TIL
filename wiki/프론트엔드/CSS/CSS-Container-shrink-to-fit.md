---
tags: [css, flexbox, layout]
updated: 2026-09-10
---

# CSS shrink-to-fit과 max-width의 한계

`max-width`(또는 그걸 감싼 `Container $maxWidth` 같은 컴포넌트 prop)는 폭의 **상한선**일 뿐, 폭을 **확보**해주지는 않는다. 부모 요소가 이미 콘텐츠 크기로 줄어들어 있으면(shrink-to-fit), 그 안의 자식이 `max-width: 840px`을 걸어도 실제로는 부모가 줄어든 만큼(예: 587px)만 차지하게 된다.

## 발생 조건

`<main>` 같은 요소에 `margin: 0 auto`만 걸려 있고 명시적 `width`가 없는 상태로, 그 `main`이 **flex item**으로 배치돼 있으면 `flex-basis: auto` 기본값 때문에 콘텐츠 크기만큼만 차지하는 shrink-to-fit 동작을 하게 된다. 이 상태의 `main` 안에서 `max-width`로 폭을 지정하는 방식은 "제한"만 될 뿐 실제 폭을 만들어내지 못한다.

## 해결

폭을 **지정**해야 하는 위치에서는 `max-width` 대신 **명시적 `width`**를 준다. `max-width`는 어디까지나 "이 이상은 못 커진다"는 상한 제약이지, 최소 확보를 보장하는 속성이 아니라는 걸 구분해서 써야 한다.

우연히 이 문제를 안 겪는 경우도 있는데, 자식 컴포넌트(예: 테이블류)가 자체적으로 폭을 갖고 있으면 부모의 shrink-to-fit 여부와 무관하게 결과가 맞아떨어지기 때문 — 즉 다른 페이지에서 문제없이 동작했다고 해서 이 레이아웃 구조 자체가 안전하다고 볼 수는 없다.

## 관련: 전체 배경색이 필요한데 컨테이너가 shrink-to-fit일 때

페이지 전체에 회색 배경 같은 full-bleed 배경을 깔고 싶은데, 감싸는 컨테이너(`main` 등)가 콘텐츠 크기로만 줄어들어 있어 CSS만으로는 뷰포트 전체를 못 채우는 경우, 페이지 컴포넌트가 **마운트된 동안만 `document.body`의 배경색을 바꾸고 언마운트 시 원복**하는 방식으로 우회할 수 있다. SPA에서는 라우팅으로 다른 페이지로 이동했을 때 정상적으로 원래 배경색이 복원되는지 반드시 확인해야 한다.

관련: [[JOBIS-FE-V2/프로젝트-현황]] — 실제로 이 문제를 겪은 사례. 2026-09-09 세션에서는 학생 앱 공용 `router.tsx`의 `<main style={{flex:1}}>`이 같은 원인으로 줄어들어 있는 걸 발견해, 페이지별 우회 대신 `width: "100%"`를 라우터 레벨에 추가해 앱 전체를 근본 수정했다.

## 후속 함정 — shrink-to-fit을 고치면 `margin: auto` 중앙 정렬이 깨진다

위 수정(`main`에 `width: 100%` 추가) 자체가 새 회귀를 만들 수 있다. `GlobalStyles`에 `main { margin: 0 auto; }`가 있고 `main`이 `flex-direction: column` 컨테이너의 flex item이면:

- **수정 전**: `main`에 명시적 `width`가 없어 콘텐츠 크기로 줄어듦(shrink-to-fit) → **flexbox에서 교차축(cross-axis) `auto` 마진은 `align-items: stretch`를 무력화**하는 성질 때문에, `margin: 0 auto`가 줄어든 `main`을 가운데로 밀어줌.
- **`width: 100%`를 추가한 후**: `main`이 뷰포트 전체를 차지 → 교차축 auto 마진이 밀어줄 여백이 없어져 사실상 0 → `main` 안의 `Container`(`max-width` prop만 있고 자기 정렬 마진이 없는 컴포넌트)가 왼쪽에 그대로 붙어버림.

즉 "부모의 shrink-to-fit을 없앤다"와 "자식이 스스로 가운데 정렬된다"는 서로 다른 문제라서, 하나를 고치면 다른 하나가 새로 드러날 수 있다.

**해결**: 정렬 책임을 부모(`main`의 `margin: auto`)가 아니라 **폭을 제한하는 자식 컴포넌트 자신**(`Container`)에 둔다 — `max-width`를 쓰는 컴포넌트라면 항상 `margin-inline: auto`를 같이 넣어서 자기 자신을 가운데 정렬하게 만들면, 부모(`main`)가 full-width든 shrink-to-fit이든 영향을 안 받는다.

```css
/* Container.tsx */
width: 100%;
margin-inline: auto;   /* 추가 — max-width가 있을 때만 의미 있고, 없으면 no-op */
${({ $maxWidth }) => $maxWidth && `max-width: ${parseValue($maxWidth)};`}
```

2026-09-10 세션에서 실제로 겪은 순서: `main`에 `width: 100%`를 추가(2026-09-09) → 전 페이지 콘텐츠가 좌측으로 쏠리는 회귀 발생 → 원인이 `main`의 `margin: 0 auto`가 교차축에서 무력화된 것임을 확인 → `main`은 그대로 두고 `Container`에 `margin-inline: auto`를 추가해 해결. `main`에 `width: 100%`가 없던 어드민 앱은 원래도 가운데였고 이 변경은 no-op이라 영향 없음.

관련: [[JOBIS-FE-V2/프로젝트-현황]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
