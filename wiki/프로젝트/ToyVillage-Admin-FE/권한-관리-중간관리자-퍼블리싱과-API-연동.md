---
tags: [toyvillage, frontend, publishing, api, harness, e2e]
updated: 2026-11-05
---

# 권한 관리(중간 관리자 지정·해제) 퍼블리싱과 API 연동 (#289~#298)

관리자 웹에 중간 관리자(`MIDDLE_ADMIN`)를 지정·해제하는 화면을 퍼블리싱(#289/PR #290) → API 연동(#291/PR #292) → CodeRabbit 반영(#295/PR #296) → `develop → main`(PR #298) 순서로 진행한 기록. 하네스가 막은 지점(백엔드 API 부재, 승인 명령), Figma 수정에 따른 설계 변경, 재사용할 판단을 정리한다.

## 화면과 구조
- 라우트 `/settings/permissions`, 사이드바 `권한 관리` 메뉴는 mock에 이미 있었으나 `to`가 없어 연결이 필요했다.
- 목록과 지정 후보는 **`GET /app/admin/employees` 한 번**으로 받아 `role`로 나눈다: `MIDDLE_ADMIN`=목록, `EMPLOYEE`=후보. 지정·해제는 `PATCH /app/admin/employees/{appAdminId}/role`에 `MIDDLE_ADMIN`/`EMPLOYEE`를 보내는 하나의 API.
- 컴포넌트: `PermissionsPage.tsx`, `MiddleManagerTable.tsx`, `AssignMiddleManagerDialog.tsx`, 공용 `shared/ui/Radio.tsx`·`Checkbox.tsx`.
- feature끼리는 서로 import할 수 없어서, 다른 feature(엑셀 다운로드 `ExportFormatField`)에 있던 라디오·체크박스를 쓰려면 `shared/ui`로 옮겨야 한다. 이런 **공용 변경은 승인 항목으로 따로 올린다**.

## 흐름과 하네스가 멈추는 지점
- 퍼블리싱: spec이 없으면 AI가 임의로 만들지 않고 사용자에게 확인(새 spec 생성/기존 연결). ② 시나리오 승인·⑦ 육안 확인은 사람 게이트.
- API: `yarn harness:api:approve <feature>`와 `--freeze`는 AI 세션(`CLAUDECODE`)에서 **거부**되어 개발자가 직접 터미널에서 실행한다. 승인 전에는 API 코드·테스트를 쓰지 않는다. 명령 목록을 만들어 넘기고 "했어"를 받은 뒤 구현.
- **백엔드에 API가 없으면 구현하지 않고 질문 목록을 반환**한다. 이번엔 처음에 지정·해제 API와 지정일 필드가 없어 멈췄고, 백엔드가 만든 뒤 Swagger(`/v3/api-docs`, `.env`의 `SWAGGER_USERNAME/PASSWORD`)·백엔드 `develop` 코드·Notion 명세를 따로 대조해 계약을 작성했다. Swagger는 필수 여부가 없어 `role`은 백엔드 코드를 보고 필수로 판단.
- 승인된 명세를 바꾸는 수정(#295: "성공 뒤에만 갱신"→"실패 뒤에도 재조회")은 명세·시나리오를 고친 뒤 **재승인 → 코드 → `--freeze` 재실행**이 필요하다.

## Figma 수정으로 바뀐 설계
- 지정 모달: 체크박스·여러 명·`2명 지정하기` → **라디오·한 명·`지정하기` 고정**(선택 전 비활성). 목록에서 `지정일` 열 제거(백엔드에 필드도 없었음).
- 그 결과 "여러 명을 차례로 호출, 중간 실패 시 중단" 설계가 필요 없어졌다(요청 1회 = 부분 실패 없음). 디자인이 바뀌면 시나리오·계획·계약을 같이 재검토해야 한다.

## 실패 후 재조회 (#295/PR #296)
- CodeRabbit 지적: 해제 실패 뒤 목록이 옛 상태로 남음(다른 탭에서 이미 바뀐 경우 등). 원인이 같은 **지정 실패도 함께** 고침.
- 동작: 지정 실패=모달 유지+재조회(고른 직원이 후보에서 빠지면 `지정하기` 비활성), 해제 실패=모달 닫고 재조회, **재조회까지 실패하면 보던 목록·열린 모달 유지**. 전체 오류 카드는 `isLoadingError`(첫 조회 실패)일 때만.
- 부작용: 재조회로 행이 사라지면 키보드 초점이 있던 해제 버튼도 사라져 초점이 문서 처음으로 떨어진다(CodeRabbit 제안: `중간 관리자 지정` 버튼으로 이동). 마우스에는 영향 없고 드물어서 **반영하지 않기로 결정**. docstring 비율 경고도 손대지 않음.
- 테스트: 가짜 서버가 200이면 응답 본문과 무관하게 역할을 바꾸게 해 "서버엔 반영됐지만 응답 형식이 틀린 경우"(S8)를 검사.

## 작은 함정들
- **검색 입력의 브라우저 기본 지우기 X**(`type="search"`): 색이 튀고 Figma에 없어 `global.css`의 `input[type='search']::-webkit-search-cancel-button { display: none; }` 한 줄로 전역 제거. 영향 범위는 처음 3곳으로 말했으나 **4곳**(팀 관리 `AddTeamMemberDialog` 포함)이었음 — 전역 규칙 영향 범위는 grep으로 센 뒤 말할 것. 이제 검색어는 손으로 지워야 한다.
- **`Radio.tsx`의 `url("…")`은 큰따옴표여야 한다**: 켜진 아이콘이 인라인 SVG라 안에 작은따옴표가 있다. `yarn prettier`가 작은따옴표로 바꾸면 아이콘이 깨질 수 있어 주석으로 경고를 남김.
- 공용 토스트·취소 버튼이 Figma와 약간 다른 것(토스트 그림자 색, 해제 모달 `취소` 테두리)은 다른 화면까지 바뀌므로 **그대로 두고 보고만** 했다.
- `git mv`로 스테이징된 이름 변경이 첫 `docs` 커밋에 섞이면 푸시 전에 `git reset -q origin/develop` 후 경로 지정으로 다시 커밋. 이후 `mv` 사용.
- 개발 서버가 메모리 부족으로 자동 종료될 수 있다(Claude Code가 백그라운드 셸을 정리). 다시 띄우라고 하기 전엔 띄우지 않는다.
- 로컬 개발 서버가 **스테이징 API**를 가리키면 지정·해제가 실제 직원 역할을 바꾼다 — 눌러본 뒤 되돌릴 것.
- 로그인 응답 검사(`authApi.ts` `isAppAuthRole`, `session.ts` `isSessionUser`, `AppSessionRole`)는 `APP_ADMIN`·`EMPLOYEE`만 허용 → `MIDDLE_ADMIN`은 `로그인에 실패했습니다`로 막힌다(브라우저에서 응답을 가짜로 넣어 재현, 토큰 저장 안 됨). **중간 관리자는 앱 전용이라 정상 동작**으로 결론, 고치지 않음. 다만 비밀번호가 맞아도 같은 문구라 이유를 알 수 없다(필요하면 문구 개선).

## 운영 배포 전 확인(미해결)
- 백엔드 `main`(`release :: v1.0.0`)에는 역할 변경 API(#259)가 없고 `develop`에만 있다. 운영이 `main` 기준이면 프론트를 올렸을 때 **지정·해제가 전부 실패**한다(운영 기준 여부 미확인, 백엔드 CD는 `develop`에만 걸려 있음). `role` 응답은 백엔드 `main` 코드에 있으나 운영 반영은 미확인 — 없으면 직원 계정 관리 목록까지 조회 실패로 표시.
- 실서버 동작은 직접 눌러본 결과가 없다(CI·로컬 테스트는 모두 가짜 서버 기준). PR #298은 열린 상태.
- Notion 명세에 역할 변경 API 행 없음(백엔드에 추가 요청 필요). 직원 생성 계약(`app-admin-employee-create`)에 Swagger의 `role` 요청 필드가 빠져 있으나 서버 기본값이 `EMPLOYEE`라 지장 없음.
- `feed-detail` S7, `feed-list` S3 e2e가 #290·#292·#296에서 계속 재시도로만 통과 — 원인 미조사, 별도 이슈 후보([[ToyVillage-Admin-FE/공지사항-날짜-시간-제거]]에도 같은 기록).

## 관련
- [[ToyVillage-Admin-FE/프로젝트-현황]], [[ToyVillage-Admin-FE/하네스-시스템]], [[ToyVillage-Admin-FE/팀-작업-규칙-하네스-문서]], [[ToyVillage-Admin-FE/엑셀-다운로드-API-연동]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-4, 2026-11-05)
