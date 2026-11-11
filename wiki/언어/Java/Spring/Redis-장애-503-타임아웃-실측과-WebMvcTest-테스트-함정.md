---
tags: [Spring, Redis, 테스트, 예외처리, JKYMHS-Backend]
updated: 2026-11-11
---
# Redis 장애 503 처리·타임아웃 실측과 `@WebMvcTest` 테스트 함정

Redis가 죽거나 멈췄을 때 인증·로그인 제한은 우회하지 않고 503을 주도록 두 예외를 한 핸들러로 잡고, 타임아웃을 2초로 줄여야 멈춤 상황에서 요청이 60초씩 매달리지 않는다. 예외 응답 전체 구조는 [[스프링-CRUD-계층구조]] 참고.

## 503으로 바꾸는 예외 두 개
- `RedisConnectionFailureException` — 연결 자체가 안 됨(Redis 꺼짐)
- `QueryTimeoutException` — 연결은 되는데 응답이 없음(Redis 멈춤)
- 로그는 stack trace 없이 메시지만 남긴다(장애 중엔 요청마다 에러라 trace까지 찍으면 로그가 넘침).
- 명세의 "Redis 끊겼을 때" 표: 인증코드·로그인 제한 → 503(우회 금지) / 경로·후보 캐시 → 캐시 없이 계산(해당 기능에서 직접 `try-catch`) / 새 추천 → 실패(횟수 차감 X). 핸들러는 첫 번째 경우만 기본 처리한다.

## `timeout` 설정 유무 실측 (Redis를 직접 멈추고 요청)
| 상황 | 응답 | 시간 |
|---|---|---|
| A. 정상 | 200 | 0.24초 |
| B. 꺼짐 | 503 | 0.20초 |
| C. 멈춤 + `timeout: 2s`, `connect-timeout: 2s` | 503 | **2.0초** |
| D. 멈춤 + 설정 없음 | 503 | **60.0초** |
| E. 복구 후 | 200 | 0.002초 |
- 설정이 없으면 로그인 요청 하나가 60초 멈췄다가 503. `application.yml`의 `spring.data.redis.timeout`/`connect-timeout`을 2s로.
- 타임아웃이 나도 **명령이 취소되지는 않는다**: Redis가 살아난 뒤 밀린 명령이 그대로 실행돼 E에서 카운트가 2가 아니라 5였다. 로그인 실패 카운트가 조금 더 올라갈 수 있으나 보안상 안전한 방향.
- 별도 `RedisConfig`는 아직 안 만듦: 인증 기능이 문자열·Hash만 써서 Boot 자동 구성 `StringRedisTemplate`이면 충분. JSON을 저장하는 `auth:link` 같은 키가 필요해질 때 만든다.

## `@WebMvcTest`에서 테스트용 컨트롤러가 404가 되는 이유
- 예외 응답을 검증하려고 테스트 클래스 안에 일부러 예외를 던지는 `TestController`를 넣었더니 10개 중 9개가 404로 실패.
- 원인: **Spring Boot 테스트는 테스트 클래스 안의 중첩 클래스를 컴포넌트 스캔에서 일부러 제외**한다. 해결: `@Import(TestController.class)`로 직접 등록.
- 500 테스트는 예외 메시지에 `secret: …`을 넣고 응답에 그 글자가 없는지까지 확인(내부 메시지 비노출 검증).
- 처음에 `@WebMvcTest`가 JPA를 안 띄우는 점이 `@EnableJpaAuditing` 위치 문제와도 연결된다 → [[JPA-Auditing-동작-원리와-EnableJpaAuditing-분리-이유]].

## CodeRabbit이 짚은 에러 핸들러 버그 3건과 확인 방법
- (4xx→500, 405 `Allow` 헤더는 [[스프링-CRUD-계층구조]]·[[JKYMHS-Backend/인증-API-구현과-PR-리뷰-결정사항]]에 있음.)
- **`Accept: application/xml` 요청**: 에러 본문을 만들 컨버터가 없어 원래 예외가 `ServletException`으로 그대로 터졌다 → 에러 응답 Content-Type을 `application/json`으로 고정해 503 JSON이 나감.
- 지적마다 테스트를 하나씩 추가하고, **고치기 전 코드에서는 실패하고 고친 뒤 통과하는지**까지 확인해 버그를 실제로 잡는 테스트임을 검증.
- 우리 `ErrorResponse`(record)와 Spring의 `org.springframework.web.ErrorResponse`(인터페이스)는 이름이 같아 Spring 쪽은 전체 경로로 쓴다.
- 핸들러에서 같이 확인된 점: `@Valid` 에러가 여럿이면 어느 필드 메시지가 `message`에 올지 순서가 고정이 아님(`email`이 먼저 선언돼도 `password`가 먼저 나옴), `timestamp`가 나노초까지 나옴(초 단위로 자를 수 있음), 405는 노션 에러 코드 DB에 없어 추가 필요.

## CodeRabbit 설정(`.coderabbit.yaml`)
- `language: ko-KR`, `tone_instructions`(왜 문제인지+고친 코드 예시), `profile: chill`(중요한 지적 위주, 더 꼼꼼히는 `assertive`), `auto_review`는 `develop`·`main` 대상 PR(draft 제외), `path_instructions`에 프로젝트 규칙 7개(패키지 구조·`global`이 `domain`에 의존 금지·다른 도메인은 Service로만 접근·엔티티 `@Setter` 금지·엔티티 직접 반환 금지/DTO는 record·에러 응답 형식과 500 내부 메시지 비노출·비밀값은 환경변수·Redis 장애 시 인증 503). 공식 스키마 검증 통과.
- **설정 파일을 PR에 넣어도 GitHub App 설치는 별도**다(org에 권한을 주는 작업). 설치 전에 열린 PR은 자동 리뷰가 안 돌 수 있어 `@coderabbitai review` 코멘트로 시작. 푸시된 새 커밋은 자동으로 재리뷰되며 새 지적이 더 나올 수 있다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
