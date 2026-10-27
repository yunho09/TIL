---
tags: [docker, kubernetes, kind, kubectl, docker-desktop, 리눅스]
updated: 2026-10-27
---

Docker Desktop 내장 쿠버네티스로 로컬 클러스터를 만들었더니, 이미지 다운로드가 중간에 끊긴 뒤 **내부 이미지 3개의 바이너리가 잘린 채로 남아** LoadBalancer가 `<pending>`에 멈췄다. 이후 가상 컴퓨터(VM) 층이 없는 **기본 Docker Engine + kind**로 갈아타 메모리를 약 3.9GB → 1GB로 줄였다. 두 구성의 차이와 함정을 정리한다. (2026-10-24 작업)

## Docker Desktop 내장 쿠버네티스 (kind 모드)

- 노드 1개짜리 클러스터(`desktop-control-plane`), kubectl 컨텍스트 `docker-desktop`. 내부적으로 kind(Kubernetes IN Docker) 방식이라 노드가 컨테이너다.
- 설정은 `~/.docker/desktop/settings-store.json`의 `KubernetesEnabled`, `MemoryMiB`, `DataFolder` 등. 리눅스에서도 qemu **VM 안에서** Docker가 돌아 메모리를 많이 먹는다(상한을 7.7GB→4GB로 낮춤).
- kubectl과 서버 버전은 ±1 마이너까지 지원(kubectl 1.37.1 ↔ 서버 1.36.1 정상).
- LoadBalancer 서비스는 `cloud-provider-kind`+envoy가 호스트 `localhost:<포트>`로 열어 준다. `EXTERNAL-IP`(172.18.x)는 VM 내부 주소다.

## 문제: 끊긴 pull 뒤 이미지 파일이 잘림

- 첫 시작 실패 원인: `pulling image ... httpReadSeeker ... EOF` (Desktop VM을 통한 큰 파일 pull만 끊김, 네이티브 엔진은 정상).
- 미리 pull해 재시도하면 **성공한 것처럼 보이지만** containerd 스냅샷에 파일이 잘려 있었다.
  - `kind-cloud-provider` 컨테이너: `exec format error`로 크래시 루프 — 바이너리가 **0바이트**.
  - envoy, registry-mirror 바이너리: 정확히 **24MiB(25,165,824바이트)**에서 잘림(정상은 약 86MB).
  - kindest/node 이미지는 멀쩡했다.
- 진단: 같은 이미지를 네이티브 엔진으로 받아 `docker export` tar의 파일별 sha256을 비교. VM 네트워크 자체는 60MB 다운로드 5/5 정상이라 **끊긴 순간의 잔재**로 판단.
- 해결: 숨은 시스템 컨테이너 `docker rm -f` → `docker rmi` → 재pull → 네이티브 사본과 4개 이미지 전부 일치 확인 → Desktop 재시작. 그 뒤 `localhost:80` 200.
- 교훈: LoadBalancer가 `<pending>`이면 `kind-cloud-provider` 컨테이너 로그부터 본다. "pull 재시도 성공"을 정상으로 믿지 말고 파일 크기·해시를 비교한다.

## Docker Desktop 사용 함정

- 터미널의 `docker desktop start`는 Claude Desktop이 띄운 `claude-cowork-vm`(qemu)을 자기 VM으로 오인해 "already running"만 뜨고 안 된다(`stop`도 실패). 앱 메뉴/`/opt/docker-desktop/bin/docker-desktop` 또는 `systemctl --user start docker-desktop`으로 켠다.
- 열려던 Compose 프로젝트가 없어도 Desktop이 마지막으로 보던 베타 "Compose file viewer" 화면을 복원하다 `Could not open compose project`를 띄울 수 있다. compose.yaml이 정상이면(`docker compose config`) 무해하고, 왼쪽 메뉴 Containers로 나가면 된다.
- Update 버튼은 재시작을 부르므로, 중요한 시점 전에는 누르지 않는다(이미지 pull 문제 재발 우려).
- 컨텍스트는 `desktop-linux`(Desktop) vs `default`(기본 엔진). Desktop이 켜지면 `~/.docker/config.json`의 `currentContext`를 `desktop-linux`로 바꾼다. 돌아올 때 `docker context use default`.

## 기본 Docker + kind로 전환

| | Docker Desktop | 기본 Docker + kind |
|---|---|---|
| Docker | VM 안에서 실행 | VM 없이 직접 실행 |
| 클러스터 | `docker-desktop` | `kind-kind`, 노드 `kind-control-plane` |
| 메모리 | VM 3.9GB | 전부 약 1GB (kind 노드 542MiB, mysql 424MiB) |
| LoadBalancer | `localhost`로 바로 열림 | **없음** → `kubectl port-forward svc/이름 8090:80` |
| 직접 빌드한 이미지 | 바로 사용 | `kind load docker-image 이름:태그` 필요 |

- 기본 엔진은 `docker.service` 비활성+`docker.socket` 활성이라 `docker ps`를 치면 데몬이 깨어난다. kind 노드 컨테이너 restart 정책은 on-failure:1.
- 직접 만든 이미지를 쓰는 파드는 `--image-pull-policy=Never`를 붙여야 인터넷에서 찾지 않고 로드해 둔 이미지를 쓴다.
- 노드 `docker restart kind-control-plane` 후 Ready까지 약 58초. **실제 재부팅 후 자동 복구는 미확인** — 안 켜지면 `docker ps` 후 `docker start kind-control-plane`.
- 쿠버네티스를 안 쓸 때 `docker stop kind-control-plane`으로 0.5GB 회수.
- 호스트 8080은 Spring 서버가 쓰고 있어 연습 포트는 8090대를 쓴다.
- k9s·lazydocker(TUI)는 GitHub 릴리스에서 받아 `sha256sum -c`로 검증 후 `~/.local/bin`에 설치. 한글 로케일 테두리 문제는 [[k9s-한국어-로케일-테두리-깨짐-RUNEWIDTH]].

## 볼륨(DB 데이터)을 엔진 사이로 옮기는 법

- alpine 컨테이너로 tar 스트림을 Desktop 엔진 → 기본 엔진으로 파이프하고, sha256 파일 목록(192개)으로 일치 검증. 원본은 Desktop에 남긴다.
- compose 볼륨은 `com.docker.compose.*` 라벨(config-hash 포함)을 똑같이 붙여 만들어야 compose가 같은 볼륨으로 인식한다. 라벨 없이 `docker run -v`로 자동 생성되면 라벨이 빠진다.
- zsh는 변수를 단어 분리하지 않아 `docker volume create $labels`가 `unknown flag`로 실패 → bash 배열(`args+=(--label "$l")`)로 해결. → [[zsh-단어분리-안됨-glob-nomatch-함정]]
- 이전 중 DB가 잠깐 꺼지므로 떠 있던 앱(Spring 등)은 에러가 나면 재실행. Docker 종료 시 restart 정책 없는 컨테이너가 같이 멈추는 점은 [[루트-공간-부족-Docker-디스크-data-이전과-Discord-로그-스팸]].

## 기타

- 숙제 검사용 원래 클러스터는 Desktop을 켜고 `kubectl config use-context docker-desktop`으로 복귀(Desktop은 삭제하지 않고 꺼 둠).
- zsh 탭 자동완성: `~/.zshrc`에 `command -v kubectl >/dev/null && source <(kubectl completion zsh)`.
- zsh에서 `echo ===`는 `=` 확장으로 실패 → `echo '---'`. `pgrep -f`는 자기 셸까지 매치되므로 `pgrep -x`나 PID 사용.

## 출처

Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
