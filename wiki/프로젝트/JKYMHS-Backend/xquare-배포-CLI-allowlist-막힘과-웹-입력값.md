---
tags: [xquare, 배포, CLI, 웹, JKYMHS, Spring]
updated: 2026-11-11
---
# xquare 백엔드 배포 — CLI allowlist 막힘과 웹 입력값

JKYMHS-Backend(Gradle·Java 21)를 DSM 학생 PaaS **xquare**에 배포하는 방법. 학생은 kubectl 대신 전용 `xquare` CLI(또는 웹 `xquare.dsmhs.kr`)를 쓴다. CLI는 GitHub 숫자 ID **allowlist**에 없으면 로그인이 막혀서, 이 건은 웹 배포로 전환하고 보류했다. 프로젝트 전반은 [[JKYMHS-Backend/프로젝트-현황]].

## CLI 구조와 사전 조건
- 레포에 학생용 kubeconfig 배포 흐름은 없다. CLI가 xquare 서버 API를 부르고 서버가 k8s를 처리한다. 설치는 `curl -fsSL .../xquare-cli/main/install.sh | sh`(`/usr/local/bin`에 권한이 없으면 `~/.local/bin`), 로그인은 `xquare login`(브라우저 GitHub OAuth).
- 사전 조건 3가지: ① allowlist 등록 ② 조직(JKYMHS)에 **xquare GitHub App 설치**(조직 owner만 가능, 없으면 `app create`가 `GitHub App not installed`+설치 링크) ③ 본인이 레포에 접근 가능.
- 흐름: `project create`(이름은 소문자·숫자만) → `link` → `addon create db mysql --storage 2Gi`(4Gi 미만)·`addon create redis redis` → `app create backend --build-type gradle --java-version 21 --branch develop --endpoint 8080:<이름>.dsmhs.kr`(`--dry-run`으로 미리보기) → `env set` → `trigger --watch`·`logs --build`/`-f`. CLI 기본값 함정: Java **17**(우리는 toolchain 21), 브랜치 **main**(우리는 develop). 빌드는 `./gradlew bootJar -x test`라 Testcontainers 테스트는 안 돈다.

## 로그인 `access not granted`의 원인 (2026-11-10 진단)
- 증상: 브라우저에는 "Authentication successful!"이 뜨지만 CLI는 `access not granted — Your GitHub account has not been added to this platform`. 브라우저 화면은 **GitHub 인증 단계까지만** 성공했다는 뜻이고, 그 뒤 xquare 서버의 allowlist 검사에서 거절된다.
- 서버는 GitHub 아이디가 아니라 **숫자 ID**로 판정하며, 목록은 `team-xquare/xquare-onpremise-project-gitops-repo`의 `allowed-users.yaml`(`main`). 운영진이 `xquare admin allowlist add <id>`를 실행하면 `feat: update allowlist [actor: ...]` 커밋이 생기므로, 그 파일·커밋 유무로 등록 여부를 직접 확인할 수 있다.
- xquare에는 **웹 대시보드(`xquare-infra-backend-v3`)와 CLI용 `xquare-server` 두 시스템**이 있어, 운영진이 웹 쪽에만 등록했을 가능성이 크다("등록했다"는 답에도 계속 실패). 요청할 때 "CLI용 allowlist"라고 명시하고 숫자 ID를 같이 보낸다. 브라우저 프로필(예: 학교 프로필)의 GitHub 계정이 등록 계정과 같은지도 확인.

## 웹 배포 입력값 (xquare.dsmhs.kr)
- 웹은 GitHub allowlist가 아니라 **사이트 자체 계정**(아이디 4~15자, 비밀번호 8~20자+특수문자, 학번 4자리, 한글 이름, 이메일 인증)이라 CLI 막힘과 무관.
- 이름 규칙이 CLI와 다르다: 팀 **영문 소문자 3자 이상**, 애드온 **소문자·하이픈 3자 이상**(CLI에서 쓰려던 `db`는 너무 짧음 → `jkymhs-db`, `jkymhs-redis`), 앱 이름은 소문자·하이픈(숫자 X).
- 앱 생성 칸: Repository `JKYMHS-Backend`, Branch `develop`, 빌드 타입 `gradle`, Version **`21`**(예시 값은 17), Build Command `./gradlew bootJar -x test`, Output Path `/build/libs/*.jar`, URL `<이름>.dsmhs.kr`(**한 단계만**, `a.b.dsmhs.kr` 불가) / 포트 `8080`. 웹에는 기본값이 없어 Version·Build Command·Output Path를 직접 입력. 이후 develop에 push하면 자동 배포(수동 `trigger`를 또 하면 두 번 돈다).
- **MySQL 애드온**: 8.0, `root`에 **비밀번호 없음**(`MYSQL_ALLOW_EMPTY_PASSWORD`), Redis도 인증 없음. 호스트=애드온 이름, DB 이름=애드온 이름의 `-`→`_`(웹 `jkymhs-db`→`jkymhs_db`; CLI에서 `mysql`로 이름 지으면 시스템 스키마와 겹침).

## 환경변수 함정 (application.yml 기준)
- `DB_HOST`·`DB_PORT`·`DB_NAME`·`DB_USERNAME`·`REDIS_HOST`·`REDIS_PORT`·`JPA_SHOW_SQL=false`·`MAIL_*`를 첫 배포 전에 넣는다.
- **`DB_PASSWORD`는 빈 값으로라도 키를 추가**해야 한다. 안 넣으면 기본값 `jkymhs`가 들어가 root 접속이 실패한다. 웹 화면이 빈 값 입력을 막는지는 미확인(서버 DTO에는 검증 없음) — 막히면 `application.yml` 기본값을 손봐야 한다.
- **`JWT_SECRET`은 기본값이 없어** 없으면 서버가 안 뜬다(`openssl rand -base64 48`). 메일 설정을 빼면 기본 `localhost:1025`(Mailpit)로 보내다 실패해 이메일 인증이 502.
- 도메인 예약어: `*-xquare-infra`, `xquare-remote-access-*`, `*-observability-dashboard`, `xquare-server`.
- 실제 데이터가 쌓이면 `ddl-auto: update`를 `JPA_DDL_AUTO=validate` 등으로 바꾸는 것을 고려.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/JKYMHS-Backend/develop-2)
