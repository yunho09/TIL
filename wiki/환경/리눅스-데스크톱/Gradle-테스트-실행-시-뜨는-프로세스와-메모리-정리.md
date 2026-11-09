---
tags: [Gradle, Testcontainers, 메모리, JKYMHS]
updated: 2026-11-09
---

# Gradle 테스트 실행 시 뜨는 프로세스와 메모리 정리

JKYMHS 백엔드에서 `./gradlew test`를 한 번 돌리면 런처·Gradle 데몬·컴파일 워커·테스트 워커·테스트용 Docker 컨테이너가 함께 뜬다. 이 중 데몬과 컴파일 워커는 실행이 끝나도 남아서, worktree 여러 곳에서 테스트를 반복하면 RAM 27GB/30GB + 스왑 15GB 전부가 차는 사태가 났다(Gradle만 약 6GB).

## 실행 한 번에 뜨는 것

| 무엇 | 개수 | 메모리 | 언제 꺼지나 |
|---|---|---|---|
| gradlew 런처 (JDK 17) | 1 | 64MB | 실행이 끝나면 |
| **Gradle 데몬** (JDK 17) | 0~1 | 500~700MB | 3시간 유휴 후. 실행이 끝나도 남음 |
| **컴파일 워커** (JDK 21, javac) | worktree당 1개로 추정 | 300~400MB | 데몬이 꺼질 때 |
| **테스트 워커** (JDK 21) | 1 | ~600MB | 테스트가 끝나면 |
| 테스트용 Docker 컨테이너(MySQL 8.4, Redis 7.4, ryuk) | 각 1 | ~500MB | 테스트 워커 종료 시 ryuk가 정리 |

## 열리는 기준

- **데몬**: 놀고 있는 데몬이 있으면 재사용, 다른 빌드가 쓰는 중이면 새로 띄움 → worktree 두 곳에서 동시 실행하면 데몬 2개.
- **컴파일 워커**: Gradle은 JDK 17, 프로젝트는 `toolchain 21`(`build.gradle:10-12`)이라 버전이 달라 컴파일러를 별도 JVM으로 띄움. 데몬이 사는 동안 남아 쌓임(40분~1시간 된 워커 10개쯤 관찰). (worktree별로 생긴다는 건 개수로 추정한 것이고 Gradle 문서로 확인하지 않음.)
- **테스트 워커**: `test` 태스크에 병렬 설정이 없어 실행당 JVM 1개(`build.gradle:48`).
- **테스트 컨테이너**: `IntegrationTestSupport`를 상속한 클래스(28개)가 하나라도 돌면 뜸. `TestContainers.java` static 블록에서 MySQL·Redis를 띄우며 테스트 JVM당 한 세트를 모든 테스트가 공유. `@WebMvcTest`·순수 단위 테스트만 돌리면 안 뜸.
- `compose.yaml`의 `jkymhs-mysql`·`jkymhs-redis`·`mailpit`은 로컬 개발용이라 테스트와 무관, `docker compose up` 때만 뜸.

## 메모리가 쌓인 원인

1. 두 worktree(27·28)에서 테스트가 **동시에** 돌아 데몬·테스트 워커·MySQL이 2세트.
2. worktree가 12개라 빌드할 때마다 컴파일 워커가 쌓임.
3. 다른 Claude 세션(`--dangerously-skip-permissions`)이 테스트를 반복 실행 — 외부에서 Gradle을 죽여도 몇 초 뒤 다시 살아남. 근본 해결은 그 세션을 Esc로 멈추거나 끝내는 것.

## 줄이는 방법

- 작업이 끝나면 해당 저장소에서 `./gradlew --stop` (데몬+컴파일 워커 함께 종료, 4~6GB 확보. 이번에 27GB→19GB, 가용 3GB→11GB).
- `~/.gradle/gradle.properties`에 `org.gradle.daemon.idletimeout=600000`(10분)으로 유휴 종료를 앞당김 (제안만 했고 적용은 미확인).
- 테스트는 worktree 한 곳에서 순서대로, 필요한 것만: `./gradlew test --tests '*LoginApiTest'`.
- 테스트 컨테이너를 `docker rm`으로 직접 지우려 했으나 권한 거부됨 — Gradle이 끊기면 Testcontainers(ryuk)가 정리하므로 직접 지울 필요 없음.

## 관련
- [[리눅스-메모리-점유-앱별-진단-PSS]] — 앱별 메모리 합산 방법
- [[Claude-Code-MCP-고아-프로세스와-메모리-정리]] — 세션 수에 따른 메모리 점유
- [[Docker-credsStore-테스트-때-암호-창-반복]] — 같은 Testcontainers 기동 경로
- [[Gradle-JDK-17-못-찾음-Homebrew-libexec-경로]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE), 2026-11-09
