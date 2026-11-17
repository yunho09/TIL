---
tags: [JKYMHS, 배포, xquare, systemd, Spring, Cloudflare, 프록시]
updated: 2026-11-17
---
# 생 컨테이너 직접 배포 — systemd·자동 배포·프록시 뒤 실제 IP

xquare 웹(502)과 CLI(allowlist 누락)가 둘 다 막혀 담당자에게 **생 컨테이너**(Proxmox, Ubuntu, 2코어/4GB/20GB, root)를 받아 JKYMHS 백엔드를 직접 띄웠다. 서버·MySQL·Redis를 한 컨테이너에 동거시키고, develop을 5분마다 따라가는 자동 배포 타이머까지 구성했다. 이전 시도는 [[xquare-배포-CLI-allowlist-막힘과-웹-입력값]], 프로젝트 전반은 [[JKYMHS-Backend/프로젝트-현황]].

## 접속 방식 (비밀번호 전용 서버)
- 컨테이너는 **비밀번호 SSH만** 된다(키 로그인 꺼짐, scp·원격 명령 실행도 막힘). 비밀번호를 채팅에 공유하지 않으려고 `~/.ssh/config`에 `jkymhs` 호스트를 만들고 **ControlMaster**(`ControlPersist=8h`)로 사람이 한 번 로그인한 연결을 Claude가 이어받게 했다.
- 함정: 일반 `ssh host`로 접속하면 공유 소켓이 안 만들어진다. 반드시 config의 `jkymhs`(ControlMaster 옵션 포함)로 접속해야 한다. 기존 config는 백업(`config.bak-날짜`)했고, 원래 있던 `xquare` 호스트 설정(계정 다름)은 건드리지 않았다.

## 서버 구성
```
Cloudflare(HTTPS) → 학교 프록시(10.0.1.10) → 컨테이너 10.0.1.25:8080
jkymhs.service (Spring Boot, 전용 리눅스 계정 jkymhs, 메모리 최대 1GB)
   ├─ MySQL 8.4  127.0.0.1:3306 (DB jkymhs + 전용 사용자)
   └─ Redis 8.0  127.0.0.1:6379
```
- apt로 Java 21·git·MySQL·Redis 설치, 서버 시간대 Asia/Seoul. **DB 비밀번호·JWT 키는 서버에서 랜덤 생성해 환경변수 파일에 직접 기록**(`openssl rand`), 채팅·대화 기록에 값이 남지 않게 했다.
- 파일 위치: 환경변수 `/etc/jkymhs/jkymhs.env`, 실행 jar `/opt/jkymhs/app.jar`, 소스 `/opt/jkymhs/src`, 유닛 `/etc/systemd/system/jkymhs.service`. 부팅 시 자동 시작, 죽으면 5초 뒤 재시작.
- **서버에서 직접 빌드**: scp가 막혀 PC에서 만든 jar를 올릴 수 없고, 레포가 public이라 서버가 `git clone` 가능. `bootJar`가 만든 jar 하나에 코드·라이브러리·내장 Tomcat이 다 들어 있어 Java만 있으면 `java -jar`로 돈다. 빌드 약 1분·메모리 약 1GB(4GB에선 충분).
- 메일(`MAIL_*`)이 비어 있으면 이메일 인증 메일이 안 나가고, 회원가입은 이메일 인증 토큰이 필수라 **메일 설정 전까지 아무도 가입 못 한다**(로그인·재발급·로그아웃 API는 동작). 외부 접속 가능 여부는 서버 안에서 로그인 API를 불러 우리 앱의 401 응답이 오는지, 테이블(`users` 등)이 자동 생성됐는지로 검증.

## 자동 배포 (pull 방식 타이머)
- GitHub는 코드를 보관만 할 뿐 서버에 아무것도 push하지 않는다 — 머지해도 서버는 그대로. 비밀번호 전용·원격 실행 차단이라 GitHub Actions가 접속하는 방식은 불가 → **서버가 스스로 5분마다 develop을 확인**하는 systemd 타이머(`jkymhs-autodeploy`)를 둔다(public 레포라 권한 설정 불필요).
- 흐름: 새 커밋 발견 → `deploy.sh`(fetch/checkout → `./gradlew bootJar -x test` → 기존 jar를 `app.jar.prev`로 백업 → 재시작) → **90초 안에 응답 없으면 prev로 자동 롤백**. 빌드 실패는 재시작 전에 멈춰 기존 서버가 유지된다. 실패한 커밋은 재시도하지 않고 다음 새 커밋을 기다린다. 동시 실행은 락으로 막음. develop만 따라가므로 수동으로 다른 브랜치를 올려도 새 develop 커밋이 생기면 되돌아간다.
- 확인 명령: `cat /opt/jkymhs/DEPLOYED`(현재 버전), `journalctl -u jkymhs-autodeploy`, `journalctl -u jkymhs -f`, 수동은 `/opt/jkymhs/deploy.sh`.
- 검증 범위: 강제 1회 실행(23초)·외부 정상 응답·타이머 자체 실행(새 커밋 없어 무동작)은 확인. **롤백 경로는 고장 난 버전을 만들어 시험하지 않았고 문법 검사만 통과**.
- 위험: 서버 빌드는 `-x test`이고 레포에 CI(`.github/workflows`)가 없어 **테스트가 어디서도 안 돈다** → 버그 PR이 머지되면 그대로 배포됨(빌드는 되는데 기동 중 죽으면 서버 중단). PR 전 로컬 `./gradlew test` 또는 PR 테스트용 Actions 필요.
- 관광지 PR(#18)이 머지되면 새벽 4시 TourAPI 동기화가 켜져 로컬 수집과 **하루 1,000회 키 한도를 나눠 쓰게 되므로**, 서버에 `BATCH_ENABLED=false`를 넣어 뒀다(수집 완료 후 true). 이후 PR의 새 환경변수는 기본값이 있어 기동은 막히지 않으며, FE 주소가 정해지면 `CORS_ALLOWED_ORIGINS`·`FRONTEND_BASE_URL`만 넣으면 된다.

## 도메인: 서브도메인으로 HTTPS 얻기
- 도메인을 살 필요 없이 담당자 도메인 `dsmhs.kr`의 하위 주소를 받는다(와일드카드가 Cloudflare 경유 HTTPS로 응답). 기대하던 `https://edgl.dsmhs.kr`로 연결됨(FE 레포 PRD의 서비스명 "어디 갈래(EDGL)" 약자). 백엔드가 `edgl`을 쓰므로 FE는 `edgl-web` 등 다른 이름으로 요청해야 한다.
- 포트 포워딩(`xquare.xyz:포트`)만 받으면 HTTPS가 없어 **HTTPS 프론트에서 호출 시 브라우저가 혼합 콘텐츠로 차단**한다(Postman·Swagger 테스트용으로만 충분).
- 단점: `http://` 접속이 HTTPS로 리다이렉트되지 않고 그대로 처리된다 → Cloudflare "Always Use HTTPS"는 담당자에게 요청.

## 프록시 뒤에서 모든 요청이 한 IP로 보이는 문제
- 처음엔 모든 요청이 실제 사용자 IP가 아니라 중간 프록시 IP(`10.0.1.10`)로 기록됐다. 로그인(1분 20회)·인증 메일(1분 5회) 제한이 IP 기준이라 **전체 사용자가 한도 하나를 공유**하게 된다.
- 해결: 서버 환경변수 `SERVER_FORWARDHEADERSSTRATEGY=native`(= `server.forward-headers-strategy: native`) 추가 후 재시작. 코드는 그대로. 실제 IP(`14.50.190.128`)가 기록되고, **요청에 가짜 `X-Forwarded-For`를 넣어도 무시되는 것**(스푸핑 불가)까지 시험으로 확인. 다른 환경에서도 똑같이 하려면 `application.yml`에 넣는 선택지가 있으나 코드 변경이라 사용자 결정으로 남겼다. 관련 미결 항목은 [[인증-API-구현과-PR-리뷰-결정사항]]의 "프록시 IP".

## MySQL·Redis 구성과 포트 구조
- 둘 다 **apt 직접 설치**(Docker 아님, 4GB라 Docker 메모리도 아낌), systemd가 부팅 자동 시작·재시작 관리. 앱은 `/etc/jkymhs/jkymhs.env`의 값이 `application.yml`의 `${DB_HOST:localhost}` 자리에 채워져 접속한다. 메모리: Spring ~460MB, MySQL ~520MB, Redis ~14MB.
- **MySQL**: 데이터 `/var/lib/mysql`(컨테이너 디스크라 재시작해도 유지), `127.0.0.1:3306`만 수신. 계정 둘 — `root`(비밀번호 없음, `auth_socket`이라 서버에 root로 들어온 사람만 `mysql jkymhs`로 접속)와 `jkymhs`(랜덤 비밀번호, `jkymhs` DB 권한만 → 앱 계정이 털려도 다른 DB는 못 건드림). 테이블은 SQL이 아니라 `JPA_DDL_AUTO=update`로 Hibernate가 시작 시 `@Entity`를 보고 생성. **`update`는 추가만** 하므로 필드 삭제·타입 변경은 DB에 반영되지 않는다.
- **Redis**: `127.0.0.1:6379`만 수신, 비밀번호 없음(외부 접근 불가라 기본값 유지), RDB 스냅샷(1시간 1개/5분 100개/1분 1만 개 변경 시 `/var/lib/redis/dump.rdb`). 갑자기 죽으면 마지막 저장 이후 값이 사라질 수 있으나 넣는 건 인증 코드·IP별 요청 횟수 같은 **몇 분짜리 값뿐**이고 장기 보관 대상인 리프레시 토큰은 MySQL(`refresh_tokens`)에 있어 문제없다.
- **포트로 구분**: Spring 8080은 `*`(전체)라 프록시를 거쳐 외부에서 들어오고, MySQL·Redis는 `127.0.0.1`(자기 자신)이라 컨테이너 안의 Spring만 접속 가능. 외부에 열린 건 SSH(`xquare.xyz:2222`→22)와 API(`edgl.dsmhs.kr`→8080) 둘뿐. 도메인 하나로 기능을 나누는 건 포트가 아니라 **URL 경로**(`/api/auth/login` 등, Spring이 `@PostMapping`으로 분기). DB·Redis는 도메인으로 접근하는 대상이 아니라 Spring이 안에서 대신 접속한다.
- **PC에서 DB 보기**: `ssh jkymhs` 후 `mysql jkymhs`뿐. IntelliJ 등에서 쓰려고 SSH 터널(포트 포워딩)을 시험했으나 **담당자 쪽 SSH가 터널까지 막아 연결 실패**.
- 백업 없음 — 사용자가 생기면 `mysqldump` 일 1회 타이머 필요.

## AI 추천 연동: AI 서버는 DB 접근이 필요 없다 (PR #30 코드 기준)
- DB 접근이 막혀 있어도 문제없는 이유: Spring이 MySQL에서 후보 장소를 조회·점수순 정리(`CandidateService`)해 **요청 본문에 담아** AI 서버로 보내고, AI는 받은 후보만으로 코스를 짠다. 응답은 Spring이 검증·저장.
- 계약: `POST {AI_BASE_URL}/v1/courses`, 헤더 `Authorization: Bearer`(선택), 응답 대기 20초(초과·에러 시 백엔드 기본 코스로 대체). 요청은 `conditions`(날짜·동행·페이스·실내외·출발지 좌표), `profile`, `candidates`(contentId·좌표·영업시간·`restDays`·축제기간·`required`), `rules`(코스 3개·하루 2~3곳·점심/저녁 시간·`noPlaceAfter`), `violations`. 응답은 `courses[].days[].items[]`(`contentId`·`stayMinutes`·`reason`).
- 규칙: `contentId`는 받은 후보 안에서만, `required: true`는 반드시 포함, 영업시간·휴무·축제 기간 준수. 백엔드가 위반을 `violations`에 담아 **한 번 더 재요청**하고 그래도 틀리면 기본 코스로 대체. `stayMinutes`가 이상하면 90분으로 보정.
- 기본값 `AI_PROVIDER=mock`(AI 서버 없이 Spring이 간단한 코스 생성). 실제 서버는 (a) AI 담당이 따로 배포해 `AI_BASE_URL=https://…`(컨테이너는 외부 HTTPS 호출 가능) 또는 (b) 같은 컨테이너에 `http://localhost:8000`. (b)는 외부 AI API 호출형이면 가볍지만 **모델을 직접 돌리면 4GB(이미 ~1GB 사용)로 부족**. 미정이라 AI 팀원에게 배포 위치·Bearer 사용·형식·20초 가능 여부를 질문. 형식은 PR #30 미머지 상태라 리뷰 중 바뀔 수 있다.

## 현재 한계 (2026-11-17)
- 서버에는 develop `8a61d53`(PR #7·#8·#9·#11·#12, 회원 인증까지)만 배포. 관광지·탐색·찜 등 나머지 PR(#14·#16·#18…)은 미머지. DB는 비어 있음(테이블 3개 0건, Redis 키 0).
- 모아 둔 관광지 CSV는 PC에만 있음. 올리려면 PR #18 머지·배포 후 CSV를 압축해 SSH 터미널로 흘려보내고 `TOUR_IMPORT_CSV_DIR` 지정 후 재시작(scp 막힘). 
- DB 백업 없음(컨테이너 디스크에만 존재), 메일 미설정, HTTP→HTTPS 리다이렉트 없음.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/JKYMHS-Backend/develop-2)
