---
tags: [PR, 스택PR, Docker, Testcontainers, JKYMHS, 어디갈래, 워크플로]
updated: 2026-11-10
---
# 남은 API 8개 스택 PR(#16~#30) 구성, 로컬 개발 환경(Docker), 수동 테스트 함정

인증 이후 남은 API(공통 기반·콘텐츠·탐색·회원·찜·여행·AI 추천·캘린더)를 병렬 워크플로로 구현해 **머지 없이 스택 PR 8개**로 올린 구조와, 로컬에서 띄우는 컨테이너, 직접 테스트할 때 실패하기 쉬운 지점을 정리한다. 인증 PR은 [[JKYMHS-Backend/인증-API-구현과-PR-리뷰-결정사항]], 데이터는 [[JKYMHS-Backend/TourAPI-데이터-수집과-FE-ERD-불일치]], 스택 PR 머지 기법은 [[Git-스택-PR-재정렬-force-with-lease]].

## PR 구성 (2026-11-10, 전부 미머지)
| PR | 내용 | base | 테스트 |
|---|---|---|---|
| #16 | 공통 기반(에러 코드·커서·시계·이벤트·비동기 실행기) | develop | 51 |
| #18 | 지역·분류·콘텐츠, TourAPI CSV 적재·04:00 동기화 | #16 | 108 |
| #21 | 탐색 목록·자동완성·상세·주변·출발지 검색 | #18 | 156 |
| #22 | 내 정보·취향·동의·비밀번호·탈퇴 | #14(#16·#18 머지본) | 363 |
| #25 | 찜·조회 기록·행동 데이터·인기 점수 배치 | #22(#21 머지본) | 445 |
| #26 | 여행·일정안·확정·경로 계산·방문 확인 | #22 | 425 |
| #29 | 구글 캘린더 연동·업로드·자동 동기화 | #26 | 478 |
| #30 | AI 코스 추천(기본은 mock) | #26(#25 머지본) | 560 |

- **브랜치 결정**: 공개 API는 `develop`에서 S→D0→D1로 쌓고, 로그인이 필요한 API는 인증 후속 브랜치(#14)에서 땄다(#8 단독으로는 부족). 그래서 회원 쪽 PR diff에 인증 내용이 함께 보이며 PR 본문에 적었다.
- **머지 순서**: 아래쪽 PR부터 **merge commit**으로. squash하면 위에 쌓인 PR이 전부 꼬인다. 인증 PR과 #16·#18이 만나는 지점에서 `ErrorCode.java`·`application.yml`·`.env.example`이 한 번 충돌하며 양쪽을 다 남기면 되고, 해결본이 #22 머지 커밋에 이미 있다.
- **공통 기반을 따로 뺀 이유**: 여러 브랜치가 동시에 고치면 충돌이 나는 파일(특히 `ErrorCode`)을 한 번에 정의해 두고 나머지는 쓰기만 하게 한다. 같은 code에 문구를 상황별로 바꿔 던지는 3인자 `CustomException(ErrorCode, message, description)`을 쓴다. 422는 Spring 7에서 `UNPROCESSABLE_ENTITY`가 deprecated라 `UNPROCESSABLE_CONTENT`.
- 공통 기반 요소: `CodeEnum`+`@JsonValue`·컨버터(없는 enum 값 → 400 "입력값이 올바르지 않습니다."), `CursorCodec`(base64url JSON, 변조 시 400 `INVALID_CURSOR`, 숫자가 Integer/Long/Double로 섞여 나와 `requireLong` 등 헬퍼 필수)·`CursorPage{items,nextCursor,hasNext}`, KST `Clock` 빈, `batch.enabled`로 켜는 스케줄링, `async.enabled=false`면 `SyncTaskExecutor`(→ AFTER_COMMIT 리스너가 인라인 실행되므로 그 안의 DB 쓰기는 `REQUIRES_NEW`), 도메인 간 이벤트(`UserWithdrawnEvent` 등)와 행동 가중치(조회 1·찜 3·일정 저장 5·방문 만족 10·불만족 -3).
- `security` 스타터를 넣은 뒤로는 새 공개 API마다 `SecurityConfig`에 `permitAll`을 추가해야 한다(#22가 regions·categories·travel-styles, #25가 contents). **새 API가 401이면 제일 먼저 `SecurityConfig`**.

## 워크플로 운영에서 배운 점
- 에이전트가 DNS 오류(`EAI_AGAIN`)로 죽으면 뒤 단계가 전부 "upstream failed"가 된다. 스크립트에 **이미 만든 이슈·브랜치·워크트리를 재사용하는 resume 규칙**과 null 결과 1회 재시도를 넣고 `resumeFromRunId`로 이어 돌리면 성공한 단계는 캐시로 재사용된다. 결과가 비정상이면 `journal.jsonl`을 읽는다.
- `gh pr create`가 네트워크 reset으로 실패해도 PR은 만들어져 있을 수 있다 → 재시도 전에 `gh pr list --head <브랜치>` 확인.
- 병렬 에이전트의 "테스트 통과" 보고는 워크트리에 남은 테스트 결과 파일(마지막 커밋 이후인지)로 재확인했다.
- 메인 작업 트리에 커밋 안 된 수집 도구 변경이 남아 있으면 다른 브랜치로 체크아웃이 막히므로, 브랜치에 들어간 파일은 되돌리고 데이터 폴더는 `.git/info/exclude`(로컬 전용 무시)에 넣어 둔다.

## 로컬 개발 환경 (Docker)
- 스프링 서버는 Docker가 아니라 `./gradlew bootRun`(8080). Dockerfile 없음.
- **`compose.yaml`** (`docker compose up -d`): MySQL 8.4(`jkymhs-mysql`, 3306, DB·계정·비밀번호 `jkymhs`, root `root`, utf8mb4·Asia/Seoul), Redis 7.4(`jkymhs-redis`, 6379), Mailpit(`jkymhs-mailpit`, SMTP 1025·웹 8025, 인증·방문 확인 메일을 가로채 http://localhost:8025 에 표시, 인증 메일 PR부터 존재). **MySQL만 볼륨**이라 `down`은 데이터를 남기고 `down -v`가 지운다. 테이블은 `ddl-auto: update`라 기존 제약은 고치지 못해 **스키마가 크게 바뀐 PR을 테스트하기 전엔 `down -v`**.
- **테스트는 Testcontainers**(MySQL 8.4·Redis 7.4·ryuk)가 테스트 JVM당 1회, 랜덤 포트로 띄워 로컬 DB·compose 포트와 안 부딪힌다. Docker만 켜져 있으면 되고 꺼져 있으면 `./gradlew test`가 전부 실패(코드 문제 아님). 메일·구글·카카오·ODsay·TourAPI·AI 서버는 가짜 응답.
- 외부 키(없어도 서버는 뜨고 해당 기능만 어림값·오류): `KAKAO_REST_API_KEY`(출발지 검색·택시), `ODSAY_API_KEY`(대중교통, 서버용이라 공인 IP 등록), `CALENDAR_TOKEN_ENCRYPTION_KEY`(`openssl rand -base64 32`, **한 번 정하면 변경 금지** — 바꾸면 전원 캘린더 재연동), Google Cloud Calendar API·권한 범위 2개.
- Redis 확인: `docker exec -it jkymhs-redis redis-cli` → `KEYS *`, `HGETALL`, `TTL`.

## 직접 테스트할 때 실패하기 쉬운 곳
- **CI가 사실상 없음**(CodeRabbit뿐, 기본 브랜치가 아닌 base의 PR은 SKIPPED) → 브랜치마다 `./gradlew test`를 직접 실행.
- **FE Vite 프록시 `changeOrigin: true` + 백엔드 `CORS_ALLOWED_ORIGINS` 미설정(기본값 비어 있음) = 브라우저 POST 전부 403**. Host가 `localhost:8080`으로 바뀌는데 Origin은 FE 포트라 cross-origin으로 판정된다. `.env`에 FE 실제 포트를 넣는다(워크트리마다 포트가 다름). 프록시가 `/api` 접두사를 rewrite로 떼면 쿠키 `Path=/api/auth`와 안 맞아 재발급이 조용히 실패. 허용 헤더는 `Authorization`·`Content-Type`뿐이라 Sentry 헤더 등을 붙이면 사전 요청이 403.
- **`.env`는 실행 디렉터리 기준**(`optional:file:.env`)이라 워크트리에서 띄우려면 `.env`를 복사해야 한다(없으면 `JWT_SECRET` 미해결로 기동 실패). 값에 따옴표 금지, `KEY=` 빈 값은 기본값을 덮어씀(`AUTH_COOKIE_SECURE=`이면 시작 실패).
- Secure 쿠키는 http://localhost에서만 저장되고 Safari·LAN IP 접속은 거부 → `AUTH_COOKIE_SECURE=false`. curl은 `-c/-b` 쿠키 jar, 안 보내면 `-H "Cookie: refreshToken=…"`.
- 실제 시간 경과(잠금 10분·코드 5분·액세스 30분·리프레시 14일)·실제 SMTP·구글 동의 화면은 테스트가 목/짧은 시간으로 대체해 검증 안 됨. 구글 인가 코드는 **URL 디코딩해서** 보내야 하고(`4%2F0…`→`4/0…`) 같은 code 재사용은 401 `invalid_grant`.
- 잠금 해제 시각은 JVM 시스템 시간대 기준(서버가 UTC면 KST와 달라 보임), IP 제한은 `getRemoteAddr()`라 프록시 뒤에서는 전원이 한 IP로 집계된다.

## 팀과 맞출 것 (미결)
- AI 서버 `POST /v1/courses` 요청·응답 형식은 제안안(출발지 좌표 `conditions.origin` 추가), ERD·노션 명세 반영, FE에 에러 문구 기준 공유(탭에 안 맞는 필터 값만 "잘못된 필터 값입니다."), 일정안 이름 변경은 `version` 불변, `indoor_outdoor.csv` 검토, 구글 신규 가입의 약관 동의 저장 스펙.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
