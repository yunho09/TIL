---
tags: [Spring, JPA, Auditing, 테스트, JKYMHS-Backend]
updated: 2026-11-11
---
# JPA Auditing 동작 원리와 `@EnableJpaAuditing`을 별도 클래스로 빼는 이유

`BaseTimeEntity`의 어노테이션은 "시각을 넣을 칸" 표시일 뿐이고, 실제로 넣는 담당자(`AuditingHandler`)는 `@EnableJpaAuditing`이 만들어 연결해야 생긴다. 빠져도 에러 없이 `NULL`이 저장된다. 메인 클래스에 붙이면 `@WebMvcTest`가 깨져서 별도 `@Configuration`으로 뺀다(Boot 4.1.1에서 실험으로 확인). 기본 구성은 [[스프링-CRUD-계층구조]]·[[ERD-to-JPA-엔티티-변환-팁]] 참고.

## 세 가지가 모두 있어야 동작
- `@EntityListeners(AuditingEntityListener.class)` — 저장/수정 직전(`@PrePersist`/`@PreUpdate`)에 호출될 **리스너 등록** (엔티티 쪽, `BaseTimeEntity`)
- `@CreatedDate`/`@LastModifiedDate` — 리스너가 값을 넣을 **칸 표시**
- `@EnableJpaAuditing` — 리스너가 일을 넘길 **담당자 생성·연결** (설정 클래스 쪽)
- 하나라도 빠지면 컴파일·실행 모두 통과하고 `created_at`/`updated_at`이 조용히 `NULL`로 저장된다.

## 왜 `@EnableJpaAuditing`이 필요한가 (spring-data-jpa 4.1.1 소스 기준)
- `AuditingEntityListener`는 `AuditingHandler handler` 필드를 갖고, 처음엔 `null`이다. `touchForCreate`/`touchForUpdate`는 `if (handler != null) handler.markCreated/markModified(target)`이라 **handler가 없으면 아무 일도 안 하고 통과**한다.
- `@EnableJpaAuditing`은 `@Import(JpaAuditingRegistrar.class)`가 붙은 표시일 뿐이고, 그 Registrar가 ① `jpaAuditingHandler` 빈 등록 ② 리스너의 `auditingHandler` 자리에 그 빈 주입을 한다.
- `@Configuration`은 스프링이 시작할 때 이 클래스를 설정으로 읽게 하는 표시(없으면 `@EnableJpaAuditing`도 실행 안 됨). 컴포넌트 스캔 범위(`BackendApplication` 패키지 하위)에 있으면 따로 등록할 필요 없다.
- 옵션 4개(전부 기본값이면 괄호 생략): `setDates=true`, `modifyOnCreate=true`(처음 저장할 때 `updatedAt`도 채움 → **가입 직후 `created_at`과 `updated_at`이 같은 시각**), `dateTimeProviderRef`(기준 시계 교체, 테스트에서 시각 고정), `auditorAwareRef`(`@CreatedBy`/`@LastModifiedBy`용 현재 유저 제공).
- `@CreatedBy` 류는 ERD에 `created_by`가 없어 안 쓴다. 쓰려면 `AuditorAware` 빈이 필요.
- 대안: Hibernate `@CreationTimestamp`/`@UpdateTimestamp`(설정 클래스 불필요, Hibernate 전용), JPA 표준 `@PrePersist`/`@PreUpdate`(시각 넣는 코드를 직접 작성).

## 메인 클래스가 아니라 별도 파일에 두는 이유 (직접 실험)
- 프로젝트 사본 두 개에 같은 `@WebMvcTest`를 넣고 비교.

| 위치 | 결과 |
|---|---|
| A. `BackendApplication`(메인)에 `@EnableJpaAuditing` | ❌ 테스트 실패 |
| B. `JpaAuditingConfig`(별도 `@Configuration`) | ✅ 통과 |

- A의 에러: `Error creating bean with name 'jpaAuditingHandler' ← 'jpaMappingContext' ← IllegalArgumentException: JPA metamodel must not be empty`
- 원인: `@WebMvcTest`는 JPA·DB를 띄우지 않는데, 테스트도 메인 클래스를 기준으로 시작해 메인에 붙은 어노테이션을 **무조건 읽는다**. 그래서 JPA가 없는데 Auditing 핸들러를 만들다 실패. 별도 `@Configuration`은 슬라이스 테스트가 스캔하지 않아 안전. Spring Boot 문서 Testing → User Configuration and Slicing에서도 이 방식을 권장.
- 블로그 예제는 메인에 바로 붙이는 경우가 많으니 주의.

## `@Column(updatable = false)`가 왜 필요한가
- JPA의 기본 UPDATE는 변경된 컬럼만이 아니라 **전체 컬럼을 SET**한다(dirty checking이 UPDATE를 자동 생성).
- `@CreatedDate`는 **INSERT 때만** 값을 채운다. 요청 DTO로 `new User(id, email, …)`를 새로 만들어 저장하는 식으로 `createdAt`이 `null`인 객체를 저장하면 `created_at = NULL`로 덮여 **가입일이 사라지고 에러도 없다**.
- `updatable = false`면 UPDATE의 SET 목록에서 `created_at`이 빠진다. 자바 객체에서 값을 바꿔도 에러 없이 무시된다.
- JPA 쪽에서만 막는 것이라 DB 제약은 아니다(MySQL에서 직접 UPDATE하면 바뀜). `show-sql: true`로 UPDATE문에서 컬럼이 빠진 것을 확인할 수 있다(User 엔티티 단계에서 확인 예정이었음).
- 시각 컬럼이 한쪽만 있는 테이블(예: `refresh_tokens`·`wishlists`는 `created_at`만, `user_consents`는 `updated_at`만)은 `BaseTimeEntity`를 상속하면 ERD에 없는 컬럼이 생기므로 상속하지 않고 필요한 칸만 엔티티에 둔다.

## 어노테이션이 읽히는 곳 (import 패키지가 다른 이유)
| 읽는 도구 | 어노테이션 | 패키지 |
|---|---|---|
| Lombok(컴파일 때 코드 생성) | `@Getter` | `lombok` |
| JPA | `@MappedSuperclass`, `@EntityListeners`, `@Column` | `jakarta.persistence` (Boot 3+는 `javax`가 아님) |
| Spring Data | `@CreatedDate`, `@LastModifiedDate`, `@EnableJpaAuditing` | `org.springframework.data…` |
- `@Setter`는 일부러 안 붙인다(시각은 JPA만 넣어야 함). `@MappedSuperclass`가 없으면 자식 테이블에 컬럼이 안 생긴다.
- `@Configuration`은 후보가 여러 개 뜨므로 `org.springframework.context.annotation`을 고른다.
- `@RequiredArgsConstructor`는 enum 전용이 아니라 **초기값 없는 final 필드를 받는 생성자**를 만들어 주는 것이다. `ErrorCode`의 `INVALID_INPUT(HttpStatus.BAD_REQUEST, "…")`는 그 생성자 **호출**이고(생성자 없이 쓰면 `constructor ErrorCode … cannot be applied to given types; required: no arguments`), `javap`로 보면 `private ErrorCode(HttpStatus, String)`이 만들어져 있다(enum 생성자는 자동 `private`이라 값이 정해진 개수로 고정). enum은 **항목을 맨 위에** 쓰고 `;` 뒤에 필드를 쓴다. 같은 어노테이션이 Service·Controller의 생성자 주입(DI)에도 쓰인다 → [[빈과-DI]].

## 커밋 단위
- `BaseTimeEntity`만 커밋하면 시각이 안 채워지는 반쪽짜리라, `JpaAuditingConfig`와 **한 커밋**으로 묶었다(`chore: BaseTimeEntity와 JPA Auditing 설정 추가`). 기능 없는 기반 작업은 `feat`가 아니라 `chore`.
- `BaseTimeEntity` 단독 테스트는 가짜 엔티티가 필요해 로컬 DB에 쓰레기 테이블이 생기므로 `User` 엔티티를 만들 때 함께 테스트하기로 미뤘다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
