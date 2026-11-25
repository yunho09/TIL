---
tags: [Spring, 인증, PR, JKYMHS, 어디갈래]
updated: 2026-11-25
---
# JKYMHS-Backend 인증 API 구현(#2~#6, #13)과 PR 읽기 순서·결정사항

"어디갈래" 백엔드의 인증을 이슈별 스택 PR(#7~#12)로 구현하고, FE 요청으로 쿠키·`code` 필드를 후속 PR(#13)로 분리한 기록. 구현 패턴과 함정은 [[Spring-Security-JWT-로그인-잠금-리프레시-토큰-설계-함정]], 현황·ERD 매핑은 [[JKYMHS-Backend/프로젝트-현황]].

## PR 구조 (2026-11-03 기준, 전부 미머지)
| 이슈 | PR | base | 내용 |
|---|---|---|---|
| #1 공통 기반 | #7 | develop | `BaseTimeEntity`, `JpaAuditingConfig`, `ErrorCode`/`CustomException`/`ErrorResponse`/`GlobalExceptionHandler`, Redis 타임아웃 2초 |
| #2 회원·Security·JWT | #8 | #7 | `User`, `RefreshToken`, JWT 필터/프로바이더, `SecurityConfig`, `TokenService`, `LoginAttemptService`, Redis 유틸 |
| #3 이메일 인증 | #9 | #8 | 인증 메일 발송·검증(Mailpit로 로컬 확인) |
| #4 가입·로그인 | #11 | #8 | `AuthService`, 약관 동의 |
| #5 재발급·로그아웃 | #12 | #11 | rotation, 탈취 시 전체 폐기 |
| #6 구글 로그인 | #10 | #8 | `LOGGED_IN`/`LINK_REQUIRED`/`SIGNED_UP` 3갈래 |
| 후속 | #13 | (#7~#12 머지 후 develop) | `code` 필드, 리프레시 쿠키, CORS, 재발급 10초 유예, `family_id` |

- 스택 방식과 머지 규칙은 [[Git-스택-PR-재정렬-force-with-lease]]. 머지 순서 #7 → #8 → #9·#11·#10 → #12.
- 병렬 작업은 갈래마다 **수정 가능한 공용 파일을 미리 지정**(`build.gradle`은 #3만, `User.java`는 #6만 등)해 충돌을 막고, 여러 이슈가 같이 쓰는 코드(인증 에러 코드 전체·토큰 발급·로그인 잠금·IP 제한)는 #2에 먼저 넣었다.
- 읽기 권장 순서: #8 → #9 → #11 → #12 → #10 → #13. 핵심은 `SecurityConfig` → `JwtAuthenticationFilter` → `JwtProvider` → `JwtAuthenticationEntryPoint` → `TokenService.issue()` → `LoginAttemptService` → `AuthService.signup()/login()` → `TokenService.reissue()` → `GoogleAuthService.login()`. `TestContainers`·`IntegrationTestSupport` 내부와 Lua 문법은 후순위.

## 명세에 없어서 정한 것 / 열린 결정
- 비밀번호 재설정 인증 메일은 **가입 안 된 이메일이어도 같은 200**(실제 메일은 안 보냄). 다만 가입된 이메일일 때만 실제 발송하므로 응답 시간으로 가입 여부 추측 가능 — 막으려면 비동기 발송이 필요하고, 그러면 발송 실패를 502로 못 알린다(미결).
- 구글 가입(`SIGNED_UP`) 직후 "필수 약관 동의 전 다른 API 403"과 약관 동의 저장 API는 이슈에 "P1에서 결정"이라 **미구현**(현재는 동의 없이도 다른 API 사용 가능).
- 구글: 같은 이메일 계정이 이미 다른 구글 계정과 연결돼 있으면 `LINK_REQUIRED`가 조용히 덮어쓰므로 409 `EMAIL_ALREADY_EXISTS`. `login`에는 `@Transactional`을 걸지 않는다(구글 호출 동안 DB 커넥션 점유 방지) — 대가로 가입 직후 토큰 발급이 실패하면 계정만 남아 다음 구글 로그인이 `LOGGED_IN`이 된다. 구글 sub는 클라이언트에 주지 않고 `GoogleLinkTokenService` 토큰으로 대신한다. 구글 4xx 본문은 warn 로그(클라이언트 ID/시크릿 오류도 "인가 코드 만료·재사용"처럼 보여서).
- 토큰 유효기간(액세스 30분·리프레시 14일)은 명세에 "제안값". 탈취 판단 범위("그 기기 family만 vs 전 기기")는 명세대로 전 기기 유지, 미결.
- 배포 구성: 프록시 뒤면 `getRemoteAddr()`가 프록시 IP 하나라 IP 제한이 전원을 묶는다. `X-Forwarded-For`는 신뢰 프록시 설정 없이 읽으면 위조 가능.
- 배포 도메인이 정해져야 쿠키(`SameSite`)·CORS 설정이 확정된다.
- 로그아웃의 `FOR UPDATE`: 에이전트는 제거 권고(벌크 UPDATE가 같은 행을 이미 기다림, 제거 시 deadlock 197→114/300, 탈취 처리 패배 95→0), 현재는 유지. deadlock은 의도적으로 강제한 경합 수치이며 실제 운영 빈도 아님.
- 노션 명세 정비: 에러 `code`를 API 명세 42페이지에 추가(14개 신규 이름은 팀 확인 필요: `INVALID_CURRENT_PASSWORD`, `IDENTITY_VERIFICATION_FAILED`, `GOOGLE_NOT_LINKED`, `PASSWORD_ALREADY_SET`, `PASSWORD_REQUIRED_FOR_UNLINK`, `ONBOARDING_ALREADY_DONE`, `REQUIRED_CONSENT_CANNOT_REVOKE`, `SCHEDULE_VERSION_CONFLICT`, `SCHEDULE_ITEM_DUPLICATE`, `SCHEDULE_ALREADY_ENDED`, `REQUIRED_CONTENT_MISMATCH`, `VISIT_CHECK_NOT_AVAILABLE`, `CALENDAR_PERMISSION_DENIED`, `CURSOR_VERSION_MISMATCH`). 일정 1개→코스 3개 변경 반영 중 — **AI를 몇 번 호출해 코스 3개를 만들지, 실패 시 FALLBACK도 3개인지, 코스 3개일 때 지도·식사 목록 UI, ERD의 `courseId` 보존 여부는 미정**. 기능 명세 FUNC-001-01(토큰 전부 쿠키)·001-07(비밀번호 "정확히 6자")·001-09/10은 아직 새 규약과 불일치.
- ERD에 추가할 컬럼: `refresh_tokens.rotated_at`, `refresh_tokens.family_id`.
- 노션 에러 코드 DB는 API 명세 기준(410, 423, 비밀번호 8~20자, 인증 토큰 만료 401)으로 고쳐야 함. 로그인 잠금은 PRD가 429, API 명세가 423 — 423 유지.
- 의미 규칙 질문: 모든 401이 "인증이 필요합니다."인지(코드에는 다른 문구의 401이 5개) / `JwtAuthenticationEntryPoint` javadoc의 "401이면 FE가 토큰 재발급"은 재발급 엔드포인트 자신의 `INVALID_REFRESH_TOKEN`에서 루프를 만든다.

## 공통 에러 응답 구조 (#7, #13)
- 응답: `{ message, status, timestamp, description }`, #13부터 맨 앞에 `code`(= `ErrorCode` 상수명, 이름을 바꾸면 FE가 깨짐). `ErrorCode`에 없는 Spring 4xx(415 등)는 `INVALID_INPUT`.
- `CODE_RESEND_COOLDOWN`을 IP 제한과 분리해 FE가 구분 가능하게 함. 인증 메일 발송 429는 재전송 제한/IP 제한으로 나뉘고 메일 발송 실패는 502.
- 클래스 읽는 법은 [[스프링-CRUD-계층구조]]의 "ErrorCode·CustomException 상세" 참고.

## 작업 중 확인된 사실
- 병렬 에이전트 결과를 합치기 전에 서로 merge해 충돌 여부를 확인했고, Gradle이 `UP-TO-DATE`로 0.6초에 끝나 "통과"처럼 보인 적이 있어 테스트 재확인은 `cleanTest test`로 강제했다.
- 멈추라는 메시지가 도착하기 전에 에이전트가 이미 커밋·테스트를 끝냈다 → 보고서에 "테스트 안 돌림"이라 쓰면 거짓. 실제 결과(214 통과)를 그대로 PR 초안에 유지.
- 동시 20번 검증 테스트가 5/200으로 간헐 실패(`attempt > MAX_ATTEMPTS` 분기의 `delete`가 키를 먼저 지워 아무도 토큰을 못 받음) → 삭제 한 줄 제거로 0/400.

## 진행 (2026-11-04)
- PR #7이 merge commit으로 머지됨. 브랜치(`1-chore-common-base`)를 지워야 #8의 base가 `develop`으로 바뀌므로(저장소의 `deleteBranchOnMerge`가 꺼져 있음) 머지 화면에서 직접 삭제. 이후 #8부터 파일별로 읽는 중(테스트 파일은 안 봄).
- #7의 CodeRabbit **Major** 코멘트(415 같은 클라이언트 4xx까지 500으로 응답)는 11-01에 달린 것이고 이미 `ae95386`("스프링 MVC 클라이언트 오류 상태코드 유지와 405 Allow 헤더 추가")에서 수정됨. 코멘트가 가리키는 줄(500 응답)은 그대로여서 Outdated 처리가 안 돼 열려 보였을 뿐 → 화면의 코드는 코멘트 시점의 옛 버전. Resolve conversation 후 머지하면 된다.
- 필터가 토큰이 잘못돼도 바로 401을 안 주고 실패 이유(`만료된 액세스 토큰` 등)를 요청 속성에 남겨 넘기면, `JwtAuthenticationEntryPoint`가 그 이유를 `description`으로, 없으면 `액세스 토큰 없음`으로 401 응답을 만든다.

## 진행 (2026-11-10)
- #8(JWT 기반)·#11(가입·로그인)은 머지됐고 #10(구글)·#12(재발급·로그아웃)·#14(쿠키·`code`, base #12)가 열려 있다. 머지 순서는 #10 → #12 → #14, 그 위에 [[JKYMHS-Backend/남은-API-스택-PR-구성과-개발환경-Docker]]의 #16 이후가 쌓인다.
- 리뷰 후속: 구글 동시 가입 500은 #10(`32cde78`), 로그인 길이 상한은 develop 기준 새 PR #33(이슈 #32), 이모지 비밀번호 500은 #22(`92587cd`)에서 수정(`PasswordRule`로 통합돼 재설정도 같은 규칙). develop에는 #22가 머지될 때까지 이모지 버그가 남는다. 원인·해법은 [[Spring-Security-JWT-로그인-잠금-리프레시-토큰-설계-함정]].
- `refresh_tokens`에 `family_id NOT NULL`·`rotated_at`이 추가돼, #12로 만든 기존 행은 `family_id=''`로 채워져 옛 토큰이 한 family로 묶인다 → #14 테스트 전 `docker compose down -v`. 직접 테스트 체크리스트·CORS 403 함정은 [[JKYMHS-Backend/남은-API-스택-PR-구성과-개발환경-Docker]].
- 미결 유지: 구글 신규 가입(`SIGNED_UP`)이 약관 동의 저장 없이 완료됨(FE TRD는 "신규→약관 동의→온보딩"), 구글 이메일은 소문자 정규화 없이 정확 일치로 조회(대문자 Workspace 계정은 `LINK_REQUIRED`가 안 뜰 수 있음, 낮은 위험).

## 진행 (2026-11-11)
- 머지 현황: #8(`9a48ed8`)·#9(`77156a5`)·#11(`273014e`)·#12 머지. #10(구글)과 #33(로그인 길이 상한)은 develop 기준 바로 머지 가능, #14는 #10 다음. 열린 PR #16~#30은 그 위에 쌓인 상태.
- **머지된 브랜치를 base로 둔 PR은 base를 develop으로 바꿔야** 한다(안 바꾸면 이미 머지된 옛 브랜치로 들어가 버림). #8 머지 후 #9·#10·#11, #11 머지 후 #12, #12 머지 후 #14의 base를 그렇게 바꿨고 그때마다 `git merge-tree`로 충돌 여부를 확인했다.
- 머지 순서 #11 → #10 → #12 → #14: 기술적으로는 #10·#11 순서가 무관(충돌 없음)하지만 #12·#14가 #11 위에 쌓여 있고, 이슈 번호 순서(3→4→5→6)와 구글 "계정 연결"이 이메일 가입을 전제하기 때문.
- **#14가 GitHub에서 "충돌"로 뜨지만 실제 충돌이 아님**: #14 브랜치에 develop의 여러 브랜치를 따로따로 합쳐서 두 브랜치의 공통 조상 커밋이 둘이 됐고, 로컬 git은 둘을 같이 봐서 충돌 없이 합치는데(결과가 #14 파일과 동일) GitHub은 하나만 기준으로 비교해 충돌로 표시한다. 머지 직전에 #14에 develop을 한 번 합쳐 두면 사라진다(파일 내용 불변). #14 브랜치는 #10 위에 쌓였으므로 #10을 먼저 머지하면 diff가 쿠키·`code`·CORS만 남는다(미리 보면 60개 파일).
- **#16은 진짜 충돌**: #12가 `ErrorCode`에 `INVALID_REFRESH_TOKEN`을 추가했고 #16도 같은 구역에 상수를 추가 → 양쪽을 다 남기면 된다(머지 직전에 정리).
- 비밀번호 재설정 확인 API(코드 발송·확인만 있고 변경 API 없음)는 이슈 #31로만 남김. 상세는 [[Spring-Security-JWT-로그인-잠금-리프레시-토큰-설계-함정]]의 "이메일 인증 → 가입 → 로그인 흐름 상세".
- 학습용 PR 읽기 방식: 사용자가 파일별 설명을 중간에 "꼭 봐야 할 것만·코드와 함께·API가 뭘 하는지(사용자 관점)"로 바꿨다 → 앞으로 PR 설명은 핵심 위주로 압축.

## 노션 명세 ↔ PR 코드 대조 (2026-11-11)
- 스펙 마지막 수정 11/2. 이후 커밋 `bbe47ec`(인증 코드 동시 검증 수정)는 내부 수정이라 API 불변.
- **API 페이지는 목표 규약(refreshToken httpOnly 쿠키, 에러 `code` 필드)을 적었지만 PR 브랜치 코드는 아직 본문 `refreshToken`·`code` 없음** → #13 머지 전에는 FE가 페이지대로 붙이면 동작하지 않는다. 공통 응답 규약은 "지금 붙일 동작은 각 API 페이지 확인"이라고 안내하지만 API 페이지가 이미 목표 규약이라 어긋난다.
- 코드에는 있는데 명세에 없는 것: 구글 로그인 409 `EMAIL_ALREADY_EXISTS`, 계정 연결 409 `GOOGLE_ACCOUNT_ALREADY_LINKED`. 로그아웃한 토큰 재사용 시 명세는 단순 401이지만 **코드는 전체 세션을 폐기**(#13 체크리스트에도 없음).
- 인증 코드 오입력 5회: **코드는 5번째 오입력에서 코드를 무효화**한다. 문서의 "5회를 넘으면 삭제"가 틀린 쪽이라 코드 기준으로 문구만 고치면 됨(코드 주석만 확인, 실행 검증은 안 함).
- 낡은 문구: 공통 규약의 "열려 있는 PR #7~#12"(#7은 11/3 머지), 회원가입 페이지 "DB 갱신 필요", 기능 명세서 51개가 전부 "시작 전"(auth는 "진행 중"이 맞음), `CODE_RESEND_COOLDOWN`이 "구현됨"인데 브랜치 `ErrorCode`엔 없음, `SCHEDULE_LIMIT_EXCEEDED`가 규약에 없는 용도로 5개 페이지에 쓰임. `ACCOUNT_LINK_REQUIRED`는 DB에만 있고 코드는 구글 로그인 결과값 `LINK_REQUIRED`로 처리.
- 코드와 일치 확인: Redis 키·잠금 5회/10분·인증 코드 5분·재전송 60초·인증 완료 토큰 10분, 액세스 30분·리프레시 14일(IP 한도 20/5회는 미확인).
- **FE 문서와의 의도적 불일치(명세 유지)**: FE TRD는 SSE, 규약은 polling 확정 → FE 문서를 고친다. FE PRD의 `LOGIN_LOCKED` 429 등 옛 값은 11/1 결정대로 명세(423) 유지. 구글 신규 가입은 PRD가 약관 단계 이탈 시 계정 미생성, 명세·#10은 즉시 생성(명세의 "약관 동의 전 403"은 코드·이슈에 없음).
- FE 기준 명세 누락(개인 최신 명세 대조): `trips`·`plans`(여행 > 일정안) API와 `PLAN_NOT_CONFIRMED`·`CALENDAR_NOT_WRITABLE`, 함께 가기 좋은 곳·캘린더 목록·좌표→주소 변환·탈퇴 안내 삭제 개수 API, 탐색 응답의 좌표·실내외·오늘 운영 상태·빠른 필터·지도 범위 조회, 추천의 시·군·구 복수 선택·실내 위주·경로 `polyline`, 회원 위치정보 동의와 새 온보딩 취향 필드(관심 카테고리 제거, 시간대 3종, 이동 편의 단일 질문).
- 문서 사본 주의: 팀 API 명세 사본은 10/12 수정본(쿠키·`code` 없음)이고 FE TRD가 링크한 명세는 다른 워크스페이스 사본이라 사본을 고쳐도 링크는 그대로 → 링크 교체 또는 양쪽 수정 필요.

## 비밀번호 재설정 API 구현 (PR #34, 이슈 #31, 2026-11-16)
- `POST /api/auth/password/reset`. 성공하면 **모든 기기 로그아웃(리프레시 토큰 폐기) + 로그인 잠금 해제**. 테스트 7개 추가.
- **계정 열거 방지**: 가입하지 않은 이메일도 인증 토큰 오류와 **완전히 같은 예외**를 던져 401 응답 본문이 글자 하나까지 같게 했다. `CustomException`의 `description`이 응답 본문에 그대로 실리므로, 예외만 같고 description이 다르면 가입 여부가 드러난다 → 구현 때 응답에 description이 실리는지부터 확인할 것.
- #33: 주석·PR 본문에 틀리게 적힌 Redis 키 이름 정정.

## 구글 로그인(PR #10) 코드 읽기 — 설계 이유 (2026-11-20)
- **흐름**: FE가 구글에서 받은 일회용 `code`를 `POST /api/auth/google {code, redirectUri}`로 넘김 → 백엔드가 구글에 두 번 호출(`POST /token`으로 code→토큰, `GET /userinfo`로 `sub`·email) → `LOGGED_IN`/`SIGNED_UP`/`LINK_REQUIRED`. `sub`는 구글 계정마다 고정이라 이메일(바뀔 수 있음)보다 식별자로 안전.
- **결정 순서**: ① 이 sub가 연결된 회원 → `LOGGED_IN`, ② 같은 이메일 회원 → `LINK_REQUIRED`(토큰 없이 `linkToken`만), ③ 둘 다 없음 → 비밀번호 없는 계정 생성+토큰(`SIGNED_UP`). **같은 이메일이어도 자동 연결하지 않는다** — 비밀번호 확인 없이 연결하면 같은 이메일의 구글 계정만으로 남의 계정에 들어갈 수 있다. 같은 이메일 계정이 비밀번호 없는 구글 전용이면 본인 확인 수단이 없어 409.
- **`GoogleOAuthClient`**: 토큰 API는 JSON이 아니라 **form(`application/x-www-form-urlencoded`)** 으로만 받는다. 구글 4xx(코드 만료·재사용)→우리 401, 5xx·타임아웃→502, `email_verified=false`→401. **타임아웃 연결 3초·응답 5초**(없으면 구글이 멈출 때 요청 스레드가 쌓여 서버 전체가 멈춤). Spring Boot 4는 별도 스타터 없이는 `RestClient.Builder`를 자동 생성하지 않아 `GoogleOAuthConfig`에서 `RestClient`를 직접 만든다.
- **`login`에 `@Transactional` 없음**: 메서드 전체에 걸면 구글 응답을 기다리는 몇 초간 DB 커넥션을 붙잡아, 구글이 느려질 때 커넥션 풀이 고갈돼 무관한 API까지 멈춘다(외부 API 호출 코드의 흔한 실수). DB 작업은 각 서비스가 자기 트랜잭션으로.
- **동시 가입**: 버튼 연타로 같은 구글 계정 가입이 동시에 오면 하나는 UNIQUE 위반(`DataIntegrityViolationException`) → 500 대신 **재조회해서 ①·②로 처리**.
- **`GoogleLinkTokenService`**: Redis 키 `auth:link:{linkToken}`, 값 `{memberId, googleSub, googleEmail}`, TTL 10분. **sub를 FE에 주지 않는 이유**: FE가 sub를 들고 연결 요청에 되돌려 보내는 구조면 요청 조작으로 다른 sub를 끼워 넣을 수 있어, 서버(Redis)에만 두고 FE에는 추측 불가능한 UUID만 준다.
- **`link`(`POST /api/auth/google/link`)**: 비밀번호 실패를 **이메일 로그인과 같은 실패 카운트(`login:fail:{email}`)에 합산**(구글 연결 쪽으로 돌아 무한 시도 방지, 5번 틀리면 423), 비교 **전에** 시도를 먼저 기록(`start()`)해 동시 100건도 5번까지만 비교, 저장 직후 `flush`해 같은 sub를 먼저 연결한 요청의 UNIQUE 위반을 커밋 시점이 아니라 그 자리에서 잡아 409로 변환. 구글 이메일은 소문자로 정규화(`GoogleUserInfo`).
- #10은 리프레시 토큰을 응답 본문으로 주고 이후 머지된 #14가 쿠키로 바꿨다.

## 구글 OAuth 흐름·결과 3가지의 책임 경계·flush 설명 (2026-11-25)
- **인가 코드 방식**: FE는 API를 부르지 않고 **브라우저를 구글 로그인 URL로 이동**(`accounts.google.com/o/oauth2/v2/auth?client_id&redirect_uri&response_type=code&scope&state`)시킨다 → 구글이 `redirect_uri?code=…&state=…`로 돌려보냄 → FE가 `code`만 꺼내(`state` 대조) `POST /api/auth/google`로 전달. 액세스 토큰·사용자 정보는 `client_secret`을 가진 **백엔드만** 받는다(code만으론 무용, 일회용·수 분 만료). 구글 SDK 없이 `RestClient`로 `oauth2.googleapis.com/token`·`openidconnect.googleapis.com/v1/userinfo`를 직접 호출(주소는 `google.token-uri`·`user-info-uri` 설정). 팝업 방식(Google Identity Services)도 code를 받아 넘기는 구조는 같다.
- **`redirectUri`는 두 번, 같은 값**: ① FE가 구글로 보낼 때(돌아올 곳), ② 백엔드가 code→토큰 교환할 때(그때 그 주소 맞다는 확인, 다르면 `redirect_uri_mismatch`) — code를 가로채도 다른 주소로는 못 쓰게 하는 장치라 FE가 본문에 같이 보낸다. 구글 콘솔 "승인된 리디렉션 URI"에도 로컬·배포 주소를 **글자 하나(http/https·포트·끝 `/`)까지 같게** 미리 등록해야 한다.
- **책임 경계**: 본인 확인(비밀번호 체크)은 구글이 끝낸다. `LOGGED_IN`/`SIGNED_UP`/`LINK_REQUIRED`는 구글이 주는 게 아니라 **우리 서버가 구글이 준 sub·email을 DB와 비교해 정한 enum**이며 `/api/auth/google`에서만 나온다(일반 `/api/auth/login`은 가입자만 쓰므로 성공/실패 둘뿐). 한 API가 로그인+가입을 같이 처리해서 필요한 구분. `LINK_REQUIRED`는 "이메일로 가입했던 사람이 처음 구글 버튼을 누를 때"이고 **아직 로그인이 아니다**(토큰·쿠키 없이 `linkToken`만, 10분). 연결 API(`/link`)는 일반 로그인과 달리 **로그인+`google_sub` 저장**까지 하며 이메일 대신 `linkToken`을 받아 요청 조작을 막는다. 일반 로그인으로 그냥 들어가면 로그인은 되지만 연결은 안 돼 다음 구글 로그인에서 또 `LINK_REQUIRED`.
- **구글 가입 계정의 일반 로그인**: 비밀번호가 없어 `POST /api/auth/login`이 409("구글로 가입한 계정입니다")이고, 비밀번호 비교 전에 확인하므로 실패 카운트에 넣지 않는다. 로그인한 뒤 #22의 `POST /api/members/me/password`(설정, 비밀번호 없는 계정용)로 만들면 두 방식 모두 가능. `PATCH`는 이미 비밀번호가 있는 사람의 변경용. 반대로 이메일 가입자는 구글 로그인→`LINK_REQUIRED`→비밀번호 입력으로 연결.
- **코드 디테일**: `Boolean.TRUE.equals(userInfo.emailVerified())`(null이면 NPE 대신 "미인증" 처리), `catch`는 좁은 `HttpClientErrorException`(4xx→401)을 넓은 `RestClientException`(→502)보다 먼저(순서가 반대면 컴파일 에러), 구글 응답 `access_token`은 `@JsonProperty`로 매핑하고 필요한 필드만 받음, `GoogleLoginResponse`는 `@JsonInclude(NON_NULL)` + `loggedIn()/signedUp()/linkRequired()` 정적 팩토리로만 만들어 **결과와 필드 조합이 항상 맞게** 한다.
- **`saveAndFlush`가 필요한 이유**: `save()`는 변경을 모아 뒀다가 커밋 때 SQL을 보내므로 UNIQUE(`google_sub`) 위반이 메서드를 빠져나간 뒤 터져 `try-catch`로 못 잡고 500. `flush`는 **SQL을 그 자리에서 보내 DB 검사만 받는 단계(아직 롤백 가능, 확정은 커밋)**라 에러를 코드 안에서 잡아 `DataIntegrityViolationException`→409로 바꾼다. 동시 요청에서 둘 다 통과하지 않는 건 **MySQL InnoDB가 UNIQUE 인덱스 값에 자동으로 거는 잠금** 덕분 — 먼저 온 쪽이 커밋하면 늦은 쪽은 중복 에러, 롤백하면 저장 성공. JPA는 SQL만 보내며(`@Lock(PESSIMISTIC_WRITE)`도 `SELECT … FOR UPDATE`를 만들어 줄 뿐 실제 잠금은 MySQL), #22의 `getByIdForUpdate`는 같은 회원이 서로 다른 구글 계정을 동시 연결하는 것을 막으려 회원 행에 직접 건 잠금. InnoDB는 MySQL의 저장 엔진(트랜잭션·행 잠금·FK, 5.5부터 기본값, 8.4도 동일).
- **#33 로그인 길이 상한**(`LoginRequest`의 `@Size`): 이메일 **254자**(표준 최대·가입과 동일 기준 — 실패 기록 `login:fail:{email}`이 없는 이메일도 10분 남아 무제한 길이면 큰 Redis 키가 쌓임), 비밀번호 **72자**(BCrypt가 72바이트까지만 처리해 넘는 값은 어떤 계정과도 맞을 수 없음). 가입 규칙(8~20자)으로 막지 않는 이유는 "로그인은 형식을 검사하지 않는다" 원칙 — 가입 규칙이 바뀌어도 예전 비밀번호로 로그인돼야 하므로 "말이 안 되게 긴지"만 막는다. 넘으면 `@Valid`에서 400.
- 서버(`edgl.dsmhs.kr`)에는 `GOOGLE_CLIENT_ID`·`GOOGLE_CLIENT_SECRET`을 아직 안 넣어 구글 로그인을 호출하면 구글이 거절해 401(2026-11-18 기준).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/JKYMHS-Backend/develop-2)
