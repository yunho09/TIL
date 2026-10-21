---
tags: [mysql, apt, systemd, 리눅스]
updated: 2026-10-21
---

# MySQL 로컬 설치와 온오프 명령

PC에 apt로 MySQL 8.4 서버를 직접 설치하고, 부팅 시 자동 시작은 꺼 둔 채 `mysql-on` / `mysql-off` 명령으로 필요할 때만 켜 쓰는 구성. 켜져 있는 동안 메모리를 약 520MB 쓰기 때문에 상시 구동 대신 수동 토글을 택했다.

## 구성
- 설치: apt의 **MySQL 8.4** 서버 (설치 중 비밀번호 창이 두 번 뜸 — 계정 생성, 자동 시작 끄기)
- 서비스 이름: `mysql` (systemd). **부팅 시 자동 시작 비활성** → 재부팅하면 항상 꺼진 상태
- 앱·DataGrip 접속: `127.0.0.1:3306`, 사용자 `yunho`, 이 컴퓨터(로컬호스트)에서만 접속 가능, 권한 전체
- 비밀번호는 `~/.my.cnf`에 저장(본인 계정만 읽기 가능) → 터미널에서 `mysql`만 치면 비밀번호 없이 접속
- **root는 Ubuntu 기본 방식(auth_socket) 그대로** → `pkexec mysql`로만 접속

## 켜고 끄기
- 켜기: `mysql-on` → `active`가 나오면 켜진 것, 이어서 `mysql`
- 끄기: `mysql-off`
- 상태 확인: `systemctl is-active mysql` (`active` / `inactive`)
- 둘 다 비밀번호를 묻지 않음 (구현 방식은 세션에서 확인 못 함)
- **재부팅 뒤 접속이 거부되면** 서버가 꺼진 상태일 뿐이니 `mysql-on`부터

## 기존 Docker 방식과의 관계
- Spring 학습 프로젝트는 Docker 컨테이너로 MySQL을 쓴다 → [[MySQL-연동]], [[프로젝트/spring-practice/프로젝트-현황]]
- 이쪽은 호스트에 직접 설치한 별개 서버이며 포트 3306을 공유하므로, **둘을 동시에 켜면 포트가 충돌**할 수 있다 (Claude 보충)

## 출처
- [[raw/Claude Code 세션 자동 캡처 (/home/yunho)]]
