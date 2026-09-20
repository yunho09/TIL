---
tags: [css, react, 애니메이션, 접근성, 모션]
updated: 2026-09-20
---

`if (!open) return null` / `open && <Menu>` 구조는 닫는 순간 DOM에서 지워버려서 **나가는 애니메이션이 원리상 불가능**하다. 퇴장 전환을 넣으려면 "닫힘 상태"와 "마운트 상태"를 분리해 애니메이션 시간만큼 DOM을 남겼다가 지워야 한다. 이 페이지는 그 패턴(`useExitAnimation`)과 사이드바·아코디언·드롭다운에 적용하며 부딪힌 함정을 정리한다.

## 원인: 지워진 요소는 사라지는 모습을 보일 수 없다

- 등장(`dropIn` 등)은 마운트되는 순간 재생되니 조건부 렌더링만으로도 된다. 퇴장은 언마운트가 먼저 일어나 재생할 요소가 없다.
- 그래서 "열릴 때만 부드럽고 닫을 땐 뚝 사라지는" 증상이 사이드바·셀렉트·케밥 메뉴에서 똑같이 났다(같은 원인이라 리뷰에서도 같은 지적이 반복됨).

## 해결: `useExitAnimation(open, ms)` 훅

- `open`이 false가 되면 `ms`만큼 마운트를 유지했다가 내린다. 그 사이 다시 열리면 **타이머를 취소**한다.
- 마운트된 동안 `$closing={!open}` 같은 플래그로 퇴장 keyframe(`dropOut`)을 재생한다.
- 사라지는 동안 `pointer-events: none` — 안 걸면 "삭제" 클릭으로 메뉴가 닫히는 중에 항목이 한 번 더 눌릴 수 있다.
- 사이드바·Toast·드롭다운 2종이 각자 갖고 있던 같은 타이머 로직을 이 훅 하나로 합쳤다.
- `prefers-reduced-motion: reduce`이면 대기 없이(0ms) 즉시 내린다.
- 트리거가 목록과 붙어 보이는 셀렉트는 목록이 붙어 있는 동안 트리거도 "열린 모양"(테두리·라운드)을 유지해야 테두리가 먼저 떨어지지 않는다.

## `animationend` 대신 타이머를 쓴 이유

코드리뷰(CodeRabbit)는 "`animationend`에서 `open`이 여전히 false일 때만 제거"를 제안했다. 채택하지 않은 이유:

- 탭이 백그라운드이거나 애니메이션이 중간에 취소되면 `animationend`가 **오지 않아** 메뉴가 DOM에 영영 남을 수 있다. 타이머는 어떤 경우든 끝난다.
- 저장소의 Toast·사이드바가 이미 타이머 방식이라 세 곳이 같은 훅을 쓸 수 있다.
- "닫히는 중 다시 열면 제거하지 않는다"는 요구는 타이머 취소로 동일하게 충족된다.

## 접히는 아코디언: `grid 0fr → 1fr` + `inert`

- 높이 전환은 `grid-template-rows: 0fr → 1fr`. 접힌 그룹은 DOM에 남으므로 높이 0 + `visibility: hidden`으로 초점이 못 들어가게 한다.
- **접힌 상태에 `padding`이 남으면 `border-box` 때문에 높이가 8px 남는다** → 접힘 상태 padding 0.
- 리뷰(Copilot) 지적: `visibility`를 transition 목록에 넣으면 discrete 속성이라 hidden 전환이 끝까지 미뤄져, 접히는 180ms 동안 하위 링크가 포커스·클릭 가능하다.
  - 제안대로 `visibility` 전환을 빼면 항목이 즉시 사라지고 빈 상자만 줄어들어 애니메이션이 깨진다.
  - 채택한 해결: 전환은 유지하고 **접기 시작과 동시에 `inert`**를 걸어 초점·클릭·접근성 트리에서 즉시 제외. 눈에는 접히는 게 보이지만 이미 "탑승 불가" 상태.
- 사이드바 패널도 닫는 애니메이션이 끝날 때까지 `inert`로 초점·접근성 트리에서 뺀다.

## 등장 애니메이션과 `transform` 충돌

- 위치 보정에 `transform: translateY(-36px)`을 쓰고 있던 모달은 `popIn`(transform 애니메이션)이 그 값을 덮어써 충돌했다 → 같은 위치를 `margin-bottom: 72px`로 대체.
- 일반화: 등장 연출에 transform을 쓸 요소는 정적 위치 보정에 transform을 쓰지 말고 margin·top 등으로 옮긴다.

## 전환 값 한 곳에 모으기 · reduced-motion

- 길이·이징·공용 keyframes를 `shared/ui/motion.ts`에 모은다(color 120 / reveal 180 / overlay 240ms). 이 저장소는 style-policy상 `tokens.ts`가 색·폰트 전용이라 별도 모듈로 뒀다. 느리다 싶으면 숫자 한 곳만 고치면 전체가 같이 바뀐다.
- `global.css`에 `@media (prefers-reduced-motion: reduce)`로 전환·애니메이션을 즉시 끝내고, JS 언마운트 타이머도 0으로 떨어뜨린다(멀미 등으로 OS에서 모션을 끈 사용자에게 강요하지 않음).
- **"배포했는데 애니메이션이 안 보인다"**: 배포 실패·캐시 외에, 본인 OS의 "동작 줄이기(우분투: 설정 > 접근성)"가 켜져 있으면 reduced-motion 규칙 때문에 정상 배포에서도 예전처럼 뚝 바뀐다(Claude 진단, 사용자 확인 결과는 세션에 없음).

## 곁가지: `yarn format`은 전체로 돌리지 않는다

이 저장소는 prettier 기준으로 정리돼 있지 않아 `yarn format`이 327개 파일을 재포맷했다. 자기 변경 파일만 남기고 나머지는 되돌려야 한다(→ [[ToyVillage-Admin-FE/프로젝트-현황]] 이슈 #108 절의 넓은 glob prettier 함정과 같은 계열).

## 관련
- [[backdrop-filter-blur-애니메이션-배경-스크롤-끊김]] — 애니메이션 성능 쪽 함정
- [[테이블-오버플로우-hidden-드롭다운-메뉴-잘림]] — 같은 케밥 메뉴의 클리핑 문제
- [[ToyVillage-Admin-FE/프로젝트-현황]] — 이슈 #176·PR #179 적용 기록

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-3) — 2026-09-20 (이슈 #176 가벼운 전환 애니메이션, PR #179, Copilot·CodeRabbit 리뷰 반영)
