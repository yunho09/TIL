---
tags: [Spring, Security, JWT, Redis, Testcontainers, MySQL]
updated: 2026-11-03
---
# Spring Security JWT 인증 구현 — 로그인 잠금·리프레시 토큰 설계와 리뷰에서 재현된 함정

Spring Boot 4.1 / Security 7.1 / jjwt 0.13 기반 인증을 구현하고 별도 리뷰어가 **실제로 재현한** 버그들을 모은 페이지. 액세스 토큰은 JWT, 리프레시 토큰은 DB에 해시로 저장한 랜덤 문자열이고, 로그인 실패 잠금·IP 요청 제한은 Redis Lua 스크립트로 구현했다. 프로젝트별 결정은 [[JKYMHS-Backend/인증-API-구현과-PR-리뷰-결정사항]].

## 토큰 설계
- **액세스 토큰(JWT, 30분)**: DB 저장 안 함. 본문은 암호화가 아니라 누구나 읽을 수 있으므로 회원 ID(`sub`)만 넣는다.
- **리프레시 토큰(14일)**: JWT가 아니라 `SecureRandom` 32바이트 문자열. 재발급 때 어차피 DB를 봐야 하고, 서명 키가 유출돼도 위조할 수 없다.
- **해시만 저장**: DB가 유출돼도 토큰을 못 쓴다. **BCrypt가 아니라 SHA-256**을 쓰는 이유는 같은 입력이 항상 같은 값이라 받은 토큰을 해시해 `findByTokenHash`로 찾을 수 있어서(BCrypt는 매번 값이 달라 조회 불가). `token_hash`에는 unique 인덱스.
- **폐기는 행 삭제가 아니라 `revokedAt` 채우기** → 폐기된 토큰이 다시 오는 것을 탈취 신호로 판단할 수 있다.
- **rotation**: 재발급마다 새 토큰 발급 + 이전 토큰 폐기. 폐기된 토큰이 다시 오면 전 기기 로그아웃. 401을 던져도 "전부 폐기"는 커밋되도록 `@Transactional(noRollbackFor = CustomException.class)` 계열 설정이 필요하다(`RuntimeException`이면 기본은 롤백).
- 동시 재발급 경합은 `@Lock(PESSIMISTIC_WRITE)` 행 잠금으로 직렬화, 일괄 폐기는 `@Modifying` 벌크 UPDATE.

### 재발급 유예(grace)와 token family (#13에서 추가)
- 새로고침·탭 두 개 동시 복원·응답 도착 전 새로고침이면 **폐기된 토큰이 정상 사용자에게서 다시 온다** → 그대로 탈취로 보면 모든 기기가 로그아웃된다.
- 대응: 재발급으로 폐기된 토큰은 `rotated_at`을 기록해, **10초 이내 재사용은 새 토큰 발급, 이후는 전체 폐기+401**, 로그아웃으로 폐기된 토큰은 단순 401(다른 기기를 끊을 이유 없음).
- 유예를 넣자 리뷰에서 **로그아웃 직후 10초 안에 이전 토큰으로 세션이 되살아남**이 재현됨 → 토큰 묶음 `family_id`(로그인 때 발급, 재발급 때 상속)를 두고 로그아웃 시 묶음 전체 폐기.
- 만료 확인을 폐기 확인보다 먼저: 안 그러면 반년 전에 만료된 옛 토큰으로도 전 기기 로그아웃을 유발할 수 있다(행이 purge되지 않으므로).
- `@Lock`을 지워도 기존 테스트가 통과했다 → 락을 쥔 스레드를 두고 다른 스레드가 500ms 내 끝나지 않는지 보는 **락 대기 테스트**가 필요. 락 보유자는 레포지토리 메서드가 아니라 raw SQL로 잡아야 `@Lock` 제거 시 보유자까지 안 풀려 엉뚱한 이유로 실패하지 않는다.

## 로그인 5회 실패 → 10분 잠금
- 순서가 "잠금 확인 → BCrypt 비교(~0.1초) → 틀리면 +1"이면 동시 요청 30개가 모두 잠금 확인을 통과해 30번 시도할 수 있다(재현됨).
- 해결: **비교 전에 횟수부터 올린다.** 잠금 확인·카운트 +1·만료 10분 재설정·5 이상이면 잠금 시각 저장을 **하나의 Lua 스크립트**로 묶는다(Redis는 스크립트를 한 번에 실행해 중간에 다른 요청이 못 끼어듦). 성공하면 `reset()`으로 기록 삭제 — 5번째 시도에서 맞히면 정상 로그인.
- INCR/EXPIRE/SET을 따로 보내면 EXPIRE 실패 시 키가 영원히 남고, SET 실패 시 카운트 5 이상인데 잠금이 없는 상태가 생긴다.
- **없는 이메일도 똑같이 세고 잠근다**(가입 여부 추측 방지). 로그인에서 없는 이메일에도 가짜 BCrypt 비교를 돌리는 것도 같은 이유(응답 시간 차이).
- 구글 전용 계정(비밀번호 없음)은 비밀번호가 틀린 게 아니라 로그인 방법이 다른 것 → 실패 횟수를 올리기 전에 먼저 걸러낸다.

## 이메일 대소문자·악센트와 Redis 키 불일치
- MySQL 기본 collation(`utf8mb4_0900_ai_ci`, 로컬 compose의 `utf8mb4_unicode_ci`도 동일)은 `USER@x.com`·`usér@x.com`을 같은 값으로 찾지만 Redis 키는 문자열 그대로라 **변형마다 5번씩 추가 시도**가 가능했다.
- 해결: 요청 경계에서 `strip().toLowerCase(Locale.ROOT)`로 정규화 + `AuthRedisKeys`에서도 정규화 + 컬럼을 `columnDefinition = "varchar(255) collate utf8mb4_bin"`으로 정확히 비교하게 해 DB와 Redis의 "같은 이메일" 기준을 일치시킨다.
- 파생 위험(열린 질문): 악센트 무시 매칭이 비밀번호 재설정까지 닿으면 닮은 주소 소유자가 남의 계정을 재설정할 수 있다.

## 리뷰에서 재현된 그 밖의 함정
- **Testcontainers 공유 컨테이너가 컨텍스트 종료 때 꺼짐**: Boot 4.1의 `TestcontainersLifecycleBeanPostProcessor`는 `destroyMethod = ""`를 "프레임워크가 안 닫는다"로 보고 `container.close()`를 직접 호출한다(주석과 반대). `@DirtiesContext` 클래스가 끝나면 캐시된 컨텍스트를 쓰는 다음 클래스가 `Unable to connect to Redis`로 실패 → 테스트 순서에 따라 결과가 달라짐. 해결: 컨테이너를 빈으로 등록하지 말고 static 기동 + `@DynamicPropertySource`(또는 `ConnectionDetails` 빈).
- **비동기 엔드포인트가 유효한 토큰인데 401**: STATELESS면 Security 컨텍스트가 요청 속성(`RequestAttributeSecurityContextRepository`)에 저장되는데 필터가 저장하지 않고 `OncePerRequestFilter`는 async 2차 디스패치를 건너뛰어 인가가 익명으로 다시 돈다. `Callable`·`DeferredResult`·`CompletableFuture`·`SseEmitter` 모두 해당. 해결: `createEmptyContext()`로 컨텍스트를 만들어 set하고 `new RequestAttributeSecurityContextRepository().saveContext(...)`. 관련 ERROR 디스패치 함정은 [[Spring-Security-403이-본문-없는-401로-바뀌는-ERROR-디스패치]].
- **jjwt `Keys.hmacShaKeyFor` + `signWith(key)`는 키 길이로 알고리즘이 정해짐**: 32~47바이트 HS256, 48~63 HS384, 64+ HS512. `openssl rand -base64 48`(64자)이면 HS512로 서명된다. `signWith(key, Jwts.SIG.HS256)`로 명시하고 파서도 제한.
- **`.env` import가 test 프로필에도 적용**: 개발자 로컬 `.env`의 SMTP·구글 키가 테스트로 새어 머신마다 결과가 달라지고 실제 메일이 나갈 수 있다 → local 전용 프로필/`on-profile: "!test"` 문서로 한정. `.env.example`에서 `MAIL_PORT=`처럼 **값을 비워두면 `${MAIL_HOST:localhost}` 기본값이 빈 문자열로 덮여** 버린다(주석 처리로 해결).
- **Hibernate가 enum을 MySQL 네이티브 `ENUM`으로 생성**: `length`가 무시되고, `ddl-auto: update`로는 값을 추가해도 컬럼이 안 바뀌어 `Data truncated`. `@Enumerated(STRING)` + `@JdbcTypeCode(SqlTypes.VARCHAR)`로 VARCHAR 고정.
- **BCrypt 72바이트 한계**: Spring Security 7에서 72바이트 초과는 `IllegalArgumentException` → 500. 한글 25자가 75바이트. `@Size`는 문자 수라서 **바이트 길이 검증**이 필요.
- **필터 단계 예외는 `@RestControllerAdvice`가 못 받음**: `AuthenticationEntryPoint`·`AccessDeniedHandler`에서 `HandlerExceptionResolver.resolveException(request, response, null, new CustomException(...))`로 넘기면 401·403도 공통 JSON이 된다. 리졸버가 null을 돌려주면(처리 실패) 빈 200이 나가므로 `sendError` 폴백이 필요.
- `/api/auth/**` permitAll + 로그아웃만 exact path `authenticated`는 훗날 `/api/auth/logout/all`이 공개 규칙에 걸린다 → 공개 엔드포인트를 명시 나열하는 편이 안전.
- 잘못된 JWT 시크릿 길이 검증을 `@ConfigurationProperties`에 걸면 Boot의 시작 실패 메시지가 `Value: "<secret>"`로 시크릿을 로그에 찍는다 → 검증은 `JwtProvider`에서 값 없이 예외로.
- TRUNCATE 기반 테스트 정리는 `FOREIGN_KEY_CHECKS=1` 복구를 `finally`에, `SET SESSION lock_wait_timeout=5`로 (기본 1년이라 hang 가능).
- Security 필터 체인 순서: `.requestMatchers(...)`는 **처음 맞는 규칙**이 적용 → 구체 경로(`/api/auth/logout`)를 `/api/auth/**`보다 위에.
- **필터는 토큰이 잘못돼도 바로 401을 주지 않고** 실패 이유를 요청 속성에 남긴 뒤 통과시킨다 — 인증이 필요 없는 경로까지 막지 않고, 막을지는 `SecurityConfig` 규칙과 EntryPoint가 결정.

## 쿠키·CORS (리프레시 토큰을 httpOnly 쿠키로)
- 액세스 토큰은 본문(FE 메모리), 리프레시 토큰은 `Set-Cookie: refreshToken=…; Path=/api/auth; HttpOnly; Secure; SameSite=Lax`. localStorage는 XSS에 탈취되므로 피한다.
- `whereto.com` → `api.whereto.com`은 **같은 사이트지만 다른 오리진**이라 CORS와 `credentials: "include"`가 모두 필요하다("같은 사이트면 CORS 불필요"는 틀린 통념). 다른 사이트면 `SameSite=None`이 필요하고 Safari에서 막힐 가능성이 크다(브라우저 확인 안 함). 같은 오리진(리버스 프록시)이면 둘 다 불필요.
- 재발급 요청만 API 서버로 가므로 `Domain=.whereto.com`은 이점 없이 다른 서브도메인에도 쿠키를 보내 빼는 편이 낫다. CORS 허용 오리진은 환경변수로 받고 `*`·스킴 없는 값·`user@`·`#` 포함은 시작 시 실패시킨다.
- `@CookieValue String`은 `%ZZ` 같은 쿠키에서 500 → `Cookie` 타입으로 받아 직접 파싱. User-Agent에 한글이 있으면 헤더 저장에서 500이 날 수 있다.
- CORS 필터가 거절한 요청의 403 본문은 Spring 기본 `Invalid CORS request`라 공통 에러 형식이 아니다(브라우저가 FE 코드에 숨김). `Origin` 헤더가 없는 요청은 필터를 통과한다.

## SSE 대신 폴링을 택한 이유 (작업 상태 갱신)
- 규모(동시 수십 명)에서 폴링 부하는 1~1.5초당 단순 조회 정도라 문제 없고 SSE로 줄어드는 건 최대 1~2초 지연뿐(실측이 아니라 계산).
- 브라우저 기본 `EventSource`는 `Authorization` 헤더를 못 붙이고, 쿠키는 `Path=/api/auth`라 SSE 경로엔 안 간다 → 인증을 따로 설계해야 한다. 연결 수명·하트비트·프록시 버퍼링·재연결 복원, 서버 증설 시 Redis pub/sub까지 따라온다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
