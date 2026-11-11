---
tags:
  - spring
  - crud
  - controller
  - service
updated: 2026-11-11
---

# 스프링 CRUD 계층구조

> 요청 하나가 지나가는 길: **Controller → Service → Repository → DB**
> 각 계층은 바로 아래 계층만 [[빈과-DI|생성자 주입]]으로 받는다.

## 계층별 담당 질문

| 계층 | 담당 질문 | 예 |
| --- | --- | --- |
| Controller | HTTP를 어떻게 자바로/자바를 HTTP로 바꾸나 | JSON 변환, URL 매핑 |
| Service | 우리 앱의 규칙은 무엇인가 | 제목 필수, 수정 절차 |
| Repository | 어떻게 저장하고 꺼내나 | SQL, 테이블 → [[JPA-엔티티와-리포지토리]] |

앱의 본체는 Service. Controller는 교체될 수 있는 입구, Repository도 교체 가능(Map → MySQL로 실제 교체해봄. Service 코드는 안 바뀌었다).

## Controller — HTTP 번역가

```java
@RestController
public class PostController {
    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @PostMapping("/posts")
    public Post create(@RequestBody PostRequest request) {
        return postService.create(request.title(), request.content());
    }
}
```

- `@RestController` = [[스프링-컨테이너|창고]] 등록 + 요청 매핑 + 리턴값을 응답 본문(JSON)으로
- 담당은 **URL + HTTP 메서드 조합**으로 정해짐 — `GET /posts`와 `POST /posts`는 완전히 다른 입구

> [!warning] 직접 겪은 것
> 전체 조회를 한다며 Postman에서 POST로 보냈더니 → `create()`가 실행되어 null 게시글이 **생성**됨. 같은 주소여도 메서드가 다르면 다른 문이다.

### 요청에서 값을 꺼내는 3종 세트

| 어노테이션 | 값의 위치 | 예 |
| --- | --- | --- |
| `@PathVariable` | URL 경로의 빈칸 | `/posts/`**`1`** |
| `@RequestParam` | 물음표 뒤 | `?keyword=`**`첫`** |
| `@RequestBody` | 요청 본문 JSON | `{"title": ...}` |

감각: **누구를**은 경로, **옵션**은 쿼리스트링, **덩어리 데이터**는 본문.

> [!important] JSON 키 이름 = record 필드명
> `@RequestBody`는 키 이름이 일치하는 필드만 채우고, 못 찾으면 **에러 없이 null**로 둔다.
> `record PostRequest(String title, String content)` ← JSON도 정확히 `"title"`, `"content"`여야 함.

## Service — 규칙의 방

```java
@Service
public class PostService {
    private final PostRepository postRepository;

    public Post update(Long id, String title, String content) {
        Post post = postRepository.findById(id).orElse(null);  // 찾고
        if (post != null) {                                    // 정책 판단
            post.update(title, content);                       // 엔티티에게 시키고
            postRepository.save(post);                         // 저장
        }
        return post;
    }
}
```

- 한 줄짜리 전달 메서드가 많아 보여도, **로직이 생길 자리를 잡아두는 것** (검증, 알림, 권한...)
- 규칙을 Service에 두는 이유: 입구(Controller)가 여러 개 생겨도 규칙은 한 곳
- `Optional`: `findById`는 "있을 수도 없을 수도 있는 상자"를 반환 → `.orElse(null)`로 처리 중 (TS의 `Post | undefined` 감각)

## DTO와 Entity — 같은 "데이터 모양"이지만 향하는 곳이 다르다 (2026-10-28 추가, bium-be 예시)

- **DTO**: 프론트와 주고받는 JSON 모양(`~Request` 받는 상자 / `~Response` 돌려주는 상자). **Entity**: DB 테이블 모양. 한 줄로: DTO=API 경계, Entity=DB 경계.
- 엔티티를 그대로 JSON으로 내보내면 안 되는 이유: ① `deletedAt`·`createdBy` 같은 불필요/민감한 값이 나가고 연관 엔티티가 줄줄이 딸려 나오거나 직렬화 에러, ② 요청 DTO에 받을 필드만 두면 JSON에 `fridge` 등을 끼워 넣어도 안 바뀜 + `@NotBlank`·`@Size` 검증을 DTO에 두고 컨트롤러 `@Valid`가 서비스 전에 걸러냄, ③ 화면이 원하는 묶음(냉장/냉동/실온·칸별)과 DB의 한 줄씩 쌓인 행 사이를 변환, ④ 컬럼을 바꿔도 DTO가 같으면 API 계약이 유지됨.
- 흐름: JSON → 요청 DTO(`@Valid`) → Service가 엔티티를 찾아 `item.update(...)` → `Response.from(엔티티)` → JSON. 비움 DTO는 `record`(불변·접근자 자동 생성).
- DTO 모양을 FE·BE가 다르게 알고 있으면 화면이 깨진다 → API를 만들기 전에 DTO 형식부터 같이 정한다.

## 예외가 에러 응답이 되는 길 (비움 구조)

- `throw`는 그 메서드를 즉시 끝내고 예외를 위로 올림 → 컨트롤러도 안 잡음 → `@RestControllerAdvice`의 `GlobalExceptionHandler`가 `@ExceptionHandler(BusinessException.class)`로 받아 JSON(`{success:false, data:null, error:{code,message}}`)과 상태 코드로 변환.
- `ErrorCode` enum에 상태 코드+메시지를 묶고, 서비스는 `throw new BusinessException(ErrorCode.X)`만 쓴다 → 에러 응답 모양이 API마다 달라지지 않음. `RuntimeException`을 상속해 `throws` 선언 불필요.
- 서비스 메서드는 **빠르고 공짜인 검사(멤버 확인·재료 존재)를 먼저, 느리고 비용 드는 호출(Gemini)은 그 뒤에**. 거절된 요청이 호출 횟수 제한을 소모하지 않게 하는 순서도 이 때문.
- 서비스에 `new` 없이 쓰이는 필드는 `@RequiredArgsConstructor`(final 필드 생성자 자동 생성)+`@Service` 빈의 생성자 주입 → [[빈과-DI]]. `@Transactional` 대신 `TransactionTemplate`로 저장 구간만 감싸는 이유는 [[Spring-Boot-4-JPA-실전-함정-메모]].

### ErrorCode·CustomException·ErrorResponse·GlobalExceptionHandler 상세 (2026-11-03 추가, JKYMHS-Backend)

- **`ErrorCode`**: 값마다 `HttpStatus`+문구를 가진 enum(`@Getter`+`@RequiredArgsConstructor`로 getter·final 필드 생성자 생성). `HttpStatus`는 Spring이 만든 상태코드 enum(`getStatus().value()`가 숫자). 같은 409라도 `EMAIL_ALREADY_EXISTS`/`GOOGLE_ONLY_ACCOUNT`처럼 상태코드만으론 구분이 안 돼 이름·문구를 따로 붙인다. `@Getter`는 모든 필드, `@RequiredArgsConstructor`는 초기값 없는 final 필드만 대상.
- **`CustomException extends RuntimeException`**: `errorCode`+`description`을 담는다. 1인자 생성자는 `this(errorCode, errorCode.getMessage())`로 description을 기본 문구로 채우고 2인자가 실제 저장. `super(errorCode.getMessage())`는 부모의 message 자리를 채워 `e.getMessage()`·로그용이며 **응답에는 쓰이지 않는다**(응답 `message`는 ErrorCode, 커스텀 문구는 `description`). 서비스가 직접 판단한 규칙 위반만 던지고, 입력 형식은 `@Valid`, 예상 못 한 오류는 그냥 올려 500.
- **`ErrorResponse`**: `record`(`message`, `status`, `timestamp`, `description`; 접근자에 `get` 없음). `of(...)` 정적 팩토리 3개가 오버로딩으로 ①→②→③ 순서로 채워 넘기고 실제 생성은 ③(`HttpStatusCode` 받음 — Spring 415처럼 `ErrorCode` 없는 에러용).
- **`GlobalExceptionHandler`**(`@RestControllerAdvice`+`@ExceptionHandler`): 가장 구체적인 타입의 핸들러가 선택되고 파일 순서는 무관. `CustomException`→ErrorCode 상태, `MethodArgumentNotValidException`→400(첫 필드 문구가 message, 전체 `필드: 문구`가 description), 깨진 JSON·파라미터 누락→400, `NoResourceFoundException`→404, 메서드 불일치→405(`Allow` 헤더), Redis 연결 실패·타임아웃→503, 그 외 `Exception`→500. 마지막 안전망에서 Spring `ErrorResponse` 인터페이스의 4xx는 원래 상태를 유지하지 않으면 415 등이 500이 된다(우리 `ErrorResponse`와 이름이 겹쳐 Spring 쪽은 FQN). 컨트롤러 단계 예외만 처리하므로 Security 필터의 401/403은 `HandlerExceptionResolver`로 넘겨야 한다 → [[Spring-Security-JWT-로그인-잠금-리프레시-토큰-설계-함정]].
- Redis `timeout`/`connect-timeout`을 2초로 두면 Redis가 멈춰도 요청이 오래 매달리지 않고 위 핸들러가 503으로 응답.
- JPA Auditing: `BaseTimeEntity`(`@MappedSuperclass`+`@EntityListeners(AuditingEntityListener)`+`@CreatedDate/@LastModifiedDate`, 생성 시각 `updatable=false`)는 `@EnableJpaAuditing`이 없으면 시각이 `null`. 메인 클래스가 아니라 별도 `JpaAuditingConfig`에 둔다 — 동작 원리와 실험은 [[JPA-Auditing-동작-원리와-EnableJpaAuditing-분리-이유]], Redis 503·타임아웃 실측은 [[Redis-장애-503-타임아웃-실측과-WebMvcTest-테스트-함정]].

### 컨트롤러 반환 방식과 요청 헤더 (2026-11-11 추가, JKYMHS-Backend 인증 컨트롤러 읽으며)

- **`@PostMapping`에 주소를 안 적으면 클래스의 `@RequestMapping` 주소가 그대로** 쓰인다. 적으면 클래스 주소 뒤에 이어 붙는다(`/api/auth/google` + `/link`).
- **반환 타입 고르기**: 객체(record)를 그냥 돌려주면 Spring(Jackson)이 200+JSON 본문으로 보낸다(record는 접근자 `accessToken()`이 있어 바로 되고, 일반 클래스는 getter 필요). 상태 코드가 항상 같으면 `@ResponseStatus(CREATED)` 같은 애너테이션으로 정한다. **`ResponseEntity<T>`(상태 코드+헤더+본문 상자, Spring 제공, `<T>`는 본문 타입)로 감싸는 건 헤더를 붙여야 할 때(리프레시 토큰 `Set-Cookie`)나 상황별로 상태 코드가 달라질 때**. 에러 응답은 둘 다 아니고 서비스에서 `CustomException`을 던져 `GlobalExceptionHandler`가 만든다. 기본은 객체 반환, 필요할 때만 `ResponseEntity`.
- 컨트롤러 메서드 선언 순서는 `접근제어자 반환타입 메서드이름(매개변수)`. 서비스는 결과 데이터(`GoogleLoginResponse`)만 만들고 컨트롤러가 그걸 `ResponseEntity`로 감싼다.
- `@JsonInclude(NON_NULL)`을 record에 붙이면 null 필드가 JSON에서 빠져, 한 DTO로 `LOGGED_IN`/`SIGNED_UP`(토큰 있음)과 `LINK_REQUIRED`(토큰 없음, `linkToken`·`email`만) 같은 갈래별 응답을 낼 수 있다. FE는 `result` 값으로 다음 화면을 정한다.
- **`@RequestHeader(value = USER_AGENT, required = false)`**: 요청 헤더 하나를 꺼내는 기능. User-Agent는 리프레시 토큰 행(`refresh_tokens.user_agent`)에 "어느 기기의 로그인인가" 기록용으로만 저장하고 **읽는 코드는 아직 없다**(ERD에 있는 컬럼이라 만든 것, 탈취 조사·"로그인한 기기 목록"에 쓸 수 있음). 클라이언트가 마음대로 바꿔 보낼 수 있어 보안 판단에는 쓰지 않는다.
- **`@Transactional` 기초**: DB 작업 여러 개를 전부 성공/전부 취소로 묶는다(정상 종료=커밋, 예외=롤백). 따라오는 기능이 **변경 감지**(트랜잭션 안에서 읽은 엔티티 필드를 바꾸면 `save()` 없이 커밋 직전 UPDATE 자동 실행 — 로그아웃의 `token.revoke(now)`가 이 방식)와 잠금 유지(`for update`는 트랜잭션 끝에 풀림). 서비스 클래스에는 `@Transactional(readOnly = true)`를 기본으로 걸고 쓰는 메서드만 `@Transactional`로 덮어쓰면 읽기 쪽은 변경 감지·flush를 건너뛰어 가볍고 실수로 바꿔도 반영되지 않지만, **쓰는 메서드에 붙이는 걸 빠뜨리면 저장이 안 된다**.
- **Optional + 람다**: `findByTokenHash(...).filter(t -> 내 것).filter(t -> !t.isRevoked() && !t.isExpired(now)).ifPresent(t -> t.revoke(now))`는 "상자가 비면 이후 단계가 전부 건너뛰어지는" 체인. 람다(`t -> 조건`)는 이름 없는 짧은 메서드이고 `!`는 뒤집기, `&&`는 둘 다 참일 때만 참.
- 로그아웃은 토큰이 없든·남의 것이든·이미 폐기됐든 **항상 204**(응답이 다르면 "이 토큰은 살아 있다"는 정보가 샌다). 로그아웃해도 다른 기기 토큰은 유지되고, 액세스 토큰은 서버가 취소할 수 없어 남은 30분은 통과하므로 FE가 저장해 둔 액세스 토큰을 지워야 한다.

## 확인 문제

> [!question]- Q1. "제목 없는 글 거부" 검증은 Controller/Service/Repository 중 어디에? 왜?
> Service. Controller에 두면 입구가 늘 때마다 규칙이 복사/누락되고, Repository는 보관만 해야 하므로. 입구가 몇 개든 규칙은 한 곳.

> [!question]- Q2. `GET /posts`와 `POST /posts`가 안 겹치는 이유는?
> 담당 배정 기준이 URL이 아니라 **URL + HTTP 메서드 조합**이라서.

> [!question]- Q3. 서비스의 update와 엔티티의 update는 뭐가 다른가?
> 서비스의 update = 수정 업무 전체(찾기→바꾸기→저장). 엔티티의 update = 그중 "값 바꾸기" 한 단계. 서비스가 엔티티의 것을 호출한다.

## 테스트 방법 — GET은 브라우저, 나머지는 Postman/curl

GET은 주소창에 치면 끝(`localhost:8080/posts`). 본문이 필요한 POST/PUT/DELETE는 도구가 필요하다.

- **Postman** — Body 탭 → `raw` → 오른쪽 드롭다운을 반드시 **JSON**으로. `Text`로 두면 `@RequestBody` 매핑이 안 되거나 415 에러가 난다.
- **curl**:
```bash
curl -X POST http://localhost:8080/posts -H "Content-Type: application/json" \
  -d '{"title":"제목","content":"내용"}'
curl -X PUT http://localhost:8080/posts/1 -H "Content-Type: application/json" \
  -d '{"title":"수정","content":"바뀐 내용"}'
curl -X DELETE http://localhost:8080/posts/1
```

요청을 보낼 때마다 콘솔의 `show-sql`([[MySQL-연동]]) 출력으로 "내 요청 → 어떤 SQL"이 찍히는 걸 같이 보면 학습 효과가 크다.

## 다음에 볼 것

- [ ] 검증 `@Valid` — null 제목 저장 사건 재발 방지
- [ ] 없는 id 조회 시 404 (`ResponseEntity`, `@ExceptionHandler`) — 지금은 200 + 빈 본문
- [ ] `@Transactional` — 서비스 메서드의 DB 작업을 전부 성공/전부 취소로 묶기
