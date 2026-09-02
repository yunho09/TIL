---
tags:
  - spring
  - mysql
  - docker
  - jpa
updated: 2026-09-02
---

# MySQL 연동

> 메모리(Map) 저장 → MySQL로 교체. **앱을 껐다 켜도 데이터가 남는다.**
> 교체 작업이 Repository 계층 안에서 끝나고 Service/Controller는 안 바뀌었다 — 계층을 나눈 보람.

## 필요한 것 3가지

**1. 의존성** ([[Gradle|build.gradle]])
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
runtimeOnly 'com.mysql:mysql-connector-j'   // 실행 때만 쓰는 드라이버
```

**2. 접속 정보** (application.yaml)
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/spring_practice
    username: spring
    password: spring1234
  jpa:
    hibernate:
      ddl-auto: update   # 엔티티 보고 테이블 자동 생성/수정 (개발용)
    show-sql: true       # 실행되는 SQL 콘솔 출력 (학습용 창문)
```

url 해부: `jdbc:`(자바 DB 표준) + `mysql:`(종류→드라이버 선택) + `//localhost:3306`(위치) + `/spring_practice`(DB 이름)

**3. MySQL 서버** — Docker 컨테이너로 (설치 없이)

## 자동 설정과의 연결

[[스프링-컨테이너|자동 설정]]이 build.gradle에서 data-jpa를 발견 → "DB 쓰는구나" → yaml의 접속 정보로 DataSource 빈 생성.

> [!warning] 겪은 에러
> `Failed to configure a DataSource: 'url' attribute is not specified`
> = 의존성(data-jpa)은 있는데 접속 정보가 없을 때. 의존성을 주석 처리하거나, yaml에 datasource를 채우면 해결.
> **에러 자체가 자동 설정이 동작한다는 증거.**

## Docker 명령어 모음

```bash
# 최초 생성 (이미 했음)
docker run -d --name spring-practice-mysql \
  -e MYSQL_ROOT_PASSWORD=root1234 \
  -e MYSQL_DATABASE=spring_practice \
  -e MYSQL_USER=spring -e MYSQL_PASSWORD=spring1234 \
  -p 3306:3306 mysql:8

# 재부팅 후 다시 켜기
docker start spring-practice-mysql

# DB 직접 구경 (한글 깨지면 charset 옵션 필수)
docker exec spring-practice-mysql mysql -uspring -pspring1234 \
  --default-character-set=utf8mb4 spring_practice -e "SELECT * FROM post;"
```

앱과 DB는 `localhost:3306` 네트워크로만 연결. 앱을 꺼도, 컨테이너를 꺼도 데이터는 디스크에 남는다.

## ddl-auto 선택지

| 값 | 동작 | 용도 |
| --- | --- | --- |
| `update` | 없으면 만들고, 필드 늘면 컬럼 추가 | 개발/학습 (지금) |
| `create` | 매번 새로 — **데이터 날아감** | 실험 |
| `none` | 아무것도 안 함 | 운영 기본 |

## 확인 문제

> [!question]- Q1. CREATE TABLE을 안 쳤는데 post 테이블이 생긴 이유는?
> `ddl-auto: update`가 `@Entity` 클래스([[JPA-엔티티와-리포지토리]])를 읽고 시작 시 자동 생성.

> [!question]- Q2. 메모리 버전과 달리 재시작해도 데이터가 남는 이유는?
> 상태의 거처가 빈의 필드(메모리)가 아니라 MySQL(디스크)로 옮겨갔기 때문. "빈에 상태를 두지 말라"던 규칙이 이제 지켜짐 — 상태는 DB가 담당.

## 다음에 볼 것

- [ ] 비밀번호를 yaml에 평문으로 안 두는 법 (환경변수)
- [ ] 운영에서 ddl-auto 대신 쓰는 것 (마이그레이션 도구)
