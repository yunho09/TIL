---
tags: [react, zustand, 전역상태, 클로저, 면접]
updated: 2026-11-26
---

# Zustand 내부 동작 — 클로저·구독자 Set·useSyncExternalStore

Zustand는 상태를 React 트리 밖의 **클로저 변수**에 두고 **구독자 목록**을 관리한다. `setState`가 새 객체로 바꾸고 구독자에게 알리면, React 쪽은 `useSyncExternalStore`로 구독해 **selector로 고른 값**이 바뀐 컴포넌트만 다시 렌더한다.

## 원리를 줄인 코드

실제 라이브러리 코드는 더 복잡하고 버전마다 다를 수 있다.

```js
function createStore(init) {
  let state                      // 상태: 클로저 안의 평범한 변수
  const listeners = new Set()    // 구독자 목록

  const getState = () => state
  const setState = (partial) => {
    const next = typeof partial === 'function' ? partial(state) : partial
    if (Object.is(next, state)) return          // 같으면 아무것도 안 함
    const prev = state
    state = Object.assign({}, state, next)      // 새 객체로 교체 (얕은 병합)
    listeners.forEach((l) => l(state, prev))    // 구독자에게 알림
  }
  const subscribe = (l) => { listeners.add(l); return () => listeners.delete(l) }

  state = init(setState, getState)
  return { getState, setState, subscribe }
}

const useStore = (selector) =>
  useSyncExternalStore(api.subscribe, () => selector(api.getState()))
```

- `useSyncExternalStore`는 React 바깥 저장소를 구독하는 React 기본 훅
- 컴포넌트는 selector 결과를 `Object.is`로 비교하므로, 쓰는 값이 바뀔 때만 다시 그린다
- **"전역"인 이유**: 저장소가 React 트리가 아니라 **모듈 스코프(클로저)**에 있다. Provider 없이 import만 하면 어디서든 같은 저장소를 본다

## ToyVillage에서 쓴 곳

- 사이드바 열림 상태(`useSidebarStore`) 하나에만 사용
- 서버 데이터는 TanStack Query([[TanStack-Query-캐시-내부-동작-QueryCache-Observer]]), 조회 조건(탭·날짜·페이지)은 URL

## 면접에서 말할 문장

> Zustand는 상태를 React 밖의 클로저 변수에 두고 구독자 Set을 관리합니다. setState를 하면 새 객체로 바꾸고 구독자에게 알리며, React 쪽은 useSyncExternalStore로 구독해서 selector로 고른 값이 바뀐 컴포넌트만 다시 렌더링합니다.

## 꼬리 질문

- **Context API와 차이** — Context는 value가 바뀌면 그 Context를 쓰는 컴포넌트가 전부 리렌더. Zustand는 필요한 값만 구독하고 Provider도 필요 없다
- **왜 상태를 직접 바꾸면 안 되나** — `Object.is`로 참조를 비교하므로 같은 객체를 직접 고치면 바뀐 줄 모른다 ([[React-상태와-리렌더링-set함수와-불변성]])
- **TanStack Query와 공통점** — 둘 다 데이터를 React 밖 객체에 두고 구독 패턴으로 알리며, `useSyncExternalStore`로 React와 연결한다

## 출처

- Claude Code 세션 자동 캡처 (/work/code/project/ToyVillage-Admin-FE) — 2026-11-26. 세션의 설명을 정리한 것으로 라이브러리 소스·공식 문서 대조는 하지 않았다
