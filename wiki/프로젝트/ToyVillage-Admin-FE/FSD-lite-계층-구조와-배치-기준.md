---
tags: [FSD, 아키텍처, ESLint, 토빌, 면접]
updated: 2026-11-20
---

# FSD-lite 계층 구조와 배치 기준 (토빌 어드민)

토빌은 FSD(Feature-Sliced Design)에서 `widgets`를 뺀 **5계층 "FSD-lite"**로 나눴고, 위층만 아래층을 import하게 `eslint-plugin-boundaries`로 강제한다. 위·아래는 중요도가 아니라 **구체적인 정도**(위=이 앱에만 있는 것, 아래=어디든 옮겨 쓸 수 있는 것)다. 컴포넌트·훅·상태를 어느 층에 둘지의 판단 기준도 여기 정리한다. 성능 쪽 적용은 [[라우트-코드-스플리팅-React-lazy-Suspense-key와-배럴]], 숫자 대조는 [[포트폴리오-숫자-코드-대조와-면접-주의점-2026-11]] 참고.

## 5계층과 import 규칙
```
app       ← 제일 위: 앱 시작, 라우터, 인증 인터셉터 장착, Sentry 초기화
pages     ← 화면
features  ← 사용자 동작 단위 기능 (폼·버튼·대화상자·검증·상태)
entities  ← 도메인 데이터 (API·타입·표시용 글자·작은 표시 UI)
shared    ← 제일 아래: 공통 Axios, queryClient, 범용 UI, 유틸
```
- **"위에서 아래로 import한다" = 위층 파일이 아래층 코드를 가져다 쓴다**는 뜻이다. 예: `pages/activity/ActivityPage.tsx`가 `@/shared/lib`·`@/entities/activity`를 import(OK), `entities/activity/api`가 `@/shared/api/axios`를 import(OK). shared의 버튼이 `@/pages/...`를 import하면 lint 에러 → CI 머지 차단.
- 설정(`eslint.config.js` boundaries 규칙): 기본값 금지 + `{ from: 'entities', allow: ['entities','shared'] }`, `{ from: 'shared', allow: ['shared'] }`처럼 층별 허용 목록. app은 전부 import 가능, 다른 층이 app을 import하는 건 막힘.
- **같은 층끼리는 막지 않는다**(entities끼리 실제로 import: 업무 보고 → 업무). 그래서 "층 방향만 강제했다"가 정확한 표현.
- 이유: ① shared 버튼이 특정 화면을 import하면 버튼 수정 영향이 화면까지 퍼짐 ② pages↔shared 순환 의존 방지 ③ 아래로 갈수록 아무것도 몰라서 재사용 쉬움.
- **정식 FSD는 pages와 features 사이에 `widgets`가 있어 6계층** → "widgets 없는 FSD-lite"로 말한다.
- 포폴에 "(FE 팀)"으로 적힌 항목: FSD 규칙 설정은 FE 팀원이 주도. "제가 설계했다"가 아니라 "팀에서 정했고 그 규칙 안에서 화면을 만들었다".

## entities vs features
- 한 줄: **entities = 명사(데이터), features = 동사(사용자 동작)**.
- `entities/<도메인>`(18개: activity, auth, task, team, species, notice …): API 함수·서버 타입(`api/types.ts`)·화면 타입(`model/types.ts`)·표시 글자(`model/labels.ts`)·그 데이터를 보여 주는 작은 UI(`TaskStatusBadge`, `TaskTable`)·`index.ts`(배럴). API 88개(코드로 세면 89~91개라 "약 90개")를 이 18개로 묶었고 API 함수 대부분이 여기 있다.
- `features/<동작>`: 이름이 대부분 동사(`create-task`, `review-task-report`, `export-excel`, `assign-middle-manager`, `login`, `sidebar`, `species-form`). 폼·버튼·대화상자·입력 검증(`validation`)·상태(`useSidebarStore`). API는 대부분 entities 것을 가져다 쓰고 그 동작에만 필요한 API 몇 개만 직접 둔다(`dashboard`, `create-account`).
- 예: 업무 등록 = `pages`가 `features/create-task`의 폼(`TaskForm`, `TaskAssigneeTree`, `CreateTaskButton`)을 쓰고, 폼이 `entities/task` API로 저장.
- (정정 기록) 세션 중 "features 층이 얇다"고 했다가 실제 폴더를 열어 보고 "폼·대화상자가 많이 들어 있다"로 바로잡았다 — 층 설명은 폴더를 열어 확인하고 말할 것.

## 컴포넌트·훅·상태를 어디에 두나
| 대상 | 두는 곳 | 기준 |
|---|---|---|
| 도메인을 모르는 범용 UI(버튼, 확인 대화상자, 스켈레톤, 첨부 미리보기) | `shared/ui` | 어느 화면에서나 쓰고 업무/동물을 몰라도 됨 |
| 한 화면에서만 쓰는 UI | `pages/<화면>/ui` | 예: `pages/activity/ui/activityColumns` |
| 여러 화면에서 쓰지만 특정 데이터에 묶인 UI | `entities/<도메인>/ui` | 업무 상태 배지 등 |
- 토빌은 대부분 shared 아니면 pages. 컴포넌트가 전부 entities에 있는 게 아니다.
- **커스텀 훅**(useState 등 기본 훅은 어디서나 사용): `app`(`usePageViewTracking` = 화면 이동마다 GA4 페이지뷰), `features/*/model`(`useSidebarStore`, `useLogout`, `useReviewTaskReport`, `usePermissionAssignment` = 동작에 묶인 상태·처리), `shared/ui`(`useExitAnimation`, `useFocusFrame` = 도메인 무관 범용 동작). 기준: 범용이면 shared, 특정 동작이면 그 feature의 `model`, 앱 전체 1회면 app.
- **"상태"** = 화면이 지금 어떤 모습인지 기억하는 값(열림/닫힘, 입력값, 검증 오류, 작업 중). 서버에 저장된 데이터와 분리해 관리한다: **서버 데이터**(업무 목록·팀 구조)는 TanStack Query가 불러오고 캐시(쿼리 키는 entities), **화면 상태**는 features/컴포넌트 안. 앱 전체가 공유하는 화면 상태는 **사이드바 열림 하나뿐**이라 그것만 Zustand, 나머지는 컴포넌트 내부 React state.

## 인증 인터셉터가 app에서 붙는 이유 (FSD 때문)
- 공통 Axios 인스턴스(`shared/api/axios.ts`: baseURL=`VITE_API_BASE_URL`(https 검사), timeout 10초, 인터셉터 3개 — 요청에 `Authorization: Bearer` 부착 / 에러 Sentry 보고(401·403은 재발급 처리라 제외) / 401·403이면 재발급→재요청→실패 시 로그아웃, 동시 401도 재발급은 **한 번만**(single-flight)).
- 그런데 재발급 API는 `entities/auth`에 있어 더 아래층인 shared가 import할 수 없다 → **맨 위 app이 시작 시점(`main.tsx`)에 `configureApiAuthentication`으로 인터셉터를 붙인다**. "위→아래 import 규칙을 지키면서 기능을 연결하는" 사례.
- 팀 규칙은 "모든 API 호출은 공통 인스턴스"이고 파일 CDN 요청(`fetchStoredFile`, 기본 `fetch`)만 예외 — 이유와 동작은 [[공용-인증-axios-외부도메인-토큰유출]]. 보강: 이 서버는 토큰이 없거나 만료돼도 401이 아니라 **403**을 주기 때문에 인터셉터가 403도 재발급 대상으로 보고, 그래서 CDN의 403 한 번이 재발급→재요청→또 403→로그아웃으로 번졌다. `fetch`는 4xx·5xx여도 던지지 않아 `response.ok`를 직접 확인하고, 미리보기를 닫으면 `AbortSignal`로 요청을 취소한다.

## 면접: "FSD 왜 썼어요?" / "규칙 때문에 설계가 바뀐 적?" (2026-11-21 모의면접)
- FSD를 **고른 이유는 기록에 없다**(7/5 FE 팀원이 첫 세팅 커밋에서 "FSD-lite"로 한꺼번에 넣음, Vite·React·TS·TanStack Query·Zustand·Emotion·Playwright도 같은 커밋). "제가 골랐다"고 하면 "다른 구조와 비교했어요?"에서 막힌다. 답 틀: **누가 정했는지 → 써 보며 좋았던 점 → 한계**.
  > 초기 구조는 FE 팀원이 FSD로 잡았고 저는 그 위에서 개발했습니다. 좋았던 점은 의존 방향이 ESLint로 강제된다는 것입니다. 예를 들어 토큰 재발급 인터셉터가 인증 엔티티 함수를 써야 해서 shared가 아니라 app 레이어로 올리게 됐고, 규칙이 없었다면 공통 코드가 특정 기능에 묶였을 겁니다.
- 단점 질문: 작은 기능도 어느 층에 둘지 고민, 처음 보는 사람에겐 규칙이 어려움 → 그래서 같은 층끼리는 막지 않는 가벼운 방식. FSD가 아니면: 종류별 폴더(`components`/`hooks`/`pages`)는 작을 땐 편하지만 화면이 많아지면 한 기능 코드가 흩어짐. "화면 50개가 넘어도 기준이 명확했다"는 실제로 그렇게 느꼈을 때만.
- **재발급 인터셉터를 app으로 올린 이유**(한 줄): 인터셉터가 `entities/auth`의 `reissueAppToken`을 써야 하는데 shared가 entities를 import하면 층 규칙 위반 → 위 레이어 `app/config/configureApiAuthentication.ts`로 올림. 인터셉터 = 모든 요청·응답 사이에 끼는 코드(요청에 토큰 부착 → 401/403이면 재발급 → 새 토큰으로 원 요청 1회 재시도).
  - 대안(면접관이 "다르게는?"이라 파고들 때만): ① `reissueAppToken`을 shared로 내림(인증 도메인 코드가 공통 층에 섞임) ② shared엔 "재발급 함수를 받아 인터셉터를 붙이는 함수"만 두고 app이 시작 시 함수를 주입(의존성 주입). 팀은 app에서 연결하는 쪽을 택함 — **왜 그렇게 정했는지는 기록 없음**, 네가 고른 것처럼 말하지 않는다.
  - 같은 이유로 **역할 타입**(`APP_ADMIN`·`EMPLOYEE`)은 shared가 갖고 entities/auth가 가져다 씀(반대면 규칙 위반, 코드 주석에도 이유 있음).
- 응답 검사·쿼리 구조: [[응답을-unknown으로-받고-타입-가드로-검사하기]], [[TanStack-Query-기본설정-쿼리키-계층과-삭제-시-removeQueries-먼저]].

## `@` 별칭과 폴더 import (질문이 나오기 쉬운 기초)
- `@` = `src`. `tsconfig.app.json`의 `paths: {"@/*": ["./src/*"]}`(TypeScript용)와 `vite.config.ts`의 `alias: {'@': './src'}`(빌드 도구용) 둘 다에 적어야 한다. 파일 위치와 무관하게 `@/shared/api/axios`로 쓸 수 있어 `../../../` 세기를 없앤다.
- import에 **폴더 경로만** 쓰면 빌드 도구가 그 폴더의 `index.ts`를 자동으로 읽는다(약속된 규칙). 그 `index.ts`(배럴)가 여러 파일을 대신 export해 한 줄로 가져오게 해 주지만, 코드 분할에서는 이 "안내 데스크"가 문제 — [[라우트-코드-스플리팅-React-lazy-Suspense-key와-배럴]].

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-11-20 (면접 대비 설명 세션)
- Claude Code 세션 자동 캡처 (/home/yunho/.config/Claude/scratch-workspaces/68eef3b8-da94-4db5-b563-9565e29705a3/4cebf249-3d72-4a77-a55b-e742194ef06a/scratch-2026-11-20-d28dbc) — 2026-11-21 (모의면접)
