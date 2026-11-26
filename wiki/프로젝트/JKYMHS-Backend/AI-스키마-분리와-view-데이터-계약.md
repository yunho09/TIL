---
tags: [MySQL, 스키마, view, Docker, JKYMHS]
updated: 2026-11-26
---
# AI 스키마 분리와 view 데이터 계약

Spring Boot용 `jkymhs`와 AI용 `jkymhs_ai`로 스키마를 나누는 방법. 팀 논의는 지금 **스키마·계정 분리까지만** 하자는 단계이고, view(AI가 읽는 데이터 형식의 계약)는 AI가 메인 데이터를 필요로 할 때 얹는다. 아래 SQL은 설명용 예시로, 실행해 검증하지 않았다.

## 구조
- MySQL에서 **schema = database**. 스키마를 나눈다는 건 같은 MySQL 서버에 DB를 두 개 두는 것이다.
- 로컬에서는 컨테이너가 늘지 않는다. `jkymhs-mysql` 컨테이너 1개 안에 MySQL 서버 1개(3306)와 DB 2개가 있고, `jkymhs-redis`는 그대로다.
- Spring Boot는 컨테이너가 아니라 내 PC의 JVM 프로세스(IDE·`./gradlew bootRun`)로 `localhost:3306`에 접속한다(레포에 `Dockerfile` 없음, compose는 MySQL·Redis만).
- 같은 서버라서 `jkymhs_ai`의 view가 `jkymhs.users`를 바로 참조할 수 있다. **DB를 다른 MySQL 컨테이너로 나누면 view로 서로 참조할 수 없다.**

## 지금 하는 일 (스키마만 분리)
1. `jkymhs_ai` 스키마 생성
2. AI 전용 계정에 `jkymhs_ai.*` 권한만 부여. Spring Boot 계정 `jkymhs`는 그대로
3. Spring Boot 설정은 변경 없음. 연결 URL이 `/jkymhs`라 `ddl-auto`도 AI 스키마를 건드리지 않는다

### 로컬 compose
- `MYSQL_DATABASE`로는 DB를 **하나만** 만들 수 있어, 두 번째 DB·계정은 init 스크립트로 만든다.

```yaml
    volumes:
      - mysql-data:/var/lib/mysql
      - ./docker/mysql/init:/docker-entrypoint-initdb.d
```

```sql
-- docker/mysql/init/01-ai-schema.sql
CREATE SCHEMA IF NOT EXISTS jkymhs_ai
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'jkymhs_ai'@'%' IDENTIFIED BY 'jkymhs_ai';
GRANT ALL PRIVILEGES ON jkymhs_ai.* TO 'jkymhs_ai'@'%';
```

- init 스크립트는 **볼륨이 비어 있을 때만** 실행된다. 이미 띄운 팀원은 `docker compose down -v`(데이터 삭제됨) 후 다시 올리거나 SQL을 직접 한 번 실행한다.

### view를 붙이기 쉽게 미리 정할 것
- DB 이름(`jkymhs_ai`)을 팀에서 확정 — view가 이 이름을 참조한다
- collation 기본값을 `utf8mb4_unicode_ci`로 통일
- AI 결과가 회원과 연결되면 `user_id BIGINT`로 `users.id`와 타입을 맞추고, 스키마 간 FK는 걸지 않는 편이 낫다

## 운영 서버에서 나눌 때 (2026-11-26, 인스턴스 접속 확인)
- 운영은 인스턴스 하나에 Spring Boot(`jkymhs.service`)·MySQL 8.4·Redis가 함께 있다([[JKYMHS-Backend/생-컨테이너-직접-배포-systemd-자동배포-프록시-IP]]). 확인 시점의 DB는 `jkymhs` 하나(테이블 31개)이고, **`jkymhs_ai`는 아직 만들지 않았다**(팀원과 이야기한 뒤 진행).
- 기존 계정 `jkymhs`의 권한은 `USAGE ON *.*`(로그인만) + `ALL PRIVILEGES ON jkymhs.*`라, 새 스키마를 만들어도 **Spring Boot 계정에는 보이지 않는다**. 손댈 것 없음.
- 서버 root가 있으면 `mysql`로 바로 들어가 만들 수 있어 PaaS 운영진 요청은 필요 없다. 아래 SQL은 **실행하지 않았다**.

```sql
CREATE SCHEMA jkymhs_ai
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
CREATE USER 'jkymhs_ai'@'localhost' IDENTIFIED BY '<새 비밀번호>';
CREATE USER 'jkymhs_ai'@'127.0.0.1' IDENTIFIED BY '<새 비밀번호>';
GRANT ALL PRIVILEGES ON jkymhs_ai.* TO 'jkymhs_ai'@'localhost';
GRANT ALL PRIVILEGES ON jkymhs_ai.* TO 'jkymhs_ai'@'127.0.0.1';
SHOW GRANTS FOR 'jkymhs_ai'@'localhost';
```

- 계정을 `localhost`·`127.0.0.1` 둘 다 만드는 것은 기존 `jkymhs` 계정과 맞추기 위해서다.

### AI 전용 계정을 따로 두는 이유
- 필수는 아니다(Spring 계정을 같이 써도 동작). 다만 같은 계정이면 AI 서버가 `users`·`refresh_tokens`를 읽고 지울 수 있어 **스키마를 나눈 의미가 없다**.
- AI 코드 버그·마이그레이션 실수·LLM이 만든 SQL 실행이 `jkymhs_ai` 안에서 끝나고, AI 설정 파일이 유출돼도 메인 DB에 못 들어간다. 나중에 view만 읽도록 권한을 좁히기도 쉽다.
- 이 분리는 **프로그램의 계정**을 막는 것이다. 사람에게 서버 root를 주면 그 사람 기준으로는 의미가 없다.

### 역할 분담과 AI 서버 위치
- 처음 한 번(서버 root 보유자): 빈 스키마·AI 계정, 이후 `jkymhs-ai.service`와 API 키용 환경변수 파일. 계속(AI 담당): 서버 코드, `jkymhs_ai` 안의 테이블. 코드 업데이트 배포 담당은 미정.
- **지금 필요한 것은 빈 DB와 계정뿐**, 서버 작업은 AI 코드가 나온 뒤. 그 전까지 AI 담당은 로컬 Docker MySQL에 같은 스키마를 만들어 개발한다.
- MySQL이 `127.0.0.1`에서만 받으므로, AI 서버가 이 DB를 쓰려면 **같은 인스턴스에서 돌아야 한다**(아니면 MySQL을 외부에 열어야 함).
- AI는 외부 LLM API 키를 쓰는 방식 → 확인 시점 가용 메모리 약 2.9GB(2코어, GPU 없음)로 충분. Spring이 `127.0.0.1:8000`(예정)으로 호출하고 AI 서버는 외부에 열지 않는다. API 키는 채팅·레포가 아니라 서버 환경변수 파일에 직접 넣는다.

## 운영 DB 테이블 구조 (2026-11-26, DDL만 조회)
31개 테이블, 288개 컬럼. 데이터 현황은 확인 시점 기준이다.

| 도메인 | 테이블 | 데이터 |
|---|---|---|
| 회원·인증 | `users`, `refresh_tokens`, `user_consents`, `user_preferences`, `user_travel_styles` | 회원 0명 |
| 관광 콘텐츠 | `contents`, `content_details`, `content_images`, `categories`, `category_mappings`, `regions`, `tour_classifications`, `travel_style_mappings` | `contents` 약 3.2만, `regions` 284 |
| 여행·일정 | `trips`, `trip_regions`, `plans`, `plan_items`, `visit_reviews` | 비어 있음 |
| AI 추천 | `recommendation_requests`, `recommendation_request_regions`, `recommendation_required_places`, `preference_profiles` | 비어 있음 |
| 사용자 행동 | `behavior_events`, `content_views`, `content_save_logs`, `wishlists` | 비어 있음 |
| 구글 캘린더 | `google_calendar_links`, `calendar_sync_jobs`, `calendar_sync_job_items` | 비어 있음 |
| 배치 | `batch_job_logs`, `tour_sync_cursors` | 배치 기록 220 |

- **AI 관련 데이터가 이미 메인 DB에 있다**: `recommendation_requests`, `preference_profiles`, `behavior_events`, `plans.source`(AI/AI_RETRY/MANUAL), `plan_items.ai_reason`. `jkymhs_ai`로 옮길지 Spring이 계속 관리할지 AI 담당과 먼저 정해야 같은 데이터가 양쪽에 생기지 않는다(미정).
- **AI에 넘기면 안 되는 컬럼**: `users`의 `email`·`password_hash`·`google_sub`·`google_email`, `refresh_tokens.token_hash`, 캘린더 테이블의 `encrypted_refresh_token`. view에서 뺀다.
- **FK 없이 id 값만 있는 컬럼이 많다**: `content_views.content_id`, `behavior_events.content_id`, `trips.confirmed_plan_id` 등. 이 연결 관계는 컬럼 이름에서 추정한 것이다.
- **운영 서버 코드는 작업 브랜치보다 새 것**: 브랜치 `2-feat-user-security-jwt`의 엔티티는 `users`·`refresh_tokens` 정도뿐이다. 테이블 구조는 로컬 브랜치 코드가 아니라 운영 DB나 최신 브랜치로 확인한다.
- 공유용 페이지: https://claude.ai/artifact/CP5ibKGfaVnBFHWEhHYtis (생성 시 비공개, 팀원이 보려면 Share에서 범위 변경). 원본 DDL은 `/home/yunho/.config/orca/chat-scratch/aa3368d8fc361a24c02c9465217e57c3/jkymhs-prod-schema-2026-11-26.sql`(채팅용 임시 폴더라 오래 남는다는 보장은 없음 — Claude 보충).

## 나중 단계: view = 외부 스키마
- AI는 Spring Boot 테이블을 직접 보지 않고 AI 스키마의 view만 본다. 3단계 스키마 구조(ANSI-SPARC)의 **외부 스키마**를 view로 구현하는 방식으로, 테이블이 바뀌어도 view 정의만 고치면 AI가 받는 모양이 유지된다.
- 메인 데이터가 바로 필요하다면 원본 테이블 SELECT 권한을 주지 말고 처음부터 view를 몇 개 만든다. 직접 권한은 AI 코드를 엔티티 테이블 구조에 묶는다.

| 불일치 종류 | view에서 하는 일 | 예 |
|---|---|---|
| 이름 | 컬럼 별칭 | `onboarding_status` → `stage` |
| 타입·값 | enum·날짜·NULL을 정한 형식으로 변환 | enum 문자열 → 코드값, UTC ISO 문자열 |
| 민감 정보 | 컬럼 제외 | `password_hash`, `google_sub`, `email` |
| 파생값 | 계산 컬럼 | 가입 후 지난 일수, 구글 연동 여부 |

```sql
CREATE
  SQL SECURITY DEFINER
VIEW jkymhs_ai.v_user_profile_v1 AS
SELECT
  u.id                      AS user_id,
  CASE u.onboarding_status WHEN 'DONE' THEN 'done' ELSE 'pending' END AS stage,
  (u.google_sub IS NOT NULL) AS has_google,
  DATE_FORMAT(CONVERT_TZ(u.created_at, '+09:00', '+00:00'),
              '%Y-%m-%dT%H:%i:%sZ') AS created_at_utc
FROM jkymhs.users u;
```

- (Claude 보충) 세션의 예시는 `'COMPLETED'`였으나, 위키에 기록된 실제 enum은 `NOT_STARTED`/`SKIPPED`/`DONE`([[JKYMHS-Backend/프로젝트-현황]])이라 `'DONE'`으로 적었다. 컬럼 이름도 실제 테이블과 대조한다.
- `SQL SECURITY DEFINER`: view를 만든 계정 권한으로 원본을 읽으므로, AI 계정은 `jkymhs.*` 권한 없이 정해진 컬럼만 본다.
- 권한: view는 `SELECT`만, AI 결과 테이블(`ai_result`)만 `INSERT, UPDATE, SELECT`.
- (Claude 보충) 위 init 스크립트의 `ALL PRIVILEGES ON jkymhs_ai.*`를 둔 채 view를 추가하면 AI 계정이 view 자체를 바꾸거나 지울 수 있다. view를 도입할 때 권한을 좁힌다.

## view를 쓸 때 함정
- **`ddl-auto: update`와 조용한 어긋남** (가장 위험): 엔티티 필드 이름을 바꾸면 Hibernate는 새 컬럼을 추가하고 옛 컬럼을 남겨, view가 **에러 없이 갱신 멈춘 옛 값**을 읽는다. 컬럼을 직접 지우면 조회 때 `ERROR 1356 (view references invalid column)`.
  - 권장: Flyway로 테이블·view DDL을 함께 버전 관리, 운영은 `validate`, Testcontainers에서 view마다 `SELECT ... LIMIT 1` 테스트
- **view는 데이터를 복사하지 않는다**: 조회 순간 앱 DB에서 쿼리가 돈다. 학습 데이터 추출 같은 대량 조회는 API 성능에 영향. MySQL에는 materialized view가 없어 read replica나 배치 복사(ETL)를 쓴다.
- **방향은 한쪽**: view는 읽기 계약, AI 결과는 `jkymhs_ai`의 실제 테이블에 저장. Spring이 읽어야 하면 반대 방향 view(`jkymhs` → `jkymhs_ai`)를 따로 만든다.
- **타임존·collation도 계약**: `DATETIME`에는 타임존이 없고 연결은 `Asia/Seoul`. `email`은 `utf8mb4_bin`이라 기본 collation 컬럼과 조인하면 `Illegal mix of collations`가 날 수 있다.
- **이름에 버전**: 형식이 바뀌면 `_v2`를 새로 만들고 AI가 옮겨 간 뒤 v1 삭제 → 따로 배포 가능.
- **DEFINER 계정 삭제 시 view가 깨진다**: 사람 계정 말고 전용 계정(`view_owner` 등)으로 만든다.

## 상충하는 정보
- [[JKYMHS-Backend/생-컨테이너-직접-배포-systemd-자동배포-프록시-IP]]는 AI 추천을 "Spring이 후보를 요청에 담아 보내므로 AI 서버는 DB가 필요 없다"(PR #30 기준)고 정리했다. 이번 논의는 AI용 DB를 따로 두는 쪽이다. AI가 어떤 데이터를 언제부터 읽을지는 미정.
- 이번 세션은 "배포 방식이 저장소에 정해져 있지 않다"고 답했지만, 위 페이지에는 생 컨테이너 직접 배포(MySQL apt 설치, Docker 아님)가 기록돼 있다. (Claude 보충) 그 서버에는 compose init 스크립트가 적용되지 않으므로 스키마·계정 SQL을 직접 실행해야 한다.
- **(2026-11-26 확인)** 같은 날 뒤이어 인스턴스에 접속해 직접 배포 구조가 맞음을 확인했다. 앞서의 "DB는 앱 컨테이너 밖에 있다"는 설명은 틀린 추측이었다.
- 새 스키마의 collation: 로컬 compose 예시는 `utf8mb4_unicode_ci`, 운영 서버 기본값은 `utf8mb4_0900_ai_ci`라 운영 SQL은 후자로 적었다. 로컬·운영을 하나로 맞출지는 미정(위 "미리 정할 것"의 `unicode_ci` 통일안과 다름).

## 관련
- [[JKYMHS-Backend/프로젝트-현황]], [[MySQL-연동]]

## 출처
- Claude Code 세션 자동 캡처 (/work/code/project/JKYMHS-Backend)
