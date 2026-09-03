---
tags:
  - spring
  - crud
  - controller
  - service
updated: 2026-09-04
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
