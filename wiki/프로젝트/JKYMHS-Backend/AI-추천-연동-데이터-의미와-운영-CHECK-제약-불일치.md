---
tags: [JKYMHS, AI추천, JPA, MySQL, 배치]
updated: 2026-11-26
---
# AI 추천 연동 데이터 의미와 운영 CHECK 제약 불일치

AI 담당의 질문과 지적을 운영 코드(`origin/develop`, 운영 jar와 같은 버전)와 운영 DB로 확인한 결과(2026-11-26, 읽기만 함). 컬럼의 실제 의미, 이미 구현된 연동 방식, 아직 고치지 않은 문제를 정리한다. 스키마 분리는 [[AI-스키마-분리와-view-데이터-계약]].

## 운영 DB 제약과 코드 enum 불일치 (미수정)
- 코드의 `ConsentType`에는 `LOCATION`이 있는데 운영 DB `user_consents`의 CHECK 제약에는 없다 → **위치 동의를 저장하면 DB 오류로 실패**한다. 회원 0명이라 아직 드러나지 않았다.
- 원인: `ddl-auto: update`는 컬럼은 추가해도 **이미 있는 CHECK 제약은 고치지 않는다**. 테이블이 `LOCATION` 추가 커밋(11/08)보다 먼저 만들어졌다.
- enum 값을 새로 추가할 때마다(예: `failure_reason`에 `BUSY`·`ABANDONED`) 운영 DB `ALTER TABLE`이 따로 필요하다.
- 운영 CHECK 제약 32개를 코드 enum과 전부 비교했고, 어긋난 것은 `LOCATION` 하나였다.
- 제안(미결정): Flyway 도입 → 현재 운영 스키마를 baseline으로 잡고 다음 버전에서 `LOCATION` 제약 수정.

## 데이터의 의미
| 항목 | 실제 동작 | 근거 |
|---|---|---|
| `contents.id` | TourAPI contentid 그대로(자동 증가 아님) | `Content` |
| `recommendation_requests.conditions_key` | 요청 조건 전체(지역, 날짜, 시작 시각, 출발지, 동행, 아이 연령, 실내, 필수 장소(순서 유지), 대상 여행, 이전 요청, 제외 목록(정렬))의 SHA-256. 같은 key의 PENDING이 10초 안에 또 오면 앞 요청 id 반환(FE 5xx 재시도 대비) | `RecommendationService.conditionsKey` |
| `excluded_content_ids` | 다시 추천 API의 `excludeIds`(사용자 제외). 쉼표 구분, 1000자 초과분은 잘림. 직전 코스 감점은 직전 결과에서 가져옴 | `RecommendationRequest` |
| `preference_profiles.profile_key` | `common` 또는 동행 코드 소문자(`alone/couple/friends/kids/parents`). 행동이 들어올 때마다 최근 180일치 재계산(반감기 30일). 동행 프로필 행동 10개 미만이면 `common` 사용. 정기 배치 없음 | `PreferenceProfileService` |
| `behavior_events.weight` | VIEW 1, WISH 3, SCHEDULE_SAVE 5, CALENDAR 5, VISIT_SATISFIED 10, VISIT_NEUTRAL 5, VISIT_UNSATISFIED −3, AI_REMOVE −3 | `BehaviorSignalType` |
| `popularity_score` | 매시 정각(KST). 원점수 = 최근 30일 조회×1 + 새 찜×3 + 일정 담기×5, 점수 = `round(1000 × ln(1+원점수) / ln(1+최대))`. 0~1000 정수 | `PopularityJob`, `PopularityScores` |
| `rest_days` | `"MON,TUE"` / `"NONE"`(휴무 없음) / `NULL`(모름) | `RestDays` |
| `open_time`·`close_time`·`last_entry_time` | 원문의 첫 `HH:mm~HH:mm`(날짜별로 다르면 첫날), close가 더 이르면 자정 넘김. 마지막 입장은 "입장마감/매표마감 HH:mm". 해석 실패는 NULL이고 원문은 `operating_hours_raw`·`rest_date_raw`. 축제 시간은 `event_start_time`·`event_end_time` | `OperatingHoursParser` |
| `io_type` | 장소별 판단이 아니라 소분류 `lcls3` → `tour_classifications.io_type` | `ContentService` |
| `contents.updated_at` | 동기화 때마다 **값이 안 바뀌어도** 갱신(인기 점수·찜 수 변경 때는 안 바뀜). TourAPI 변경 감지는 `tour_modified_at` | `ContentService` |
| `intro_fetched_at`·`content_details.fetched_at` | 운영 정보는 30일마다 다시 받아 내용이 같아도 갱신 → 원문 해시로 한 번 더 거른다. `fetched_at`은 상세 수신 시각(새 행 NULL) | `ContentService` |
| 콘텐츠 삭제 | 행을 지우지 않고 `is_deleted=1`(다시 보이면 0). `content_images`만 지우고 다시 넣어 id가 바뀜 | `ContentService` |
| `request_date` | KST. FAILED는 횟수에서 제외, 하루 10회 | `ClockConfig`, `application.yml` |
| `user_preferences.accessible_only` | 유모차·휠체어·오래 걷기 어려움을 묶은 체크박스. 추천 로직·AI 요청에 안 쓰임(화면 문구는 FE 확인 필요) | `UserPreference` |

## 이미 구현돼 있는 연동 방식
- Spring Boot 4.1.1, Java 21. 인스턴스 1대라 ShedLock은 아직 불필요.
- 요청 접수 → 202 + requestId → FE polling. 스레드 풀 4~8, 대기열 50(가득 차면 503), 작업 제한 25초.
- 추천 결과는 DB가 아니라 **Redis에 24시간**, 진행 단계도 Redis. `completed_at` 컬럼은 없다.
- AI 호출: `ai.provider` 기본 `mock`, `http`면 `AI_BASE_URL`을 `Authorization: Bearer AI_API_KEY`로 호출. 요청 계약은 `AiCourseRequest`.
- 매시 30분 정리 배치가 24시간 지난 요청의 `origin_lat`·`origin_lng`·`origin_address`를 NULL로 바꾼다.
- 동의 철회 시 `behavior_events`·`preference_profiles`를 같은 트랜잭션에서 삭제, 탈퇴 시 `content_views`·`content_save_logs`까지.
- 음식점·숙박 반경 검색은 `ContentNearbyService`(없으면 반경 2배, 최대 5km, `ST_Distance_Sphere`, 공간 인덱스 없이 `content_type, latitude, longitude` 인덱스).

## 설계 문제와 판단 (수정 안 함)
### 인기 점수가 전부 0
- 동점이면 `ORDER BY popularity_score DESC, id`라 임의 순서도 아니고 **지역마다 contentid가 작은 200곳만** 계속 후보 풀에 들어간다. 취향이 있어도 나머지는 후보가 될 수 없다.
- AI 담당의 제안: 관광공사 방문 통계·상세 정보 유무 같은 대체 초기 점수, 또는 임베딩 재정렬.

### 기본 코스(FALLBACK)도 횟수 차감
- 코드 주석상 의도된 동작("결과를 받았으니 센다"). AI 담당의 PRD는 차감하지 않음이라 정책 결정이 필요하다. 기본 코스는 AI 비용이 없다.

### 멈춘 PENDING 정리: 타임아웃만 줄이면 안 된다
- `pending-timeout`은 10분이지만 배치가 **매시 30분 한 번**이라 실제로는 10~70분 남는다. 값만 2분으로 줄여도 효과가 거의 없다.
- 25초 제한은 요청 시각이 아니라 **스레드가 작업을 시작한 시각**부터 잰다. 대기열 끝 요청은 시작까지 최대 3분쯤 기다릴 수 있어, 2분 기준은 정상 요청을 실패시킬 수 있다.
- 실제로 멈추는 경우는 작업 중 서버 재시작.
- 대안: ① 배치를 1분마다 + 기준 5분 ② 작업 시작 때 요청 후 30초(FE 대기 시간)가 지났으면 바로 FAILED.

### 선호 점수 시간 감소: 읽을 때 반감기를 적용해도 결과가 같다
- 한 프로필의 모든 행은 같은 시각에 계산되므로 나중에 감소를 적용해도 **같은 배수**만 곱해진다.
- 후보 점수 계산은 프로필 점수를 **회원별 최댓값으로 나눠** 그 배수가 사라진다. AI에 보내는 `topCategories` 순서도 그대로(절댓값만 큼).
- "오래 쉰 회원은 취향보다 인기를 더 반영"이 목적이면, 마지막 활동 시점·행동 수로 프로필 가중치(0.4)를 줄이는 **신뢰도 배수**가 필요하다.

### AI의 DB 접근
- AI 담당 제안: AI 배치가 읽는 열만 모은 view(`ai_content_source`)를 Spring이 관리하고 AI 계정은 그 view만 읽는다.
- `ddl-auto`로는 view를 관리할 수 없어 **Flyway 도입이 먼저**다.
- 콘텐츠 테이블에는 민감 정보가 없지만 테이블 직접 SELECT는 AI 코드를 테이블 구조에 묶는다.

## 상충하는 정보
- [[JKYMHS-Backend/생-컨테이너-직접-배포-systemd-자동배포-프록시-IP]]는 사용자가 없을 때 인기순 목록 맨 앞이 "임의 관광지"로 보인다고 적었다. 이번 확인으로는 동점 정렬 기준이 `id`다(추천 후보 풀 쿼리 기준).

## 출처
- Claude Code 세션 자동 캡처 (/work/code/project/JKYMHS-Backend)
