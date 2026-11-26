---
tags: [react, state, 렌더링, 면접]
updated: 2026-11-26
---

# React 상태와 리렌더링 — set 함수와 불변성

컴포넌트는 렌더마다 다시 실행되는 **함수**라서 지역 변수는 값을 기억하지 못한다. `useState` 값은 React가 컴포넌트 밖에 보관하고, **set 함수**를 불러야 React가 리렌더를 예약한다. 새 값은 `Object.is`로 비교하므로 객체·배열은 새로 만들어 넘긴다.

## 개념

- React가 컴포넌트 함수를 부르는 것이 **렌더**. 돌려준 JSX로 화면을 만든다
- 함수 안 지역 변수는 다시 불릴 때마다 처음 값으로 돌아간다
- React는 값 변화를 지켜보지 않는다. **set 호출이 알림**이다 (Vue처럼 자동 추적하는 방식과 다름)
- `Object.is(이전값, 새값)`이 같으면 리렌더를 건너뛸 수 있다

```tsx
let plain = 0                          // 바꿔도 화면 그대로, 렌더마다 0
const [count, setCount] = useState(0)  // React가 기억

animals.push('호랑이'); setAnimals(animals)  // ❌ 같은 배열 → "안 바뀜"
setAnimals([...animals, '호랑이'])           // ✅ 새 배열
```

## ToyVillage에서 쓴 곳

- `src/entities/team/ui/TeamDetailPanel.tsx` (세션에서 origin/develop 확인) — `editing`·`draftName`을 `useState`로 두고, "수정"을 누르면 `setEditing(true)` → 다시 렌더 → 입력창 표시
- 프로젝트 배경: [[ToyVillage-Admin-FE/프로젝트-현황]]

## 면접에서 말할 문장

> 컴포넌트는 렌더마다 다시 실행되는 함수라서 지역 변수는 값을 기억하지 못합니다. useState 값은 React가 컴포넌트 밖에 보관하고, set 함수를 불러야 React가 바뀐 걸 알고 리렌더를 예약합니다.

> set 함수는 Object.is로 이전 값과 비교하기 때문에, 객체나 배열은 직접 고치지 않고 새로 만들어서 넘깁니다.

## 꼬리 질문

- **값은 어디에 저장되나** — 컴포넌트마다 React가 만드는 내부 객체(**Fiber**). 내부 구현이라 버전마다 다를 수 있다
- **훅을 if문 안에서 못 쓰는 이유** — React는 이름이 아니라 **훅 호출 순서**로 값을 찾는다. 하나가 빠지면 순서가 밀린다
- **같은 값으로 set하면 렌더가 아예 안 되나** — 보통 건너뛰지만, 컴포넌트 함수를 한 번 더 부르고 자식으로 내려가지 않고 멈추는 경우가 있다. "절대 안 불린다"는 틀림

## 틀리게 말하기 쉬운 점

- **리렌더 ≠ DOM 전체 다시 그리기**. 함수를 다시 부르는 것이고 DOM은 바뀐 곳만 고친다
- state 변경 때만 리렌더되는 게 아니다. **부모가 리렌더되면** props가 같아도 자식이 기본으로 리렌더된다
- `setCount(count + 1)` 다음 줄의 `count`는 아직 옛 값이다. (Claude 보충) state는 렌더마다의 스냅샷이기 때문

## React 심화 공부 순서 (면접 대비 계획)

면접에서 내부 동작까지 깊게 묻는다는 이야기에 대비한 순서. 목표는 AI로 만든 코드를 **AI 없이 설명**하는 것. 조각마다 개념 → 짧은 코드 → ToyVillage 실제 사용처(origin/develop 확인) → 면접 문장 → 꼬리 질문 → 헷갈리는 점 순으로 본다.

1. 상태와 리렌더링 (이 페이지)
2. 렌더 단계와 커밋 단계, 가상 DOM, 재조정
3. `key`의 역할 (리스트, Suspense 경계)
4. 상태 업데이트의 비동기·배치 처리 (`isPending`으로 연타를 못 막고 ref로 막은 이유)
5. 클로저와 stale closure
6. `useEffect` 실행 시점·의존성·cleanup
7. `useRef` vs `useState`
8. `useMemo`·`useCallback`·`React.memo`의 효과 유무
9. Context API와 리렌더 범위
10. React 밖 저장소: 구독 패턴과 `useSyncExternalStore`
11. [[Zustand-내부-동작-클로저-구독자-useSyncExternalStore]]
12. 서버 상태 vs 클라이언트 상태
13. [[TanStack-Query-캐시-내부-동작-QueryCache-Observer]]
14. `React.lazy`·Suspense·코드 스플리팅
15. 에러 바운더리

## 관련

- [[React-useEffect-useLayoutEffect-폼제출-타이밍-버그]]
- [[라우트-코드-스플리팅-React-lazy-Suspense-key와-배럴]] — `key`로 경계를 새로 만드는 사례

## 출처

- Claude Code 세션 자동 캡처 (2026-11-26) — 세션의 설명을 정리한 것으로, 공식 문서 대조는 따로 하지 않았다
