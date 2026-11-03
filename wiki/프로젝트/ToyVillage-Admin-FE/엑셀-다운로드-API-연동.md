---
tags: [toyvillage, excel, api, axios, blob, toast, react-query]
updated: 2026-11-03
---

# 엑셀 다운로드 API 연동 (#282 / PR #285)

`GET /excel`을 Swagger 기준으로 연동한 작업(이슈 #282 → PR #285, develop→main은 #286). 파일을 Blob으로 받는 axios 설정, 서버 `types`와 화면 항목의 대응, ZIP/SHEET 차이, 토스트·중복 클릭 처리처럼 **파일 다운로드 API를 붙일 때 재사용할 판단**을 정리한다. 하네스 절차 자체는 [[ToyVillage-Admin-FE/하네스-시스템]], 이전 퍼블리싱은 [[ToyVillage-Admin-FE/프로젝트-현황]] 참고.

## API 계약 (Swagger 기준)
- `GET /excel` 파라미터: `types`(배열), `mode`(`SHEET`/`ZIP`), `startDate`, `endDate`. Notion 명세 DB에는 이 API가 없어 Swagger만 근거로 삼았다.
- 서버 `types` 11개 중 `DOCUMENT`(자료실)는 화면에 항목이 없어 보내지 않는다. 휴관일은 서버에 대응 값이 없고 디자인에서도 최종 제거됐다.
- 화면 항목 10개 = 서버 값 10개 1:1: `ANIMAL`(동물 리스트, 기간 미적용), `ANIMAL_OBSERVATION`, `FEED_LOG`(4층), `FIFTH_FLOOR_FEED_LOG`, 업무관리, 업무보고, 업무일지관리, 단체예약, 공지사항, 직원 계정 관리(`TEAM_MEMBER`).
- 초기에 "개체관리→`ANIMAL`+`ANIMAL_OBSERVATION`", "먹이→4층+5층"으로 묶어 보냈다가, 디자인이 항목을 쪼개면서 1:1이 되어 묶음 코드를 삭제했다. 서버는 원래 값별로 따로 만들고 있어서 **항목 분리에는 백엔드 수정이 필요 없었다.**

## 서버 동작 (백엔드 코드·실제 ZIP으로 확인)
- **ZIP**: `types` 하나당 xlsx 하나 + 첨부파일은 `첨부파일/` 폴더에 원본으로 따로. 파일명은 서버 코드가 정함(`동물.xlsx`, `관찰일지.xlsx`, `급여일지.xlsx`, `5층 급여일지.xlsx`, `업무지시.xlsx`, `업무보고.xlsx`, `사육일지.xlsx`, `예약.xlsx`, `공지사항.xlsx`, `팀구성원.xlsx`). 화면 이름과 다르다.
- **SHEET(.xlsx)**: 시트별로 한 파일, 첨부는 엑셀 안에 포함된다.
- 개체 목록은 기간과 무관하게 전부 나가고, 기간은 관찰일지에만 적용된다. 관찰일지·업무보고는 서버가 작성 시각(`createdAt`)으로 거르는데 화면 문구는 "관찰일/제출일 기준"이라 같은 뜻인지 백엔드 확인이 필요하다.
- 데이터가 없어도 헤더만 있는 파일을 **200**으로 준다 → 프론트에서 성공과 "데이터 없음"을 구분할 수 없어 Figma의 데이터 없음 토스트는 구현하지 못했다(백엔드가 구분 응답을 줘야 함).
- CORS에 `exposedHeaders`가 없어 브라우저가 `Content-Disposition`을 못 읽는다 → **파일명은 화면에서 생성**: `{항목명|통합}[_시작_종료].{xlsx|zip}`. 기간 미적용 항목만 고르면 날짜를 붙이지 않는다.
- 서버가 첨부·사진까지 메모리에 올려 파일을 만들기 때문에 큰 요청은 서버/프록시 시간 초과에 걸릴 수 있다((Claude 보충) 추측; 한 번 실패한 원인은 끝내 확인 못 함).

## 프론트 구현 요점
- 구조: `entities/excel-export/{api,model}` + `features/export-excel/{lib,ui}`. `target`은 화면 항목 하나(`{id, name, basis, dated}`)이고 `id`가 곧 서버 `types` 값이다.
- `toExcelExportQuery`: 화면의 `targets`를 `types`로 이름 바꾸고, 날짜가 `null`("전체 기간")이면 키 자체를 뺀다. 항목은 `sortTargets`로 항상 화면 순서로 정렬해 요청한다.
- `exportExcel`의 axios 옵션:
  - `responseType: 'blob'` — 없으면 엑셀이 깨진다.
  - `paramsSerializer: { indexes: null }` — 배열이 `types[]=A&types[]=B`가 아니라 서버가 원하는 `types=A&types=B`로 나간다.
  - `timeout: 300_000` — 공통 10초는 파일 생성 중 끊기므로 이 요청만 5분.
- **에러 본문도 Blob**: `responseType: 'blob'`이면 413 오류 JSON도 Blob으로 온다. `await body.text()` → `JSON.parse` → `message`를 꺼내 토스트로 쓰고, 파싱 실패·그 외 상태(500/네트워크/시간 초과)는 공통 문구로 떨어뜨린다. 그래서 공통 문구만으로는 500과 네트워크 실패를 구분할 수 없다 → 실패 조사 시 Network 탭 Status가 필요.
- 413은 두 종류(건수 초과 / 첨부 총용량 초과)이고 서버 문구로만 구분된다.
- **연속 클릭 중복 요청**: 버튼 `disabled`는 React Query `isPending`이 `setTimeout(0)` 뒤 반영되고 Playwright `clickCount: 2`는 두 클릭을 동시에 보내므로 두 번째 클릭이 새어 들어갈 수 있다. 코드래빗이 "테스트만 고치라"고 했지만 앱의 틈이 남아, `exportingRef`로 첫 클릭 즉시 막고 `onSettled`에서 해제했다(다른 폼의 `submittingRef` 패턴과 동일).

## 토스트 결정
- Figma에 `엑셀 다운로드 · 토스트` 섹션이 따로 있었다: 성공 `엑셀 파일을 내려받았습니다`, 실패 `엑셀 파일 생성에 실패했습니다`, 데이터 없음 `선택한 기간에 데이터가 없습니다`(미구현). 처음에는 명세의 "Figma에 없다"만 믿고 문구를 지어내 틀렸다. **교훈: Figma는 메인 프레임뿐 아니라 같은 페이지의 형제 섹션까지 확인한다.**
- 공통 토스트는 폭 440px 고정·글자 28px이라 한 줄에 약 12자만 들어가 긴 문구가 3줄로 꺾인다. 공통 `Toast`에 선택 prop `fitContent`(fit-content, min 440, gap 12)를 추가해 엑셀 화면에서만 켰다. 토스트 폭 자체를 늘리면 전 화면에 영향이 가서 건드리지 않았다.
- 남긴 차이: Figma 엑셀 토스트는 테두리 아이콘+색 그림자, 공통 토스트는 채운 아이콘+검은 그림자(먹이 급여 토스트 Figma에도 색 그림자) → 공통 토스트 문제로 보고 별도 이슈 후보.

## 반응형·레이아웃 (Claude 판단)
- 항목 카드 5열(너비 232px, padding `24px 20px`, 체크박스-글자 gap 12px). Figma에 없는 1384px 이하는 2열로 전환 — 더 좁으면 `직원 계정 관리`가 줄바꿈되기 때문.

## 하네스 운영 메모
- 디자인이 바뀔 때마다(파일 형식 카드 추가 → 휴관일 제거 8개 4열 → 10개 5열) 명세·시나리오가 바뀌어 **재승인 + `--freeze`**를 반복했다. 승인 명령은 AI 세션에서 거부되므로 사용자가 직접 터미널에서 실행한다.
- 테스트 코드는 승인 후 동결되므로 리뷰 대응은 가능하면 앱 코드에서 고쳐 재승인을 피한다(ref 가드가 그 예).
- 로컬 e2e는 금지, 첫 실행은 PR CI. 커밋은 계층별 분리라 중간 커밋 하나만 체크아웃하면 타입 검사가 깨질 수 있다(브랜치 최신은 통과).

## 열린 질문 (2026-11-03 기준)
- 백엔드: CORS에 `Content-Disposition` 노출, `createdAt` 기준이 "관찰일/제출일"과 같은 뜻인지, Notion 명세 추가, 데이터 없음 구분 응답.
- 현재 버전으로 실서버 다운로드를 아직 못 해봤고, 이전 버전의 1회 실패 원인도 미확인.
- #286(develop→main) 머지 전 운영 API 서버에 `/excel`·5층 급여일지 등이 반영됐는지 확인 필요(백엔드 `main`은 `develop`보다 1163커밋 뒤처져 있어 운영 배포 기준인지 불명).

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-4)
