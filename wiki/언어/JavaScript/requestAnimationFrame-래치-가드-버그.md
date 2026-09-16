---
tags: [javascript, requestAnimationFrame, 버그패턴, 스크롤]
updated: 2026-09-16
---

# requestAnimationFrame 래치(latch) 가드 버그

스크롤·리사이즈 핸들러를 rAF로 스로틀링할 때 흔히 쓰는 패턴이다:

```js
var tick = false;
function onScroll(){
  if (tick) return;
  tick = true;
  requestAnimationFrame(frame);
}
function frame(){
  tick = false;
  // ... 실제 작업 ...
}
```

## 문제

초기화 시점에 `tick`을 미리 `true`로 두고 rAF를 걸어두는 구현이 있으면, **그 최초 rAF 콜백이 끝내 실행되지 않는 환경**에서 `tick`이 영원히 `true`로 잠긴다(래치). 이후 `onScroll`이 아무리 호출돼도 가드에 걸려 `requestAnimationFrame(frame)`이 다시는 예약되지 않는다 — 스크롤 이벤트 자체는 정상 발생하는데 그에 대한 반응만 통째로 죽는다.

**재현 조건**: 브라우저가 백그라운드 탭에서 페이지를 로드하는 경우가 대표적이다. 많은 브라우저가 비활성 탭에서는 `requestAnimationFrame` 콜백을 아예 호출하지 않거나 크게 지연시키므로, 링크를 새 탭으로 열기만 해도 이 상태에 빠질 수 있다. 사용자가 나중에 그 탭을 활성화해도, `tick` 래치가 걸려 있으면 스크롤 반응 기능이 죽은 채로 남는다.

## 해결

가드를 스스로 회복시키는 두 가지 장치를 더한다.

1. **가시성 복귀 이벤트에서 강제 리셋**: `visibilitychange`(탭이 다시 보일 때)와 `pageshow`(bfcache 복원 등) 이벤트에서 `tick=false`로 리셋하고 즉시 한 번 더 `onScroll`을 호출한다.
2. **워치독 타이머**: `setInterval`로 주기적으로 "마지막 프레임 실행 시각"과 현재 시각의 차이를 확인해, 일정 시간(예: 900ms~1.3s) 이상 프레임이 안 돌았다면 `tick`을 강제로 리셋하고 다시 예약한다.

이 두 장치는 "정상적으로 멈춘 것"(예: 탭이 실제로 숨겨진 상태에서 `document.hidden` 체크로 애니메이션을 일부러 멈춘 경우)과 "래치에 걸려 비정상적으로 멈춘 것"을 구분하기 어려운 상황에서도, 탭이 다시 보이는 순간 확실히 복구시켜준다는 점에서 안전망 역할을 한다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
