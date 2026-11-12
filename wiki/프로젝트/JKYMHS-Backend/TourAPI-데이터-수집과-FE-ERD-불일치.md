---
tags: [TourAPI, 데이터수집, ERD, JKYMHS, 어디갈래]
updated: 2026-11-12
---
# TourAPI 데이터 수집(지역·관광지·축제)과 FE↔ERD 불일치 정리

"어디갈래" 백엔드의 지역·관광지·축제 데이터는 한국관광공사 **국문 관광정보 서비스_GW**(TourAPI) 하나로 전부 받는다. 개발계정은 하루 1,000건이라 상세 정보 수집이 약 16일 걸리고, 삭제 예정인 옛 코드(`areaCode2`) 대신 법정동·새 분류 체계를 써야 한다. FE 요구와 ERD가 어긋난 곳은 별도 표로 정리했다. 프로젝트 전반은 [[JKYMHS-Backend/프로젝트-현황]], 이 데이터를 쓰는 PR 구성은 [[JKYMHS-Backend/남은-API-스택-PR-구성과-개발환경-Docker]].

## API 개요
- 신청 대상은 **국문 관광정보 서비스_GW**(data.go.kr 15101578) 1개, Base URL `https://apis.data.go.kr/B551011/KorService2`. 오퍼레이션 이름은 끝이 `2`. 공통 파라미터 `serviceKey`(**Decoding 키**), `MobileOS=ETC`, `MobileApp`, `_type=json`.
- 개발계정은 **자동 승인·하루 1,000건**. 승인 직후엔 게이트웨이 반영에 보통 1시간 안쪽이 걸려 `SERVICE_KEY_IS_NOT_REGISTERED_ERROR`가 날 수 있다(이번엔 바로 동작). 더 필요하면 활용사례 등록 후 운영계정 한도 증가 신청.
- 키는 사용자가 직접 `.env`의 `TOUR_API_KEY=`에 넣는다(Claude가 브라우저에서 키를 꺼내 옮기지 않음). `echo '키=복사한키'` 예시 문구를 그대로 실행하면 가짜 값이 `.env`에 들어가니 **예시 자리표시자는 실제 값으로 바꿔 실행**해야 한다. 빈 `TOUR_API_KEY=` 줄을 만들어 두고 편집기에서 붙여 넣는 편이 안전.

## 데이터별 오퍼레이션
| 필요한 것 | 오퍼레이션 | 비고 |
|---|---|---|
| 지역(시·도→시·군·구) | `ldongCode2` | 법정동 코드 `lDongRegnCd`/`lDongSignguCd` |
| 분류 | `lclsSystmCode2` | 새 분류 lclsSystm1/2/3 |
| 관광지(12)·문화시설(14)·음식점(39)·숙박(32) 목록 | `areaBasedList2`, `searchStay2` | |
| 축제 | `searchFestival2` | `eventStartDate` 필수 |
| 주소·좌표·개요·대표 이미지 | `detailCommon2` | |
| **운영 시간·쉬는 날·행사 시간** | `detailIntro2` | `usetime`/`restdate`, 축제는 `eventstarttime`/`playtime`/`usetimefestival` |
| 추가 이미지·요금 등 반복 정보 | `detailImage2`, `detailInfo2` | |
| 변경·삭제 동기화 | `areaBasedSyncList2` | `modifiedtime`, `showflag` |
| 반경 목록·반려동물 | `locationBasedList2`, `detailPetTour2` | |

## 꼭 알아둘 점
- **`areaCode2`·`categoryCode2`는 "미사용·삭제 예정"**. 지역은 법정동 코드, 분류는 lclsSystm. FE 리서치 기록상 옛 분류(cat1~3)는 관광지의 31%만 채워지고 새 분류는 100% 채워져 있어 실내·실외·스타일 매핑은 새 분류 기준.
- **시·도 코드가 옛 체계와 다르다**: 시·도 16개, 광주+전남이 `12 전남광주통합특별시` 하나로 묶임, 세종은 `36110`(시·도 코드가 5자리), 강원 `51`, 전북 `52`. 화면에 지역 코드를 고정하지 말고 데이터에서 읽어야 한다. 세종처럼 시군구 코드가 시도 코드와 같은 경우는 시군구 행을 따로 만들지 않도록 처리.
- lclsSystm 대분류: AC, C01(추천코스), EV, EX, FD, HS, LS, NA, SH, VE.
- 축제는 날짜가 빠진 것을 목록에서 제외, 진행 상태는 KST 기준. 이번 수집(지난 1년치 876건) 중 오늘 기준 진행·예정은 102건.
- 인기 점수는 TourAPI에 없어 찜·조회 수로 자체 집계(1시간 주기). 출발지 검색은 TourAPI가 아니라 카카오 로컬.
- 전국문화축제표준데이터는 TourAPI 축제가 부족할 때만 보조로(지자체 입력이라 중복 제거 필요).

## 수집 결과와 운영 방법 (2026-11-10)
- `tools/tourapi/collect.py`(표준 라이브러리만): `codes` → `lists` → `intro --limit N` → `export`. `intro`는 항목당 1회 호출하고 하루 한도에 걸리면 멈췄다가 이어서 받는다. `export`는 ERD 컬럼에 맞춘 CSV 4개(`regions`, `lcls_codes`, `contents`, `content_details`)를 만들고 좌표·날짜 없는 항목을 뺀다. 결과 폴더(`tools/tourapi/data/`)는 `.gitignore`.
- 받은 양: 지역 284, 분류 246, 콘텐츠 32,548(관광지 15,314·음식점 13,413·숙박 2,945·축제 876). 운영 시간은 950건 중 941건에 있지만 **쉬는 날은 71건뿐**. 상세 소개는 **16,190건 중 950건**만 받아 15,240건이 남음(하루 950건씩 약 16일).
- 매일 `python3 tools/tourapi/collect.py intro --limit 950 && python3 tools/tourapi/collect.py export`. 하루 한도를 **수집 스크립트와 서버 동기화 배치가 함께** 쓰므로 수집이 끝날 때까지 로컬 서버는 `BATCH_ENABLED=false` 권장. 서버는 `TOUR_IMPORT_CSV_DIR`를 지정하면 시작 시 CSV를 적재한다.
- 실내·실외 매핑 초안 `indoor_outdoor.csv`(소분류 246개 중 추천코스 6개 제외, 중분류 기준 + 이름 예외: 고궁 MIXED·동굴 INDOOR·전시/박람회 INDOOR 등)는 **Claude가 임의로 정한 것이라 사람의 검토 필요**.

## FE ↔ ERD 불일치 (P1 위주)
| 영역 | FE 기대 | 기존 ERD | 제안 |
|---|---|---|---|
| 여행 구조 | 여행 > 일정안 여러 개 > 하나 확정 | `schedules` 한 단계 | `trips`+`plans`, `confirmed_plan_id`, `schedule_items`→`plan_id`, 만든 방식 AI/AI_RETRY/MANUAL |
| 여행 지역·출발지 | 시·군·구 여러 곳, 출발지 | 없음 | `trip_regions`, `trips.origin_*` |
| 온보딩 | 스타일 7개 중 1~3개, 시간대 EARLY/MORNING/AFTERNOON, 이동 편의 토글 1개 | 시간대 MORNING/EVENING/ANY, 유모차·휠체어 2개 | `travel_styles`+lcls 매핑, `need_easy_mobility` 하나로 |
| 동의 | 위치·행동 데이터·이메일 알림 | AGE_14/TERMS/PRIVACY/BEHAVIOR_DATA | `LOCATION`, `EMAIL_NOTIFICATION` 추가 |
| 지역 코드 | 법정동 | 옛 `area_code` | `ldong_regn_cd VARCHAR(5)`, `ldong_signgu_cd` |
| 분류·실내외 | lcls 저장, 실내/실외/혼합 라벨 | `tour_cat_code` 하나 | `lcls_systm1/2/3`, 분류 마스터, `contents.io_type` |
| 목록 표시 | 오늘 운영 상태·쉬는 날·행사 시간·장소·빠른 칩 | 자유 텍스트 `operating_hours`뿐 | 운영 시간·쉬는 날·행사 시간 칼럼을 배치로 적재 |
| 추천 요청 | 시·군·구 여러 곳, 실내 위주 | `region_id` 하나 | `recommendation_request_regions`, `indoor_preferred` |
| 반려동물 | 장소 쪽 필터 | 장소 데이터 없음 | `contents.pet_allowed` (TourAPI `detailPetTour2`) |

- **정책 충돌**: PRD는 "현재 위치 좌표 저장 안 함"인데 ERD는 추천 요청의 출발지 좌표가 필수. 선호 시간대도 FE 안에서 온보딩 3개(09·10·13시) vs 추천 요청 4개(09·10·11·13시)로 다르다. 여행 스타일은 8개가 아니라 7개(문화·역사 통합).
- 구현 단계에서 바뀐 것: MySQL 예약어라 `accessible`→`accessible_only`, `content_details.fetched_at`은 NULL 허용(상세 새로고침 규칙은 "행 없음 OR fetched_at NULL OR 7일 경과").
- ERD에는 있지만 FE가 안 쓰는 것: 관심 카테고리(질문 삭제), `users.last_companion_type`, `event_program`·`host`·`homepage` 등 일부 상세 칼럼.

## 수집 데이터를 담는 테이블 구조 (PR #18, 2026-11-11 기준 develop 미반영)
- 당시 develop에는 `User`·`RefreshToken` 테이블뿐이고, 수집 데이터용 엔티티는 `17-feat-content-foundation` 브랜치(PR #18, 대상 브랜치가 develop이 아니라 `15-chore-api-common-base`)에 있다. PR #15·#18이 develop에 합쳐져야 CSV를 적재할 수 있다.
- **`contents`**: 장소·축제 1건=1행, PK `id`=TourAPI contentid. `content_type`(ATTRACTION/FESTIVAL/RESTAURANT/LODGING), `tour_content_type_id`(12 관광지·14 문화시설·15 축제·32 숙박·39 음식점), `sido_region_id`·`region_id`, `lcls1~3`, `category_id`, `io_type`(INDOOR/OUTDOOR/MIXED, lcls3별로 손수 지정, 비 오는 날 실내 추천용), 축제 `event_*`·`playtime_raw`, 운영 `operating_hours_raw`·`open_time`·`close_time`·`rest_date_raw`·`rest_days`·`pet_allowed`, `popularity_score`·`wish_count`·`is_deleted`.
- **`content_details`**(1:1, `@MapsId`): `overview`·`homepage`·`fee`·`contact`·`host`·`age_limit`·`event_program`·`fetched_at`. **`overview`(소개글)와 `content_images`(detailImage2)는 CSV에 없어** 상세 새로고침 배치가 돌 때까지 비어 있다 → AI 추천에서 소개글을 쓸 계획이면 주의.
- `regions`(depth 1 시·도 / 2 시·군·구, `parent_id`), `tour_classifications`(분류 코드 트리), `categories`(group_type PLACE_FILTER/FESTIVAL_FILTER/CHIP)·`category_mappings`, 그리고 `tour_sync_cursors`·`batch_job_logs`.
- **CSV 적재**: `collect.py`가 CSV 4개(contents 32,548행·content_details 950·lcls_codes 246·regions 284)를 만들고, `TOUR_IMPORT_CSV_DIR`를 주면 `TourCsvImporter`가 서버 기동 시 읽는다. 같은 키 한도(하루 1,000건)를 새벽 4시 `TourSyncJob`이 나눠 쓰므로 개발용 키로는 `BATCH_ENABLED=false`를 권한다. `content_details.event_start_time`은 TourAPI에 필드가 없어 항상 비고, 축제 시간은 `operating_hours`(→`playtime_raw`)에 있다. 데이터는 git에 올리지 않는다.
- **수집 데이터 분포 (2026-11-08 기준)**: `io_type` 실내 19,612·실외 10,495·혼합 2,441(비 오는 날 실내 추천용, lcls3별 수동 매핑). 썸네일은 82%에 있지만 **전화번호는 3%(874건)뿐**. 상세 소개 대상은 축제·관광지이고 **음식점·숙박은 수집 대상이 아니다**. 축제는 수집일 1년 전(2025-11-08)을 기준으로 조회해 시작일이 2022-11-01~2027-09-23로 퍼져 있다("최근 1년 시작"이 아님).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/JKYMHS-Backend/develop-2)
