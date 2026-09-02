---
tags:
  - spring
  - jpa
  - entity
  - repository
updated: 2026-09-02
---

# JPA 엔티티와 리포지토리

> JPA = "자바 객체 ↔ DB 테이블 변환" **표준 규격**. 구현체는 Hibernate.
> 어노테이션 패키지가 `org.springframework...`가 아니라 `jakarta.persistence`인 이유.
> [[빈과-DI|DI]]가 스프링 없이도 성립하는 설계이듯, JPA도 스프링과 별개의 표준이다.

## 엔티티 — 테이블과 자바 세계를 잇는 공식 대표

**클래스 = 테이블 구조, 객체 하나 = 행 하나.**

```
post 테이블                     Post 클래스
│ 1 │ 첫 글 │ 안녕 │    ↔    Post 객체 (id=1)
│ 2 │ 두번째│ 재밌다│    ↔    Post 객체 (id=2)
```

하는 일 3가지:
1. **테이블 설계도** — `ddl-auto: update`가 클래스를 읽고 CREATE TABLE ([[MySQL-연동]])
2. **저장 시 SQL 재료** — `save(post)` → 필드값 꺼내 INSERT로 번역
3. **조회 시 결과 그릇** — SELECT 결과 행을 Post 객체로 조립해서 반환

```java
@Entity
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;    // 관례로 title 컬럼
    private String content;

    protected Post() { }     // JPA 필수: 빈 생성자

    public Post(String title, String content) {
        this.title = title;
        this.content = content;
    }

    public Long getId() { return id; }
    public String getTitle() { return title; }
    public String getContent() { return content; }

    public void update(String title, String content) {
        this.title = title;
        this.content = content;
    }
}
```

### 줄별 핵심

- `@Id` — 기본키. JPA가 "이 객체가 어느 행인지" 판단하는 기준. 모든 엔티티 필수
- `@GeneratedValue(IDENTITY)` — id 발급을 DB에 맡김 (MySQL auto_increment)
- `Long`(대문자) — 소문자 `long`과 달리 **null 가능**. 저장 전 = 아직 행이 아님 = id 없음(null)을 표현 (TS의 `number | null`)
- getter — private 필드의 읽기 창구. **응답 JSON 만들 때 스프링이 호출** (`getTitle()` → `"title"`)
- setter 없음 — 수정은 `update()` **공식 통로 하나로만**. 변경 경로가 한 곳이면 나중에 규칙 추가가 쉽다
- 엔티티도 그냥 자바 클래스 — **자기 데이터를 다루는 메서드는 정당한 구성원** (Dog의 bark()처럼)

> [!important] 빈 생성자가 필수인 이유
> 조회 시 JPA는 "빈 객체 생성 → 컬럼값 채우기" 순서로 동작한다.
> 자바 규칙: **생성자를 하나라도 직접 쓰면 자동 기본 생성자가 사라짐** → `Post(title, content)`를 만들었으니 빈 생성자를 직접 복구해야 했던 것.
> `protected`인 이유: JPA는 쓸 수 있고, 우리 코드의 `new Post()`(빈 게시글) 실수는 컴파일에서 차단.

## 리포지토리 — 인터페이스 한 줄의 마법

```java
public interface PostRepository extends JpaRepository<Post, Long> {
    List<Post> findByTitleContaining(String keyword);
}
```

- `<Post, Long>` — 제네릭: "다루는 엔티티는 Post, id 타입은 Long" (TS 제네릭과 동일 감각)
- `extends JpaRepository` — save / findAll / findById / deleteById / count 등 **수십 개 선언 상속**
- 몸통이 비어도 되는 이유: **구현체를 서버 시작 때 스프링이 즉석 생성해서 빈으로 등록**. `EmailSender`를 내가 구현하던 일을 스프링이 대신하는 것 → `@Repository` 어노테이션도 불필요

| 메서드 | SQL |
| --- | --- |
| `save(post)` | id 없으면 INSERT, 있으면 **UPDATE** |
| `findAll()` | SELECT * FROM post |
| `findById(id)` | SELECT ... WHERE id=? → `Optional<Post>` |
| `deleteById(id)` | DELETE ... WHERE id=? |

> [!important] 쿼리 메서드 — 이름만 선언하면 구현이 생긴다
> `findByTitleContaining` → 스프링이 이름을 해석 (`findBy` + `Title` + `Containing`)
> → `WHERE title LIKE %?%` SQL 자동 생성. 몸통 0줄.

## 확인 문제

> [!question]- Q1. `new Post("제목","내용")` 직후 id 값은? 언제 채워지나?
> null. `save()` 실행 시 INSERT 후 DB가 발급한 번호를 JPA가 받아와 채운다. 그래서 id는 대문자 Long.

> [!question]- Q2. PostRepository는 인터페이스인데 어떻게 주입되나?
> 스프링 데이터 JPA가 시작 시 구현 클래스를 만들어 빈으로 등록 → 서비스 생성자에 그 구현체가 꽂힘. 우리는 규격만 알고 구현체 정체는 모름 — DI 그대로.

> [!question]- Q3. `save`가 INSERT와 UPDATE를 구분하는 기준은?
> 객체의 id. 없으면(null) 새 행 INSERT, 있으면 그 행 UPDATE.

## 다음에 볼 것

- [ ] `@Column(length=...)` 등 컬럼 세부 설정
- [ ] `Optional` 제대로 다루기 (`orElseThrow`)
- [ ] 연관관계 매핑 (`@ManyToOne` — 댓글, 작성자 붙일 때)
