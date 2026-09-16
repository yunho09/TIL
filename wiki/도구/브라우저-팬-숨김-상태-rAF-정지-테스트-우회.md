---
tags: [claude-code, 브라우저, 테스트, rAF, 도구]
updated: 2026-09-16
---

# 브라우저 팬 숨김 상태 — rAF 정지와 테스트 우회

Claude Code의 내장 브라우저 팬(임베드 브라우저)이 다른 창에 가려지거나 백그라운드로 밀리면 `document.hidden`이 `true`가 된다. 스크린샷·시각 검수 도구로 웹앱을 검증할 때 이 상태 자체가 결과를 왜곡할 수 있다.

## 증상

- `requestAnimationFrame` 콜백이 전혀 실행되지 않는다 — CSS 애니메이션 시계도 특정 시점에 멈춰 보인다.
- 스크린샷이 타임아웃되거나, 스크롤 중 프레임이 찢긴 채로(부분 페인트) 캡처된다.
- 이건 페이지의 버그가 아니라 **팬(브라우저 탭) 자체의 렌더링/컴포지팅 정지**다. 레이아웃 계산 자체는 정상이다.

## 우회 검증법

스크린샷 대신, 콘솔에서 강제로 "보이는 상태"를 흉내 내고 실제 핸들러를 실행시켜 DOM 계측으로 검증한다.

```js
Object.defineProperty(document, 'hidden', { get: function(){ return false; }, configurable: true });
window.requestAnimationFrame = function(cb){ return setTimeout(function(){ cb(performance.now()); }, 0); };
document.dispatchEvent(new Event('visibilitychange'));
window.dispatchEvent(new Event('resize'));
```

이후 스크롤을 발생시키고 `getComputedStyle`/`getBoundingClientRect`/`transform`/캔버스 픽셀 점등 여부 등으로 애니메이션 파이프라인이 실제로 도는지 확인한다. 정지 화면(스크린샷)으로는 최초 진입 시점의 정적 레이아웃까지만 눈으로 확인 가능하고, **움직임 자체(흐르는 애니메이션, 스크롤 안무)는 이 방식의 DOM 계측으로만 검증 가능**하다.

## 검증 시 오진 함정

이 우회 자체가 검사 방식의 부작용을 일으킬 수 있다. 예: 특정 섹션을 최상단으로 끌어올리려고 **앞선 섹션들을 JS로 숨겼더니**, IntersectionObserver 기반 reveal 애니메이션이 트리거되지 않아 타겟 섹션의 opacity가 계속 0으로 남았다 — "이 섹션이 통째로 사라졌다"고 오판했지만, 정상 스크롤 상태로 다시 확인하니 `opacity: 0.999`로 멀쩡했다. **검사 방식이 만든 부작용과 실제 버그를 구분하려면, 의심되는 계측값은 검사 조건을 바꿔(원래 스크롤 위치·숨긴 요소 복원 등) 한 번 더 재현되는지 대조해야 한다.**

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
