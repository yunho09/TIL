---
tags: [mysql, apt, systemd, 리눅스]
updated: 2026-10-29
---

# MySQL 로컬 설치와 온오프 명령

PC에 apt로 MySQL 8.4 서버를 직접 설치하고, 부팅 시 자동 시작은 꺼 둔 채 `mysql-on` / `mysql-off` 명령으로 필요할 때만 켜 쓰는 구성. 켜져 있는 동안 메모리를 약 520MB 쓰기 때문에 상시 구동 대신 수동 토글을 택했다.

## 구성
- 설치: apt의 **MySQL 8.4** 서버 (설치 중 비밀번호 창이 두 번 뜸 — 계정 생성, 자동 시작 끄기)
- 서비스 이름: `mysql` (systemd). **부팅 시 자동 시작 비활성** → 재부팅하면 항상 꺼진 상태
- 앱·DataGrip 접속: `127.0.0.1:3307`(2026-10-29 이전엔 3306), 사용자 `yunho`, 이 컴퓨터(로컬호스트)에서만 접속 가능, 권한 전체
- 비밀번호는 `~/.my.cnf`에 저장(본인 계정만 읽기 가능) → 터미널에서 `mysql`만 치면 비밀번호 없이 접속
- **root는 Ubuntu 기본 방식(auth_socket) 그대로** → `pkexec mysql`로만 접속

## 켜고 끄기
- 켜기: `mysql-on` → `active`가 나오면 켜진 것, 이어서 `mysql`
- 끄기: `mysql-off`
- 상태 확인: `systemctl is-active mysql` (`active` / `inactive`)
- 둘 다 비밀번호를 묻지 않음 (구현 방식은 세션에서 확인 못 함)
- **재부팅 뒤 접속이 거부되면** 서버가 꺼진 상태일 뿐이니 `mysql-on`부터
- `mysql-off` 때 "mysql.service 유닛 파일이 바뀌었으니 `systemctl daemon-reload` 하라"는 경고가 뜰 수 있다. 끄는 데는 영향 없고, 한 번 `daemon-reload` 해 두면 사라진다.
- 메모리가 빠듯할 때(스왑 포화, OOM으로 GNOME 세션이 통째로 꺼질 위험) 가장 손쉽게 확보하는 수단이 `mysql-off` — 관련 배경은 [[절전-복귀-지연-원인과-zram-도입]]

## 기존 Docker 방식과의 관계
- Spring 학습 프로젝트는 Docker 컨테이너로 MySQL을 쓴다 → [[MySQL-연동]], [[프로젝트/spring-practice/프로젝트-현황]]
- 이쪽은 호스트에 직접 설치한 별개 서버다. 처음엔 둘 다 3306을 써서 **충돌**했다 → 아래 "포트 충돌과 3307 이전"

## 포트 충돌과 3307 이전 (2026-10-29)
- **증상**: `mysql-on`을 해도 MySQL이 안 켜지고 재시작만 반복. `ss -ltn`으로 3306을 보니 JKYMHS용 Docker MySQL(`jkymhs-mysql`)이 이미 점유 — 한 포트는 프로그램 하나만 쓸 수 있어 늦게 뜬 로컬 쪽이 실패.
- **해결**: `/etc/mysql/mysql.conf.d/zz-yunho-port.cnf`에 `port = 3307` 한 줄(시스템 폴더라 sudo 비밀번호 창). MySQL은 이 폴더의 설정을 이름 순으로 읽고 **나중 값이 이기므로** `zz-` 접두로 맨 마지막에 읽히게 했고 원래 설정 파일은 건드리지 않음. `~/.my.cnf`에도 `port=3307` 추가.
- **결과**: 로컬 3307 + Docker 3306 동시 가동. 앞으로 Bium은 3307, JKYMHS는 3306.

## DataGrip·앱 연결 원리
- 연결이 되려면 ① 서버 켜짐 ② host+port 일치 ③ 그 서버에 계정 존재 ④ 그 서버에 DB 존재, 넷이 모두 맞아야 한다. 실패 유형: 꺼짐 → `Connection refused`, 계정/비번 오류 → `Access denied for user`, DB 없음 → `Unknown database`.
- 이번 DataGrip 실패 원인: 연결이 3306(Docker)을 향해 `yunho` 계정이 없어 로그인 단계에서 막힘. Port 3307 + Database 입력으로 해결. DataGrip 설정은 `~/DataGripProjects/default/.idea/dataSources.xml`에서 확인 가능.
- **MySQL 서버는 백엔드와 별개 프로세스**이고 DataGrip·백엔드는 둘 다 그 서버에 붙는 클라이언트다. 백엔드가 꺼져 있어도 MySQL만 켜져 있으면 DataGrip으로 볼 수 있고, 테이블·데이터는 디스크에 남는다. DB는 이 노트북에만 있어 팀원 PC의 데이터는 보이지 않는다.
- DataGrip의 `bium@localhost`는 **표시 이름**(`DB이름@host` 자동 생성)일 뿐 연결에 쓰이지 않아 자유롭게 Rename 가능. 실제로 중요한 것은 host·port·계정·DB 이름. 서버 구분은 포트, 서버 안의 DB 구분은 DB 이름(`jdbc:mysql://localhost:3307/bium`은 컴퓨터→서버(포트)→DB 순서).
- Spring 백엔드 연결: `.env`의 `DB_URL`/`DB_USERNAME`/`DB_PASSWORD` → `application.yml`의 `${DB_URL:기본값}`(`:` 뒤는 기본값, 여기선 3306이라 3307을 따로 지정) → `spring-boot-starter-data-jpa`(HikariCP 풀) + `mysql-connector-j`가 자동 연결. 로그 `HikariPool-1 - Start completed.`가 연결 성공 신호. `ddl-auto: update`면 첫 기동 때 Entity를 보고 테이블을 자동 생성(Bium은 18개) — [[Bium/프로젝트-현황]], [[MySQL-연동]]
- Bium 백엔드는 `JWT_SECRET`이 비어 있으면 서버가 뜨지 않아 로컬용 임의값(`openssl rand`)을 `.env`에 넣었다(.env는 Git 제외).

## 출처
- [[raw/Claude Code 세션 자동 캡처 (/home/yunho)]]
