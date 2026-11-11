---
tags: [Spring, Security, JWT, Redis, Testcontainers, MySQL]
updated: 2026-11-11
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

## `User` 엔티티·`SecurityConfig` 읽기 메모 (2026-11-04 보충)
- **DB unique 제약은 `existsByEmail` 확인이 있어도 필요**: 같은 이메일로 동시에 가입 요청이 오면 둘 다 확인을 통과할 수 있고, 그때 두 번째 저장을 DB가 막는다. `google_sub`의 unique는 MySQL이 `NULL`을 여러 개 허용해서 구글 미연결 회원이 많아도 충돌하지 않는다.
- 가입 방식별 컬럼: 이메일 가입=`passwordHash`만, 구글 가입=`googleSub`·`googleEmail`만, 가입 후 연결=셋 다. `googleEmail`은 가입 이메일과 다를 수 있어 따로 저장. `hasPassword()`가 false면 구글 전용 계정.
- 생성자를 `private`+정적 팩토리(`createWithPassword`, 구글은 `createWithGoogle`)로 막아 이메일 가입에 구글 값이 섞이거나 온보딩 상태가 `NOT_STARTED`가 아닌 채로 만들어지는 걸 방지. JPA용 `@NoArgsConstructor(PROTECTED)`, setter 없음. id는 `Long`(저장 전 `null`로 미저장 구분), `IDENTITY`=MySQL `AUTO_INCREMENT`.
- `OnboardingStatus`(`NOT_STARTED`/`SKIPPED`/`DONE`)는 이름만 있으면 되는 enum이라 필드·`@Getter`가 없다. 인증 PR들에는 값을 바꾸는 코드가 없어 온보딩 기능에서 변경 메서드를 추가해야 한다.
- 세션을 안 쓰는 설정: `csrf`·`formLogin`·`httpBasic`·`logout`을 끄고 `SessionCreationPolicy.STATELESS`. 직접 만든 JWT 필터는 `addFilterBefore(…, UsernamePasswordAuthenticationFilter.class)`로 Security 필터 사이에 끼운다. 컨트롤러에서는 `@AuthenticationPrincipal Long userId`로 꺼낸다. 비밀번호용 `PasswordEncoder`(BCrypt)는 빈으로 등록.
- `JwtProperties`(`jwt.*`)는 값이 없거나 키가 32바이트 미만이면 서버가 시작되지 않는다(`JWT_SECRET` 누락 시 기동 실패를 실제 서버로 확인).
- 로그인 잠금 사용 순서: `start(email)`(잠겨 있으면 423) → 비밀번호 틀리면 `recordFailure`(5번째면 423)+401 → 성공하면 `reset`. Lua는 잠금 시각이 안 지났으면 `-1`, 지났으면 기록 삭제 후 재시작 → 카운트+1·TTL 10분 → 5 이상이면 잠금 시각 저장.

## Boot 4 / Security 7 설정 함정과 리뷰 후속 수정 (2026-11-10 보충)
- **`spring-boot-starter-security`를 넣는 순간 모든 URL이 로그인 요구**, 기동 로그에 임시 계정 비밀번호(`Using generated security password`)가 찍힌다. JWT만 쓰면 `spring.autoconfigure.exclude`에 **`org.springframework.boot.security.autoconfigure.UserDetailsServiceAutoConfiguration`**(Boot 4 경로, Boot 3의 `...autoconfigure.security.servlet...`은 클래스를 못 찾아 기동 실패)를 넣고, 로그에 그 문구가 없는지로 확인. 공개 API는 `permitAll`을 직접 추가.
- Security 7: `WebSecurityConfigurerAdapter`·`antMatchers` 없음 → `SecurityFilterChain` 빈 + `requestMatchers`. jjwt 0.12+: `Jwts.parser().verifyWith(key).build().parseSignedClaims(token)`, `signWith(key, Jwts.SIG.HS256)` (옛 `setSigningKey`/`parseClaimsJws` 예제는 안 맞음). 알고리즘을 안 적으면 jjwt가 키 길이로 HS384/512를 고르므로 HS256으로 고정하고 파서에도 `.sig().clear().add(HS256)`으로 제한. 토큰에는 회원 ID(sub)만(JWT 본문은 암호화가 아닌 Base64).
- Gradle: `jjwt-api`는 `implementation`, `jjwt-impl`·`jjwt-jackson`은 `runtimeOnly`(구현 클래스를 실수로 import 못 하게 함, MySQL 드라이버와 같은 이유). 부트 BOM이 관리하는 라이브러리는 버전을 적지 말고(jjwt는 BOM에 없어 직접), Testcontainers 2.x는 모듈명이 `testcontainers-mysql`처럼 접두사가 붙는다. Docker가 꺼져 있으면 테스트 전부 실패.
- `${JWT_SECRET}`에 **기본값을 일부러 안 둠**: 키를 깜빡해도 서버가 뜨면 코드에 공개된 키로 토큰이 서명돼 위조 가능 → 없으면 기동 실패가 안전. `.env` 우선순위는 OS 환경변수 > `.env`.
- 이메일 collation(`utf8mb4_bin`)이 필요한 이유: MySQL 기본 비교는 대소문자 무시인데 Redis 키는 글자 하나만 달라도 다른 키라, `User@x.com`·`USER@x.com`으로 바꿔 가며 5회 잠금을 피할 수 있다. DTO에서 소문자 정규화 + DB는 정확 비교(정규화를 빠뜨리면 대소문자만 다른 이메일이 별개 회원으로 가입 가능).
- **구글 첫 로그인 동시 2건 → 500**: "조회 후 없으면 저장" 사이에 틈. 저장이 `DataIntegrityViolationException`이면 같은 조회를 한 번 더 해서 평소 흐름(같은 구글 계정 `LOGGED_IN`/같은 이메일 `LINK_REQUIRED`)을 그대로 탄다(무조건 409로 바꾸면 흐름이 달라짐). `login()`에 `@Transactional`이 없어 실패한 저장은 이미 롤백되고 재조회는 새 트랜잭션. 테스트는 `@MockitoSpyBean`으로 "이메일 조회 직후 다른 스레드가 저장"을 재현.
- **로그인 요청 길이 상한 없음**: 실패 기록 Redis 키(`auth:login:fail:{email}`)에 이메일이 그대로 들어가고 비교 전에 기록돼 없는 이메일도 10분간 남는다 → 1MB 이메일로 키 폭증 가능. 이메일 254자·비밀번호 72자(BCrypt 한계라 어떤 계정과도 맞을 수 없음)로 제한.
- **이모지·한글 비밀번호 500**: 정규식 `\S{8,20}`은 한글(3바이트)·이모지(4바이트)를 통과시켜 20자여도 72바이트를 넘길 수 있다(`a1`+이모지 18개=74바이트, 한글 비밀번호는 가입 성공). `[\x21-\x7E]{8,20}`(ASCII 인쇄 가능 문자)로 바꾸면 최대 20바이트. 한글·이모지 비밀번호는 이제 400이고 메시지가 같아 FE에서 입력 단계 안내가 필요.

## 이메일 인증 → 가입 → 로그인 흐름 상세와 JWT 기초 (2026-11-11 보충)

### 이메일 인증 토큰(`verificationToken`)
- 흐름: `send`(IP 제한 1분 5회 → 가입용인데 이미 가입이면 409 → 60초 내 재요청 429 → 6자리 코드를 Redis에 5분 저장 → 메일 발송, **실패하면 코드 삭제 후 502**) → `verify`(시도 +1 → 5회 초과 410 → 틀리면 400+남은 횟수 → 맞으면 코드 삭제+토큰 발급, 10분).
- **토큰 확인은 Redis 키로**: 키 `auth:verified:{용도}:{소문자 이메일}` = 랜덤 UUID, TTL 10분. 가입 때 `consume`이 Lua로 "저장값 == 보낸 토큰이면 DEL"을 한 번에 실행(0이면 401 `EMAIL_NOT_VERIFIED`). 키에 이메일·용도가 들어 있어 A 이메일 토큰으로 B 가입, 가입용 토큰을 재설정에 사용이 막히고, 같은 토큰 두 번·동시 두 요청은 하나만 통과한다. 코드를 맞히면 바로 가입시키지 않고 토큰을 주는 이유: 코드 확인과 가입은 다른 요청이라 "인증한 그 사람이 가입한다"는 증거가 필요해서.
- 토큰은 가입 처리 **전에** 먼저 지워지므로 그 뒤 단계가 실패하면(동시 같은 이메일 가입 등) 이메일 인증부터 다시 해야 한다(드물어 그대로 둠).
- 코드는 `SecureRandom`(`Random`은 예측 가능), 비교는 `MessageDigest.isEqual`(`String.equals`는 다른 글자에서 바로 멈춰 응답 시간으로 몇 글자 맞았는지 추측 가능). 코드 비교 전에 횟수부터 올리는 건 로그인 잠금과 같은 이유. 코드는 `@Pattern("\\d{6}")`. 메일 서버 타임아웃은 5초(기본 무한 대기라 SMTP가 멈추면 요청도 멈춤), 실제 Gmail은 `MAIL_FROM`을 그 계정으로 맞춰야 스팸 분류를 피한다.
- 이메일 소문자 정규화는 요청 DTO의 **record compact 생성자**(`public EmailSendRequest { email = email == null ? null : email.trim().toLowerCase(Locale.ROOT); }`)에서 한다. send·verify·가입이 규칙이 다르면 다른 Redis 키를 보게 된다. IP는 `request.getRemoteAddr()`라 프록시/LB 뒤에선 전원이 1분 5회를 나눠 쓴다(배포 시 `X-Forwarded-For` 처리 필요, 미처리).
- **빠진 기능**: `purpose: RESET_PASSWORD`로 코드 발송·확인까지는 되지만, 그 토큰으로 새 비밀번호를 바꾸는 API(`POST /api/auth/password/reset {email, verificationToken, newPassword}` → `consume(RESET_PASSWORD)` → 변경 → 그 회원 리프레시 토큰 전부 폐기)가 어느 브랜치에도 없었다(FE에는 비밀번호 찾기 화면 A-2·A-3이 있음). 이슈 #31로만 남기고 #22 머지 후 develop에서 `PasswordRule`을 재사용해 구현하기로 함.

### 가입(`SignupRequest`·`AuthService.signup`)
- 비밀번호 정규식 `(?=.*[A-Za-z])(?=.*\d)…`의 `(?=…)`는 전방탐색(위치는 안 옮기고 "어딘가에 영문/숫자 있어야" 조건만 검사). `@NotBlank`와 `@Pattern` 메시지를 같게 둔 건 빈 문자열이 두 검증을 동시에 실패시켜 어느 문구가 나올지 정해져 있지 않기 때문.
- **약관 필드는 `Boolean`(래퍼)+`@NotNull`**: `boolean`이면 JSON에서 값이 빠져도 조용히 `false`가 된다. 필수 3개는 `@AssertTrue`, 선택 2개도 true/false를 반드시 보내야 하며(없으면 400) "안 보냄"을 "미동의"로 짐작하지 않는다. 중첩 record에는 **`@Valid`를 붙여야** 안쪽 `@AssertTrue`가 돈다(빠뜨리면 필수 약관 false로도 가입됨).
- `@Transactional` 하나로 회원 저장·약관 동의·리프레시 토큰 저장을 묶어 중간 실패 시 반쪽 데이터가 안 남는다. 다만 맨 앞 `consume`(Redis)은 롤백되지 않는다. `existsByEmail`은 빠른 안내용이고 **진짜 중복 방지는 DB UNIQUE**, 동시 가입의 `DataIntegrityViolationException`은 409로 변환.
- 약관 동의는 동의한 것만이 아니라 **항상 전체 항목 행을 저장**(안 한 건 `agreed=false`, `agreedAt=null`)해 마이페이지에서 켜고 끌 때 "행이 없는 경우"를 따로 처리하지 않는다. `(user_id, consent_type)` UNIQUE.
- 로그인 확인 순서: IP 제한(429) → 구글 전용 계정(409, 실패 횟수에 안 넣음) → 잠금 확인+시도 기록(423) → 비밀번호 비교 → 성공 시 기록 삭제. 없는 이메일도 `DUMMY_PASSWORD_HASH`(실제 해시와 같은 cost `$10$`)와 비교해 응답 시간을 맞춘다. **`LoginRequest`는 `@NotBlank`(+길이 상한)만** — 형식 규칙을 알려 주면 힌트가 되고 틀려도 같은 401이면 되므로.

### JWT 기초 (학습 정리)
- JWT는 서버가 서명해 내주는 출입증: `헤더.내용.서명` 세 부분(Base64URL 인코딩일 뿐 암호화 아님). 내용 예 `{"sub":"7","iat":…,"exp":…}`, 서명 = `HMAC-SHA256(헤더+"."+내용, 서버 비밀키)`. 내용을 바꾸면 서명이 안 맞아 거절된다. 요청마다 DB를 안 봐도 누구인지 아는 게 장점.
- `parseSignedClaims`가 형식(`MalformedJwtException`)·허용 알고리즘(`JwtException`)·서명(`SignatureException`)·만료(`ExpiredJwtException`)를 한 번에 검사. 헤더의 `alg`는 보내는 쪽이 적을 수 있으므로 허용 목록을 HS256으로 제한해 `none` 같은 토큰을 막는다. `sub`가 숫자가 아니면 형식이 깨진 토큰으로 취급.
- **약점**: 한 번 발급한 JWT는 서버가 아무것도 저장하지 않아 만료 전 취소 불가 → 수명을 30분으로 짧게 하고, 대신 취소 가능한 리프레시 토큰(랜덤, DB 해시, 14일, 재발급에만 사용)을 둔다.
- 요청 인증 흐름: `Authorization: Bearer …` → `JwtAuthenticationFilter`가 검증해 `SecurityContext`에 회원 ID를 기록(실패하면 "만료/위조" 이유만 요청 속성에 남기고 **어느 쪽이든 다음으로 넘김**) → `SecurityConfig` 규칙이 로그인 필요 여부 판단 → 필요한데 기록이 없으면 `EntryPoint`가 401 → 컨트롤러는 `@AuthenticationPrincipal Long userId`.
- 리프레시 토큰 발급(`TokenService.issue`): 액세스 토큰 생성 → `SecureRandom` 32바이트를 Base64URL 43자로 → 만료 14일 → `refresh_tokens`에 **해시만** 저장(기기=로그인마다 한 행) → 원문 두 개를 `IssuedTokens` record로 묶어 반환(원문은 앱만 보유).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
