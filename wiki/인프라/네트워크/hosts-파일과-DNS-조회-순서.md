---
tags: [네트워크, DNS, hosts, 리눅스, systemd-resolved]
updated: 2026-11-20
---

# hosts 파일과 DNS 조회 순서

`/etc/hosts`는 이름→IP를 **사람이 직접 적어 두는 내 컴퓨터 전용 주소록**이다. 프로그램이 이름을 찾을 때 DNS보다 **먼저** 이 파일을 읽고, 적혀 있으면 DNS에 묻지 않는다. 캐시가 아니라 "이 이름은 무조건 이 주소"라는 고정 메모다.

## hosts 파일
- 경로는 `/etc/hosts` (끝에 `s`). 주인이 root라 고치려면 `sudo nano /etc/hosts`.
- 용도: 개발 중 `myapp.test` → `127.0.0.1`로 로컬 서버 접속, 사이트 막기(`youtube.com` → `0.0.0.0`), `127.0.0.1 localhost`(= 내 컴퓨터 자신).
- 이 PC의 내용: `localhost`, `127.0.1.1 yunho-960XHA`(호스트명), IPv6 기본 줄, Docker Desktop이 넣은 `kubernetes.docker.internal`.
- **캐시와 차이**: 캐시는 컴퓨터가 알아서 저장하고 시간이 지나면 지워져 다시 묻는다. hosts는 사람이 적고 지울 때까지 남으며, DNS가 다른 답을 줘도 hosts가 이긴다.
- hosts는 서버가 아니라 **파일**이다. 로컬 DNS 서버보다 앞 단계다.

## 이름 조회 순서 (이 우분투 PC 기준)
1. **`/etc/hosts`**
2. **mDNS** — 같은 와이파이 기기 이름(`.local`)
3. **로컬 DNS `127.0.0.53`** — 우분투의 **systemd-resolved**. 모르면 공유기·통신사 DNS·`8.8.8.8` 등에 넘긴다.

앞 단계에서 답이 나오면 뒤는 건너뛴다.

## 재귀 조회 (예: naver.com)
1. **루트 서버**: ".com은 저기서 물어봐"
2. **TLD 서버**(.com 담당): "naver.com은 저기서 물어봐"
3. **권한 있는(authoritative) 서버**(네이버 것): "naver.com은 이 IP" ← 최종 답

- 이 세 곳을 차례로 도는 건 내 컴퓨터가 아니라 **상위 DNS 서버(리졸버)**다. `127.0.0.53`은 질문을 넘기기만 한다.
- 각 단계 답이 캐시되므로 매번 루트까지 가지 않는다.

## 관련
- [[Deployment-롤아웃과-Service-타입-개념-정리]] — 쿠버네티스 헤드리스 Service의 DNS 기록

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-11-20 (`/etc/hosts`와 조회 순서는 세션 중 이 PC에서 직접 확인한 내용)
