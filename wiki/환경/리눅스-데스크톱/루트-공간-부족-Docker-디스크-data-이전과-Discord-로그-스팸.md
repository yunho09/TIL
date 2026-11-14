---
tags: [리눅스, 디스크, docker, snap, discord, journald, 로그]
updated: 2026-11-15
---

# 루트 공간 부족: Docker 디스크 /data 이전과 Discord 로그 스팸 차단 (2026-10-23)

루트(`/`) 여유가 하루 사이 19G→6.4G로 줄어 정리했다. "디스크 2개"로 알았지만 실제로는 **SSD 하나를 `/`와 `/data`로 나눠 쓰는 구조**였고, 여유가 큰 `/data`(103G)로 Docker Desktop 디스크를 옮겨 `/` 여유를 **6.4G(94%) → 22G(78%)**로 늘렸다. 이어서 Discord가 만드는 보안 차단 로그 스팸도 막았다.

## Docker Desktop 디스크를 /data로 이전

- 대상: `Docker.raw`(8.8G). 옮긴 위치는 `/data/docker-desktop/Docker.raw`.
- 순서: Docker 종료 → 복사 → **원본과 바이트 단위 동일 확인**(sparse 파일이라 빈 부분이 압축돼 8.8G→6.9G) → 설정에서 새 위치 지정 → 재기동 → 정상 확인 후에야 원본 삭제.
- 확인한 것: 쿠버네티스 노드 Ready·파드 9개 Running·이미지 유지, 옛 디스크를 잡고 있는 프로세스 없음.
- **함정**: Docker를 끄면 자동 재시작 설정(`restart` 정책)이 없는 컨테이너(MySQL·Redis 등)는 같이 멈춘다. 재기동 후 컨테이너 목록을 확인해 직접 다시 켜야 한다(MySQL "ready for connections", Redis `PONG`으로 확인).
- 앞으로 Docker 디스크가 커져도 `/`는 차지 않는다. 이전 정리에서는 이 디스크를 유지하기로 했었다 → [[디스크-정리와-부팅-서비스-최적화-점검]]

## 함께 지운 것

- VS Code 안의 yarn 캐시·휴지통, Homebrew·pkg·electron 캐시, Postman 옛 사본·옛 snap(실사용 `~/.local/opt/Postman`은 무관), Spotify 캐시(재생 시 재다운로드로 처음엔 느림).
- 허락 없이 **손대지 않은 것**: JetBrains 캐시, `~/.wine`, 옛 Claude 세션. (사용자 규칙: 지울 것은 먼저 허락을 받는다.)

## Discord 보안 차단 로그 스팸 (snap system-observe)

- 증상: Discord snap이 프로세스 목록을 읽으려다 AppArmor에 막혀 **차단 경고가 1분에 ~120건** → `kern.log`·`syslog`가 4일에 약 180M씩 증가(누적 ~380M).
- 해결: snap에 원래 정의돼 있던 `system-observe` 인터페이스를 연결 — `pkexec snap connect discord:system-observe`. 실행 중인 Discord에도 즉시 적용(재시작 불필요), 연결 후 75초간 차단 경고 0건.
- 부수 효과: Discord가 실행 중 프로그램 목록을 볼 수 있어 "XX 하는 중" 게임 표시가 동작한다.
- 남은 1건은 별개 종류(스레드 우선순위 변경, 하루 ~3건)라 무시 가능.
- 되돌리기: `pkexec snap disconnect discord:system-observe`.
- 이미 쌓인 로그는 주간 logrotate 때 압축돼 줄어든다.

## 2차 이전: 개발 폴더 심볼릭 링크 + 기본 Docker bind mount (2026-11-15)

루트 여유가 다시 **718MB**까지 줄어 개발 쪽을 `/data`(95GB 여유)로 옮겼다. 결과: 루트 **718MB → 22GB(개발 폴더 이동 후) → 28GB(Docker 이동 후)**, `/data`는 95GB → 63GB.

- **개발 폴더(~21GB → `/data/dev-home`)**: 홈의 `.cache/JetBrains`(3.4G)·`.local/share/JetBrains`(2.2G)·`.npm`(2.8G)·`.vscode`(2.6G)·`.config/Code`(1.4G)·`.yarn`(1.4G)·`.bun`(1.1G)·`.codex`·`.local/lib/node_modules`·`.local/opt/Postman`·`.cache/ms-playwright`·`.cache/pip`·`.cache/pkg`·`.cache/Homebrew`·`.omo`·`.jdks`·`.android` 등. 방식은 "복사 → 검증 → 원본을 **심볼릭 링크로 교체**"라 앱 설정을 바꿀 필요가 없다.
- **기본 Docker(containerd 이미지 3.8G + docker 볼륨·컨테이너 3.1G)**: `/data/containerd`, `/data/docker`로 복사·내용 비교 후 원본 삭제. 원래 경로(`/var/lib/containerd`, `/var/lib/docker`)에는 **`/etc/fstab`의 bind mount 두 줄**로 /data 폴더를 연결 → 재부팅 후에도 유지되고 kind 등 Docker 경로를 쓰는 도구는 설정 변경이 없다. fstab 원본은 `/etc/fstab.bak-20261115`로 백업.
- **부팅 순서 함정**: 재부팅 때 `/data`가 마운트되기 전에 Docker가 먼저 뜨면 **빈 폴더로 시작**할 수 있다. 그래서 마운트 이후에만 Docker가 켜지도록 서비스 드롭인(의존성)을 추가했다.
- 검증: 이미지 10·볼륨 7·컨테이너 5로 이전 전후 개수 동일, kind 노드 Ready, JKYMHS MySQL 볼륨 유지.
- 이전 후 루트 사용처(104GB 중 72GB): 홈 28G(앱 설정 6.6G·`.local` 6.6G·snap 앱 데이터 3.9G·Claude 세션 기록 3.9G), snap 앱 16G, `/usr` 9.4G, 스왑 파일 8G, Homebrew 3.3G, `/opt` 3.1G. 남은 후보: snap 앱·스왑 파일.
- `/data` 쪽 후속 후보: **`/data/docker-desktop`(12GB)** — 10월 24일 이후 안 쓰는 예전 Docker Desktop 디스크(지금은 기본 Docker 사용). `/data`가 부족해지면 가장 먼저 볼 곳.
- 관련: [[Docker-Desktop-쿠버네티스-이미지-손상과-기본-Docker-kind-전환]]

## 일반화

- 공간 부족 시 `df`로 파티션 구조부터 본다 — 물리 디스크 수가 아니라 **마운트된 파티션별 여유**가 기준이다.
- 큰 가상 디스크는 "복사 → 바이트 비교 → 새 위치로 기동·검증 → 원본 삭제" 순으로 옮긴다.
- 로그 급증은 `kern.log` 등에서 반복 메시지를 세어(전/후 분당 건수) 원인 앱과 snap 인터페이스 거부(denied)를 찾는다. 관련: [[Discord-영상-로딩-OpenH264-화면공유-뭉개짐]]

## 출처

- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
- Claude Code 세션 자동 캡처 (/home/yunho, 2026-11-15)
