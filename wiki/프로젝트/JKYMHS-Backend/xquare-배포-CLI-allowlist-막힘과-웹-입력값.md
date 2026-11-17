---
tags: [xquare, 배포, CLI, 웹, JKYMHS, Spring]
updated: 2026-11-17
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

## 웹 승인 구조와 배포 순서 (2026-11-16, `xquare-infra-backend-v3` 소스 확인)
- 별도 컨테이너를 받을 필요는 없다. 웹에서 팀·앱을 직접 만들고 **앱 승인**만 받으면 된다.
- 승인 필요 여부: **팀** 불필요 / **애드온(MySQL·Redis)** 불필요(만들면 바로 배포) / **애플리케이션** 필요 — 처음엔 `PENDING`, 관리자가 `PUBLISHED`로 바꿔야 실제 배포.
- DB와 서버는 같은 팀 안에서 **따로 만들어지고 따로 돈다**. 순서: ① 팀 ② 애드온(MySQL·Redis — 서버가 시작 때 붙으므로 먼저) ③ 앱+환경변수 ④ 담당자 승인 요청. DB는 한 번 만들면 계속 떠 있고, 이후 develop push 때마다 서버만 재배포.
- 환경변수 보충: `DB_USERNAME`은 `root`로 넣어야 한다(기본값 `jkymhs`). Redis 애드온도 따로 만들어 `REDIS_HOST`를 맞춘다. PR #7~#9 머지로 환경변수가 늘어 DB만 설정해서는 앱이 안 뜬다.

## 웹 백엔드 502 → CLI가 유일한 경로 (2026-11-16 실측)
- `xquare.dsmhs.kr`는 대시보드 대신 "prod ok"만 표시, `infra-backend(-stag).dsmhs.kr`(웹 백엔드)는 둘 다 **502**라 화면이 떠도 로그인·생성 불가. CLI 서버 `xquare-server.dsmhs.kr`만 정상. 일시적 장애일 수 있어 재확인 필요.
- 이 날도 `allowed-users.yaml`에 숫자 ID `207695734`가 없었고, 파일 마지막 변경이 **6월 15일**이라 "등록했다"는 말은 이 파일에 반영된 적이 없다(웹 쪽 계정 등록으로 추정). 과거 추가는 in-jun·kimsaram32 계정이 해왔다.

## 상충하는 정보
- allowlist 추가 명령: 위 진단 절은 `xquare admin allowlist add <id>`, 11-16 세션은 `xquare server allowlist add <githubId>`. 어느 쪽이 맞는지 미확인 — 운영진에게 "CLI용 allowlist 추가"로 요청하면 된다.
- CLI는 웹과 달리 관리자 승인 없이 바로 배포될 가능성이 있다고 봤으나 CLI 코드로는 확인하지 않았다.
- 대안으로 "생 컨테이너 1개(서버+MySQL+Redis 동거, SSH·8080 공개)" 할당 요청을 검토했으나 담당자 답이 없어 결정되지 않았다. 사양 2vCPU/4GB/20GB는 (Claude 보충) 추정치. 요청문에는 용도·사양·OS(Ubuntu)·SSH·8080 공개·도메인 연결에 더해 **디스크를 영구 저장소로** 요청하는 줄을 넣기로 했다 — MySQL을 컨테이너 안에서 돌리면 임시 디스크는 재시작 때 DB 데이터가 날아간다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/JKYMHS-Backend/develop-2)
