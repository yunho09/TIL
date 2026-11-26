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

## 관련
- [[JKYMHS-Backend/프로젝트-현황]], [[MySQL-연동]]

## 출처
- Claude Code 세션 자동 캡처 (/work/code/project/JKYMHS-Backend)
