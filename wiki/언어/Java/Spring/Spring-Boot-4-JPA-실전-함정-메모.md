---
tags: [Spring, SpringBoot4, JPA, Hibernate, Jackson3, 트랜잭션]
updated: 2026-10-27
---

# Spring Boot 4 / JPA 실전 함정 메모 (bium-be)

Spring Boot 4.1.1 · Spring 7 · Hibernate 7.4 · Jackson 3 환경에서 Gemini 연동과 레시피 저장을 만들며 부딪힌 것들. 3줄 요약: Boot 4에선 `RestClient.Builder` 자동 빈과 일부 테스트 패키지가 바뀌었고, H2의 JSON 컬럼은 `@JdbcTypeCode(JSON)`으로 매핑해야 하며, 외부 AI 호출은 `TransactionTemplate`로 저장 구간만 감싸 트랜잭션 밖에서 해야 한다.

## Boot 4 / Spring 7 변경점
- `RestClient.Builder`는 자동 구성되지 않음(`spring-boot-starter-restclient` 필요) → 컨텍스트 로딩 실패. 코드에서 `RestClient.builder()`를 직접 사용(타임아웃은 `SimpleClientHttpRequestFactory`).
- Jackson 3 패키지는 `tools.jackson.databind.*`, `JsonMapper.shared()`. 노드 메서드는 `asString()`/`stringValue()`/`isString()`.
- `HttpStatus.UNPROCESSABLE_CONTENT`(422) 사용.
- 테스트 어노테이션 위치: `org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc`, `@MockitoBean`은 `org.springframework.test.context.bean.override.mockito`.
- 설정 값은 `@ConfigurationProperties` record(`prefix="gemini"`) + `@EnableConfigurationProperties`, 키는 `${GEMINI_API_KEY:}`로 환경변수에서만 읽음. `spring.config.import: optional:file:.env[.properties]`로 `.env`도 읽히며(견본은 `.env.example`에 키 이름만 추가), IntelliJ 실행 구성에 따로 넣을 필요 없음.

## H2 JSON 컬럼에 문자열을 넣으면 따옴표가 한 번 더 붙음
- `String`에 `columnDefinition = "json"`으로 JSON 문자열을 넣으면 H2 테스트 DB에서 **JSON 문자열 값으로 한 번 더 감싸져** 저장된다.
- 해결: `@JdbcTypeCode(SqlTypes.JSON) List<String> steps`처럼 Hibernate JSON 매핑을 사용 → H2와 MySQL 모두 정상(MySQL에선 `json` 컬럼). 상세 조회 테스트는 `em.clear()` 후 다시 읽어 검증.
- 길이 제한이 있는 컬럼은 AI 출력을 **코드포인트 기준**으로 잘라 넣음(MySQL utf8mb4 `VARCHAR(n)`도 코드포인트 단위라 일치). 이모지 컬럼은 DB가 `utf8mb4`여야 하며 `utf8mb3`면 삽입이 500. H2는 UTF-16 단위로 세서 보조 문자가 많으면 테스트에서만 실패할 수 있음.
- `@OrderColumn`이 있는 `List` 컬렉션은 `MultipleBagFetchException`/bag 재생성 문제를 피함.

## 외부 API 호출은 트랜잭션 밖에서
- Gemini 응답이 ~9초(읽기 타임아웃 30초)라 서비스 메서드에 `@Transactional`을 걸면 DB 연결을 그만큼 붙잡아 동시 요청 시 커넥션이 마른다.
- 패턴: `recommend()`는 비트랜잭션으로 두고 **저장 구간만 `TransactionTemplate.execute`**. 조회(`getRecipe`)는 `@Transactional(readOnly = true)`. `getReferenceById`로 fridge·user 프록시를 가져와 저장(open-in-view off에서 `getFridge().getId()`는 프록시에서 바로 읽혀 지연 로딩 문제 없음).
- 테스트 함정: 테스트 클래스가 `@Transactional`이면 `TransactionTemplate`이 테스트 트랜잭션에 합류해 이 경계를 검증하지 못함. 비트랜잭션 `@SpringBootTest`를 따로 두고 Gemini 스텁 안에서 `TransactionSynchronizationManager.isActualTransactionActive()`가 false인지 단언, JPQL로 직접 정리.
- 시간대: `@CreationTimestamp`의 `LocalDateTime`은 JVM 기본 시간대라 UTC 서버면 UTC 벽시계 시각이 KST로 오해됨. `Clock`(Asia/Seoul)을 빈으로 주입해 D-day·시각을 계산하거나 `Instant`/`OffsetDateTime`, `hibernate.jdbc.time_zone` 사용. `Clock` 주입은 테스트(MutableClock)에도 유리.

## 테스트·구조 팁
- 여러 JDK 환경에서 로컬 JDK 17이 없을 때 임시로 JDK 21 + `--release 17` 컴파일 init 스크립트(`afterEvaluate`로 `build.gradle`보다 늦게 적용)로 검증할 수 있으나, 정식 해결은 JDK 17 설치 → [[Gradle-JDK-17-못-찾음-Homebrew-libexec-경로]].
- `Recipe` 엔티티를 만들기 전엔 추천 로직을 `RecommendationIngredient` 같은 평범한 record 입력/`RecommendedRecipe` 출력으로 두면 엔티티(#4) 머지 전에도 완결되게 만들 수 있다.
- 서비스가 컨트롤러 패키지와 같은 위치에서 테스트하기 쉽도록 서비스 테스트는 서비스 클래스와 같은 패키지에 둔다.

## 관련
[[레시피-추천-API-Gemini-서버-이전-설계]], [[JPA-엔티티와-리포지토리]], [[ERD-to-JPA-엔티티-변환-팁]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/bium-be)
