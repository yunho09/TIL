---
tags: [spring, spring-security, jwt, 401, 403, 테스트]
updated: 2026-10-05
---

# Spring Security: 인가 거부 403이 본문 없는 401로 바뀌는 ERROR 디스패치

커스텀 `AccessDeniedHandler` 없이 `JwtFilter extends OncePerRequestFilter` + `HttpStatusEntryPoint(UNAUTHORIZED)` 조합을 쓰면, **유효한 토큰으로 권한 밖 경로를 호출해도 403이 아니라 본문 0바이트 401**이 나간다. 원인은 핸들러 부재가 아니라 `sendError(403)` 뒤의 **ERROR 디스패치에서 JWT 필터가 다시 안 도는 것**이다. 프론트는 거부 사유를 받을 재료가 없어진다.

## 변환 체인
1. 기본 `AccessDeniedHandlerImpl`이 `response.sendError(403)` 호출 (커스텀 `accessDeniedHandler` 설정 0건이어도 기본 핸들러는 있다).
2. 서블릿 컨테이너가 `/error`로 **ERROR 디스패치**를 한다.
3. `OncePerRequestFilter`는 `shouldNotFilterErrorDispatch()` 기본값이 **true** → ERROR 디스패치에서 토큰을 다시 파싱하지 않음 → SecurityContext 비어 **익명**.
4. `/error`가 `anyRequest().authenticated()`에 걸림 → `HttpStatusEntryPoint(UNAUTHORIZED)`가 **401 빈 본문**으로 덮어씀.

- 도메인 예외는 `GlobalExceptionFilter`가 `response.setStatus()`로 직접 쓰기 때문에 ERROR 디스패치가 없고 JSON 본문이 **그대로 살아남는다**(같은 서버에서 404+JSON은 정상, 인가 거부만 401+빈 본문). 즉 "모든 403"이 아니라 **Spring Security 인가 거부(403)만** 변환된다. 초기 비밀번호 미변경 403 같은 필터 예외는 영향 없음.
- 토큰이 깨졌을 때의 401(JSON 본문 "유효하지 않은 토큰")과 본문 유무로 구분 가능: 인가 거부 401 = 본문 0바이트.

## 해결
- `JwtFilter`에서 `shouldNotFilterErrorDispatch()`를 `false`로 오버라이드, 또는 `/error`를 `permitAll`.
- 근본적으로는 `accessDeniedHandler`를 명시 등록해 403+JSON 본문을 직접 쓴다.

## 함정: MockMvc로 재현하면 403이 나와 오진한다
- `@SpringBootTest` + **MockMvc는 `sendError()` 후의 ERROR 디스패치를 실행하지 않아** 403이 그대로 보인다. 그래서 "BE는 403을 주니 네 401 관측이 틀렸다"는 잘못된 결론이 나왔다가, 실제 톰캣(`webEnvironment = RANDOM_PORT`)으로 다시 재서 배포본과 일치(401 빈 본문)함을 확인하고 철회했다.
- **서블릿 컨테이너 단계(ERROR 디스패치, 필터 체인 재진입)가 관여하는 현상은 MockMvc로 검증하지 말고 RANDOM_PORT로 잰다.**
- 가설 판별 실험 4종이 유용했다: (유효토큰+허용 라우트 → 404+JSON) / (유효토큰+권한 밖 라우트 → 401 빈 본문) / (토큰 없음 → 401 빈 본문) / (서명 변조 토큰 → 401+JSON). 첫 행이 서비스 계층까지 갔다는 건 토큰이 정상 수용된다는 증거라 "토큰 미주입" 가설이 기각된다.

## 프론트 영향
- 권한 거부 사유가 통째로 사라지고, 호출부가 401을 "로그인이 만료되었습니다"로 매핑해 두면 **세션은 멀쩡한데 만료 안내**가 뜬다. 로그아웃 루프 방어는 [[401을-세션만료로-오인한-로그인-루프]] 참고.
- 상태코드 매핑이 BE 본문보다 우선하는 FE 구조에서는, 이 BE 결함을 사용자가 못 알아채고 지나가는 부수 효과가 있었다([[API-에러-메시지-우선순위와-화면-밖-렌더링]]).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/Commonly-fe)
- 관련: [[Commonly-FE/배포본-기능-감사-2026-10]]
