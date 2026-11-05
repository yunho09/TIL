---
tags: [BeforeAI, API연동, CORS, 내부망, Tailscale, 해커톤]
updated: 2026-11-05
---

# BeforeAI 실서버 연동: 명세-서버 불일치, CORS, 내부망 판정

목 API로 만든 프론트를 배포된 백엔드에 붙이면서 확인한 것들. 노션 명세에 필드명이 없어 프론트가 정한 이름이 서버와 달랐고, CORS 설정이 `.env`에 덮여 반영이 안 됐으며, "내부망" 판정이 계정이 아니라 **접속 IP**로만 이뤄져 외부망 PC로는 승인 흐름을 시험할 수 없었다. 프로젝트 배경은 [[BeforeAI-FE/AI-작업-승인-플랫폼-설계-결정]].

## 1. 명세 대신 서버의 OpenAPI를 기준으로 대조
- Spring(springdoc)이 코드에서 자동 생성하는 `/v3/api-docs`가 로그인 없이 열려 있었다. 사람이 쓴 노션 명세는 실제 서버와 달라서 이쪽을 기준으로 삼았다. (운영 전에는 닫는 게 좋음)
- 대표 불일치:
  - 로그인 `employeeNo` → 서버는 **`employeeNumber`**. 서버가 비밀번호를 보기 전에 400을 돌려 화면엔 "요청 값이 잘못되었습니다"만 뜸.
  - 로그인 응답 `temporaryToken`·`passwordChangeRequired`, OTP 확인 필드 `temporaryToken`·`otpCode`(프론트는 `tempToken`·`code`).
  - 요청 상세는 `plan`이 아니라 `currentPlan`, 승인 대기 목록은 작업이 `task` 안에 중첩, 거부는 `planId`·`taskIds` 필수, 웹쉘 공개키는 `encryption`/`signing`으로 분리.
  - 프론트가 호출하는 78개 중 16개(대시보드·감사 로그·문서·알림 설정·Slack 등)는 서버에 없음 → 호출 코드는 두고 "아직 서버에 없는 기능입니다" 안내로 처리.
- 확인법: 가짜 값으로 양쪽 필드명을 모두 보내 보면 맞는 이름일 때만 "비밀번호 틀림(401)"이 온다.
- 서버 소스를 읽어 얻은 추가 함정(프론트 몫):
  - `hashTarget`은 FORBIDDEN 작업을 빼지만 `plan.tasks`에는 남아 있어, 그대로 비교하면 서명이 거부됨 → 비교 전에 걸러야 함.
  - 비밀번호 변경 토큰은 `GET /me`·`PUT /me/password`·로그아웃만 가능(나머지 409). 변경 뒤 토큰을 새로 받지 않으면 화면이 막힘.
  - OTP 확인 호출이 **파괴적**: 필드명이 틀리면 서버가 방금 스캔한 비밀키를 새 값으로 덮어쓰면서 클라이언트는 성공으로 표시.
  - 승인 키 등록의 `adminProof`는 실제로 검증됨: 문자열 `beforeai:register-signing-key:{keyId}:{publicKey}`를 Ed25519로 서명한 64바이트 Base64.

## 2. 백엔드 쪽 시연 차단 요인 (당시 소스 기준)
- AI 분석기(Bedrock) 미연결: 요청이 `ANALYZING`에서 멈춰 Plan이 안 생기고, `REVIEWING`을 만드는 코드가 없어 승인 흐름에 도달 불가. 실행 결과 콜백도 없어 승인 후 `RUNNING`에 머묾.
- 대응 선택지: ① 분석기 연결 ② 백엔드에 `REVIEWING` 샘플 요청·Plan 시드 ③ 시연은 목 모드로 하되 실서버 동작처럼 말하지 않기. 프론트는 분석이 ~60초 넘으면 "평소보다 오래 걸립니다"+취소를 보여 주고 가짜 Plan은 만들지 않기로 함.
- 외부망 정책 불일치: 승인·거부 API는 내부망 전용인데 `approval-context`는 외부망 작업도 승인 가능으로 표시 → 외부 승인자가 서명 후 403. `PUT /me/password`도 내부망 전용이라 첫 로그인 비밀번호 변경 계정은 외부망에서 진행 불가.

## 3. CORS: 코드 기본값보다 환경변수가 우선
- 허용 목록이 `cors.allowed-origins=${CORS_ALLOWED_ORIGINS:기본값}` 형태라, 커밋으로 기본값에 localhost를 넣고 CD가 성공해도 서버 `.env`에 `CORS_ALLOWED_ORIGINS`가 있으면 무시된다. 환경변수를 바꾸면 **서비스 재시작**이 필요.
- 진단은 출발지별 사전 요청으로: `curl -i -X OPTIONS <서버>/api/v1/auth/login -H "Origin: http://localhost:5173" -H "Access-Control-Request-Method: POST"` → 200과 `Access-Control-Allow-Origin`이 나오면 적용된 것. 배포 도메인(Pages)만 허용·localhost 403이면 아직 안 열린 것.
- 쿠키를 함께 보내므로 `Allow-Origin: *`는 브라우저에서 안 먹고 `allowedOriginPatterns("*")`가 필요(전체 개방은 운영 전 닫기).
- 로컬 개발은 Vite 프록시로 CORS를 우회: `VITE_API_BASE_URL=/api/v1` + `VITE_API_PROXY_TARGET=http://서버:포트`(프록시 대상에는 `/api/v1`을 붙이지 않음).
- 배포 화면(HTTPS Pages)에서 `http://` API를 부르면 혼합 콘텐츠로 막힘 → API에 HTTPS 필요. 리프레시 쿠키가 `SameSite=Strict`라 프론트·API 사이트가 다르면 새로고침 때 로그아웃될 수 있고, HTTP인 동안은 `AUTH_REFRESH_COOKIE_SECURE=false`가 필요(HTTPS 후 true로 복귀).

## 4. `VITE_API_BASE_URL`은 `/api/v1`까지 포함
- 프론트는 이 값 뒤에 `/auth/login` 같은 경로를 이어 붙이므로 `/api/v1`을 빼면 `/auth/login`으로 나가 실패. `/api`는 화면 경로와 구분, `/v1`은 호환 깨질 때 `/v2`를 병행하기 위한 버전.
- `.env.production.local`은 `npm run build`에서만 읽고 `npm run dev`는 안 읽어 목 API로 돈다(gitignore 대상, 팀원에게 없음). 값이 설정되면 목/개발 도구/데모 코드는 빌드 시점에 제외(번들 grep으로 검증).

## 5. 내부망 판정은 계정이 아니라 접속 IP
- 서버 `NetworkResolver`가 소켓 주소(`getRemoteAddr`)로만 판단, `X-Forwarded-For`는 일부러 불신. 기본 `INTERNAL_CIDRS=100.64.0.0/10`(Tailscale 대역).
- 그래서 **프론트 개발 도구로 내부망 전환은 불가**(화면만 바뀌고 서버가 `INTERNAL_NETWORK_ONLY` 403). 서버에 우회 헤더를 넣는 것도 보안 규칙에 뒷문을 만드는 일이라 비추천. 계정별 내부망 처리도 계정 탈취 시 외부에서 내부망 권한이 되므로 말릴 것.
- 외부망 로그인은 OTP 등록 계정만 가능(OTP 등록은 내부망 전용). 내부망이면 OTP 없이 비밀번호만으로 로그인.
- 내부망 시험 방법:
  1. 서버 `.env`의 `INTERNAL_CIDRS`에 내 공인 IP `/32` 추가 후 재시작(끝나면 제거, 같은 공유기 사용자도 내부망 취급됨)
  2. Tailscale: PC를 같은 tailnet에 넣고 서버의 `100.x.x.x`로 접속(시연에는 실제 구조라 이쪽). 서버가 tailnet에 있어야 함 — 당시 CD·배포 문서에는 흔적 없고 tailnet에 있는 건 웹쉘이라고만 적힘.
  3. SSH 터널(`ssh -N -L 18080:localhost:8080 …`)이면 서버가 출발지를 `127.0.0.1`로 보므로 `127.0.0.1/32` 추가 필요(나중에 Nginx를 앞에 두면 모두 127.0.0.1로 보이므로 빼야 함)
  4. 백엔드 로컬 실행 `SPRING_PROFILES_ACTIVE=local`(루프백이 내부망, 시드 생성)
- 외부/내부를 한 PC에서 오가기: 포트별 프록시 둘(`localhost:5175`→공인 경로=외부망, `127.0.0.1:5176`→Tailscale 경로=내부망). 쿠키는 포트를 구분하지 않아 같은 호스트명이면 리프레시 쿠키가 섞이므로 호스트명을 다르게 씀.
- 테스트 계정을 DB에 직접 넣을 때: 비밀번호는 BCrypt 해시(`htpasswd -nbBC 10 "" '비번' | tr -d ':\n'`), 외부망용은 OTP 비밀키(Base32)도 같이. 컬럼명·OTP 형식은 추정이라 `\d users`로 확인 필요 (Claude 보충, 미검증).

## 6. 분석 LLM 키 관련 결정
- Max 요금제의 월 $100 API 크레딧은 Claude Platform(Claude API)에만 적용되고 **Bedrock에는 못 씀**(AWS 청구). 이월 안 됨, 소진되면 다음 달까지 API 요청 중단, Max 사용량 한도와는 별개.
- 키는 프론트(`VITE_` 값은 번들에 노출)·저장소·채널에 넣지 말고 서버 `.env`의 `ANTHROPIC_API_KEY`로만. 만료일이 시연 전이면 시연 날 분석이 통째로 멈추므로 만료를 시연 이후로 잡고 잔액 확인. Plan JSON은 구조화 출력(`output_config.format`)으로 받고 강제 `tool_choice`는 400.
- EC2에서 Bedrock을 쓴다면 키보다 IAM 역할이 낫다. 이후 백엔드 master에 `BedrockAiAnalyzer`가 들어오는 중이라 어느 쪽으로 갈지 팀 정리가 필요했음(미확정).

## 확인 상태
- 실서버에서 확인한 것: 외부망 관리자 로그인·세션 유지, 대시보드(`/dashboard` 404라 요청 목록으로 집계), 내 요청, 승인 대기(0건), 외부망에서 막히는 기능 안내.
- **미확인**: 내부망 기능 전부(작업 요청·승인 키 등록·서명·승인·정책 등)는 서버 문서와 소스를 읽고 맞춘 것이지 실제 요청으로 돌려 보지 않음. 이 시점에 목 모드 브라우저 회귀 스크립트 다수가 실패했고 실제 버그와 낡은 테스트를 구분하는 중이었음.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/BeforeAI-FE, 2026-11-05)
