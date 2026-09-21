---
tags: [spring, jpa, entity, erd]
updated: 2026-09-22
---

# ERD → JPA 엔티티 변환 팁

ERD를 보고 Spring Boot + JPA 엔티티를 짤 때의 변환 순서, 연관관계 원칙, ERD 표기·타입 매핑, 자주 터지는 함정 정리. 기본 엔티티 규칙은 [[JPA-엔티티와-리포지토리]], 폴더 배치는 [[도메인형-패키지-구조]] 참고. (세션에서 Claude가 ERD를 직접 열어보기 전에 준 일반 가이드이며 특정 ERD에 대한 결론은 아니다.)

## 변환 순서를 고정한다
1. **테이블 → `@Entity`** 전부 껍데기만 (id만)
2. **컬럼 → 필드** 채우기
3. **FK → 연관관계** 마지막에 연결
- 관계부터 손대면 아직 없는 클래스를 참조해 컴파일 에러가 연쇄한다. 껍데기 → 필드 → 관계 순이 덜 막힌다.

## 연관관계 3원칙
- **단방향이 기본.** 양방향은 반대쪽에서 조회할 일이 실제로 있을 때만. ERD에 선이 있다고 다 양방향으로 만들지 않는다.
- **`@ManyToOne`/`@OneToOne`은 `fetch = LAZY` 명시.** 기본값이 EAGER라 안 적으면 **N+1**의 씨앗.
- **연관관계 주인 = FK를 가진 쪽.** 반대편은 `mappedBy`. 양쪽을 다 주인처럼 쓰면 UPDATE 쿼리가 따로 나간다.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id")
private User user;
```

## ERD 표기 → 코드
| ERD | 코드 |
| --- | --- |
| 1:N | N쪽에 `@ManyToOne` (여기가 주인) |
| 1:1 | FK 있는 쪽에 `@OneToOne` |
| N:M | **중간 엔티티로 푼다.** `@ManyToMany` 쓰지 않기 |
| 식별관계(점선) | 보통 복합키 → 가능하면 대리키(id)를 추가해 회피 |

- `@ManyToMany`는 나중에 "주문수량" 같은 컬럼이 중간 테이블에 추가되는 순간 구조를 갈아엎어야 한다 → 처음부터 `OrderItem` 같은 엔티티로 분리.

## 타입 함정
- `VARCHAR` → `String` + `@Column(length = 30, nullable = false)` — ERD의 길이·NOT NULL을 옮겨야 DDL 자동 생성에 반영된다
- 돈 `DECIMAL` → `BigDecimal` (`double` 금지)
- `DATETIME` → `LocalDateTime`, `DATE` → `LocalDate`
- ENUM 컬럼 → `@Enumerated(EnumType.STRING)` **필수**. ORDINAL은 enum 순서가 바뀌면 기존 데이터가 통째로 밀린다
- `BOOLEAN` → `boolean` (null 가능하면 `Boolean`)

## 공통 컬럼은 베이스 클래스로
`created_at`/`updated_at`이 여러 테이블에 있으면 `@MappedSuperclass` + `AuditingEntityListener`.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {
    @CreatedDate private LocalDateTime createdAt;
    @LastModifiedDate private LocalDateTime updatedAt;
}
```
- 메인 클래스에 **`@EnableJpaAuditing`** 을 빼먹으면 값이 안 채워진다.

## 엔티티 기본 뼈대
```java
@Entity
@Table(name = "users")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User extends BaseTimeEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```
- **`@Setter` 금지** — 값이 어디서 바뀌는지 추적 불가. `changePassword()` 같은 의미 있는 메서드로.
- `@NoArgsConstructor(PROTECTED)` — JPA용 기본 생성자는 두되 외부 사용은 차단
- **`@Data`·클래스 전체 `@ToString` 금지** — 연관관계 필드까지 순회해 무한 루프·의도치 않은 쿼리 발생

## 검증
- `spring.jpa.hibernate.ddl-auto: create` + `show-sql: true`로 띄워 **생성된 DDL을 ERD와 눈으로 대조**하는 게 가장 빠르다. 운영 전에는 `validate`/`none`으로 바꾼다 ([[MySQL-연동]]).

## 출처
- Claude Code 세션 자동 캡처 (scratch-2026-09-21-bfd610, 2026-09-22)
