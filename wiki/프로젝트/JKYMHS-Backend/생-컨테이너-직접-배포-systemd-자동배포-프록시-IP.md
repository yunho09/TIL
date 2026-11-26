---
tags: [JKYMHS, 배포, xquare, systemd, Spring, Cloudflare, 프록시]
updated: 2026-11-26
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

## 같은 컨테이너에 DB·Redis를 두는 구조 vs 분리
- 학생·초기 프로젝트에선 흔한 구성. 장점은 설정 단순·비용 0·통신이 컨테이너 안이라 빠름. 단점은 **컨테이너가 죽으면 셋 다 멈춤**, 메모리 공유, 백업 직접 관리. 서비스가 커지면 RDS·ElastiCache 같은 관리형이나 별도 서버로 분리(재배포와 DB 분리, 자동 백업, 개별 증설). xquare PaaS였다면 MySQL·Redis가 별도 "애드온"이었을 것.
- 셋은 서로 **독립된 프로그램**(각자 systemd로 켜고 끔)이고 Spring이 `localhost`로 쓰는 쪽일 뿐 — Spring을 재배포해도 MySQL·Redis와 데이터는 그대로.
- 담당자에게 컨테이너를 요청할 때 넣을 것: 용도, 사양(2vCPU/4GB/20GB), OS, 접속 방식, 외부 공개 포트, GitHub 계정, **영구 디스크**(임시 저장소면 재시작 때 DB가 날아감).

## 스택 PR 12개 머지와 관광지 데이터 적재 (2026-11-17~18)
- 열려 있던 스택 PR 12개(#10→#33→#14→#16→#18→#21→#22→#25→#26→#30→#29→#34, API 6개→71개)를 머지하고 develop `868b41b`로 배포. **머지 전에 12개를 같은 순서로 로컬에서 합쳐 전체 테스트(627개) 통과를 확인**했고, 머지 후 develop 파일 내용이 그 결과와 동일함을 대조. 머지 방식은 merge commit.
- **스택 PR 머지 함정**: #14를 머지하며 브랜치를 먼저 지우면 그 브랜치를 base로 둔 다음 PR(#16)이 **자동으로 닫힌다**(재오픈 후 base를 develop으로 변경해 해결). 순서는 "다음 PR의 base를 develop으로 옮긴 뒤 브랜치 삭제". 또 로컬 git은 충돌이 없는데 GitHub가 충돌로 표시하는 PR(#14·#16)은 develop 머지 커밋을 브랜치에 추가("Update branch"와 동일)해서 풀었다. 관련 [[남은-API-스택-PR-구성과-개발환경-Docker]], [[Git-스택-PR-재정렬-force-with-lease]].
- 머지 중에 중간 버전이 배포되지 않도록 **자동 배포 타이머를 잠깐 멈추고**, 끝난 뒤 다시 켰다.
- **CSV 서버 이전(scp 차단 우회)**: 압축 후 조각(8개)으로 나눠 SSH 터미널로 흘려보내고 서버에서 체크섬 대조·합친 뒤 해제, `contents.csv` 행 수까지 확인. 이후 `TOUR_IMPORT_CSV_DIR`로 기동 시 적재 → 지역 284·분류 315·관광지 32,547(좌표 없는 1건 제외)·상세 6,647·테이블 31개. **적재 후에는 `TOUR_IMPORT_CSV_DIR`를 주석 처리**(켜 두면 재시작마다 다시 적재), 새 CSV를 올릴 때만 켠다.
- **`BATCH_ENABLED=false` 정정**: 이 값은 TourAPI뿐 아니라 인기 점수·추천 정리·캘린더 동기화 배치까지 **전부** 끈다. 그래서 값을 빼고, 대신 서버에 **TourAPI 키를 넣지 않아** 동기화가 안 돌게(로컬 수집과 일일 1,000회 한도 공유 방지) 했다. 인기 점수는 *우리 서비스 사용자의* 최근 30일 조회·찜·일정 담기로 계산하므로 사용자가 없으면 전부 0점 → 사용자가 생기기 전엔 인기순 정렬이 무의미(목록 맨 앞이 임의 관광지로 보이는 이유).
- 외부에서 확인한 공개 API(로그인 불필요): `/api/regions`, `/api/categories`, `/api/travel-styles`, `/api/contents`(목록·상세·자동완성·주변). 출발지 검색 `/api/places/...`와 나머지는 로그인 필요(토큰 없이 401).

## AI가 DB를 직접 쓰게 할 수 있나 — 비교
| | 백엔드가 후보를 넘김(현재) | AI가 DB 직접 접속 |
|---|---|---|
| 보안 | DB를 외부에 안 열어도 됨 | AI가 같은 컨테이너에 있거나 DB를 외부 공개해야 함 |
| AI 서버 | 받은 JSON만 처리해 단순 | SQL·테이블 구조를 알아야 함 |
| 검증 | 백엔드가 응답 검증(없는 장소·영업시간) | 검증 위치가 모호 |
| 한계 | 백엔드가 고른 상위 40곳 안에서만 선택 | 전체 데이터 접근 |
- 결론: 현재 방식이 이미 "DB 데이터 활용"이다(AI가 DB를 안 볼 뿐 쓰는 데이터는 DB 출처: 관광지 정보+회원 취향). 더 필요하면 후보 수·전달 필드(설명·요금·주소는 현재 미전달)를 늘린다.

## 팀원이 받는 별도 인스턴스와 백엔드 연결 (2026-11-18)
- **인스턴스** = 우리가 받은 것과 같은 종류의 가상 서버 한 대(Proxmox 위의 컨테이너, AWS EC2 인스턴스와 같은 개념). 각 인스턴스는 **완전히 독립**이라 한쪽이 죽거나 재시작해도 다른 쪽에 영향이 없고 메모리 4GB도 각자 쓴다. 받은 직후 상태는 아마 우리와 같다: Ubuntu·root·비밀번호 SSH, 런타임(Python 등)은 직접 설치, **외부 접속(도메인·포트)은 담당자에게 따로 요청**해야 한다. (Claude 추정 — 팀원 인스턴스는 아직 못 봄)
- AI 서버가 다른 인스턴스에 있으면 백엔드가 네트워크로 호출한다:

  | 방법 | 설정 | 장단점 |
  |---|---|---|
  | 내부 IP 직접(추천) | `AI_BASE_URL=http://10.0.1.??:8000` | 도메인 불필요·빠름·외부에 비공개. 인스턴스 간 통신이 허용되는지 확인 필요 |
  | 도메인 | `AI_BASE_URL=https://edgl-ai.dsmhs.kr` | 어디서든 호출 가능하나 담당자에게 도메인 연결 요청 필요, 누구나 부를 수 있으니 **API 키 필수** |

  두 인스턴스가 같은 `10.0.1.x` 대역이라 내부 통신이 될 가능성이 높지만, 담당자가 막아 뒀을 수 있다 → 팀원이 받으면 내부 IP(`ip -4 addr show eth0`)를 공유받아 백엔드 서버에서 실제 연결을 시험한다.
- **FE를 인스턴스에 올리면 도메인 필수**(브라우저가 직접 접속). `edgl`은 백엔드가 이미 쓰므로 `edgl-web.dsmhs.kr` 같은 별도 이름이 필요하고, 카카오맵 키 등록·백엔드 CORS에도 그 주소가 들어간다.
- FE 개발 중에는 Vite 프록시 대상을 `https://edgl.dsmhs.kr`로 바꾸면(`changeOrigin: true` 필요) 배포된 백엔드에 바로 붙여 볼 수 있다. 로그인 없이 되는 API는 `/api/regions`·`/api/categories`·`/api/travel-styles`·`/api/contents`뿐이고, 가입은 SMTP 설정 전까지 불가.

## 현재 한계 (2026-11-18)
- 서버는 develop `868b41b`(12개 PR 전부, API 71개), DB에 관광지 데이터 적재 완료. 상세 정보는 수집 진행분(16,190곳 중 6,647곳)만 있고 이후 CSV를 다시 올려야 갱신된다.
- **메일(SMTP) 미설정으로 회원가입 불가** → 로그인 필요 API(마이페이지·찜·여행 계획·AI 추천·캘린더) 테스트 불가가 1순위 과제. 그 외 미설정 외부 키: Google OAuth, Kakao REST(출발지 검색·자동차 경로), ODsay(대중교통).
- AI는 mock 모드(AI 담당의 배포 위치 답변 후 `AI_PROVIDER=http`, `AI_BASE_URL`). FE 도메인이 정해지면 `CORS_ALLOWED_ORIGINS`·`FRONTEND_BASE_URL`.
- DB 백업 없음(컨테이너 디스크에만 존재), HTTP→HTTPS 리다이렉트 없음.

## systemd 서비스란 (2026-11-26)
- **systemd**는 리눅스에서 프로그램을 대신 켜고 지켜보는 관리자, 등록한 프로그램 하나가 "서비스". SSH에서 `java -jar`를 직접 치면 창을 닫을 때 같이 꺼질 수 있고, 죽거나 재부팅돼도 다시 켜지지 않는다.
- `jkymhs.service`의 핵심 줄: `Requires=mysql.service redis-server.service`(의존), `User=jkymhs`(root 아닌 전용 계정), `EnvironmentFile=/etc/jkymhs/jkymhs.env`, `ExecStart=/usr/bin/java -Xms256m -Xmx1g -jar /opt/jkymhs/app.jar`, `Restart=on-failure`+`RestartSec=5`, `WantedBy=multi-user.target`(부팅 자동 실행).
- (Claude 보충) `Requires=`만으로는 시작 순서가 정해지지 않는다. 순서는 `After=`가 정한다(실제 유닛의 `After=` 유무는 기록에 없음).
- 명령: `systemctl status|restart|stop jkymhs`, 로그 `journalctl -u jkymhs -f`.

## SSH 접속 ID와 팀원에게 접근 주기 (2026-11-26)
- `ssh <ID>@xquare.xyz -p 2222`의 `@` 앞은 리눅스 계정이 아니라 **인스턴스 번호**다. 게이트웨이가 번호를 보고 그 인스턴스에 **root**로 연결한다(게이트웨이 구조: [[xquare-infra-frontend-v3/SSH-게이트웨이-VM-배포-결정과-self-hosted-runner]]).
- 일반 서버는 `@` 앞이 계정 이름이라 `useradd` 후 그 이름으로 권한이 제한된 접속이 가능하지만, 여기서는 인스턴스 안에 계정을 만들어도 게이트웨이로 들어올 수 없다.
- 접속 정보를 공유하면 팀원은 Spring 중지·삭제, `jkymhs.env`의 DB 비밀번호·JWT 키 열람, MySQL root로 회원 데이터 열람까지 가능하다.
- 선택지: ① 믿는 팀원이면 **비밀번호를 새로 바꾼 뒤** root 정보 공유 ② xquare에 "root가 아닌 팀원용 SSH 접속을 추가할 수 있나" 문의(지원 여부 미확인) ③ 내가 대신 배포 ④ GitHub Actions 자동 배포(처음 설정 필요, 이후 서버 접속 불필요). 인스턴스 22번 포트 직접 접속은 내부망 주소로만 보여 막혀 있을 가능성이 크다(미확인).

## 상충하는 정보
- 위 "AI 서버는 DB 접근이 필요 없다"(PR #30 기준)와 달리, 2026-11-26에는 AI용 스키마 `jkymhs_ai`를 따로 두는 방향을 논의했다(미생성). 그러면 AI 서버가 다른 인스턴스에 있는 방안은 DB를 쓸 수 없다. 상세는 [[AI-스키마-분리와-view-데이터-계약]].

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/JKYMHS-Backend/develop-2)
- Claude Code 세션 자동 캡처 (/work/code/project/JKYMHS-Backend)
