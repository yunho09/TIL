---
tags: [react-router, 폼, 이탈방지]
updated: 2026-09-19
---

작성 중인 폼에서 이탈 경고를 띄우려면 앱 내부 이동은 `useBlocker`, 탭 닫기·새로고침은 `useBeforeUnload`로 나눠 막는다. 그런데 `useBlocker`는 **data router**(`createBrowserRouter` 등)에서만 동작해서, JSX `<Routes>` 방식이면 라우트 정의를 객체 구조로 옮겨야 한다.

## 구현 구조
- **앱 내부 이동**(사이드바 클릭 등): `useBlocker`로 막고 확인 모달에서 "나가기/계속 작성"을 고르게 한다.
- **탭 닫기·새로고침**: 라우터가 관여하지 않으므로 `useBeforeUnload`로 브라우저 기본 경고를 띄운다.
- **변경 여부 판단**: 폼이 제목·내용·분류·첨부 등의 변경 여부를 스스로 계산해 부모(블로커)에 알린다.
- 확인 모달은 shared로 올려 작성·수정 화면 여러 곳(ToyVillage-Admin-FE에서는 11곳)에서 재사용한다.

## 함정
- 기존 라우터가 `<BrowserRouter>` + JSX `<Routes>`면 `useBlocker`가 동작하지 않는다 → `createBrowserRouter` + `RouterProvider`로 전환이 선행 작업이다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-5) — 2026-09-19 (7월 공지 작성 화면 이탈 방지 구현 회고, 저장소 커밋 기록 기반)
