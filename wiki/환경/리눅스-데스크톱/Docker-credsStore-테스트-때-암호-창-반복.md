---
tags: [docker, 리눅스, 자격증명, gradle, 트러블슈팅]
updated: 2026-11-02
---

# Docker credsStore 때문에 테스트마다 암호 창이 뜨는 문제

JKYMHS 백엔드의 Gradle 테스트를 돌릴 때마다 암호(키링/`pass` 복호화) 입력 창이 계속 떴다. 원인은 `~/.docker/config.json`의 **`credsStore`** 설정이다. Docker가 레지스트리 자격증명을 읽으려고 자격증명 헬퍼를 부르고, 헬퍼가 `pass`에 저장된 로그인 정보를 복호화하면서 창이 뜬다. `credsStore` 줄을 지워 해결했다.

## 원인 추적
- 처음엔 정체를 몰라 범위를 좁혀 갔다: 암호 창이 뜨는 **복호화 시각**이 **Gradle 테스트 JVM이 뜬 시각**과 겹쳤다.
- 테스트가 Docker 컨테이너를 띄우면서(컨테이너 기반 테스트) Docker 자격증명 조회가 일어나고, 그때 `credsStore` 헬퍼가 `pass`를 복호화하려고 암호를 요구한다.
- (Claude 보충) 컨테이너 테스트 라이브러리가 Testcontainers라는 확증은 세션 기록에 없다.

## 해결
- `~/.docker/config.json`에서 **`credsStore` 줄만 삭제**. 변경 전 백업: `~/.docker/config.json.bak-20261102`.
- 검증: 변경 후 `UserRepositoryTest` 5개 모두 통과, 컨테이너 정상 기동, 그동안 복호화 시도 0회.
- `pass`에 저장된 로그인 정보는 **지우지 않았다**.

## 트레이드오프
- **Docker Hub pull이 익명**이 되어 횟수 제한이 더 빡빡해진다.
- 다시 `docker login`하면 비밀번호가 `config.json`에 **base64(사실상 평문)** 로 저장된다 — 주의.
- Docker Desktop을 다시 켜면 이 설정이 되돌아와 창이 또 뜰 수 있다 (실제로 되돌아오는지는 **미확인**).
- 변경 전에 시작된 테스트가 띄운 창은 그냥 취소하면 된다.

## 되돌리기
```
cp ~/.docker/config.json.bak-20261102 ~/.docker/config.json
```

## 관련
- [[JKYMHS-Backend]] 프로젝트 테스트 환경
- [[루트-공간-부족-Docker-디스크-data-이전과-Discord-로그-스팸]], [[Docker-Desktop-쿠버네티스-이미지-손상과-기본-Docker-kind-전환]] — 같은 PC의 Docker 설정 이력

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-11-02
