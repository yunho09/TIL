---
tags: [Gradle, JDK, Homebrew, IntelliJ]
updated: 2026-10-21
---
# Gradle이 JDK 17을 못 찾을 때: Homebrew `libexec` 경로

`build.gradle`이 Java 17 툴체인을 요구하는데 Gradle이 JDK 17을 못 찾아 `./gradlew test`와 IntelliJ Sync가 같은 계열 에러로 실패했다. 원인은 `~/.gradle/gradle.properties`가 Homebrew 포장 폴더를 가리키고 있었기 때문이고, 실제 JDK 홈인 `libexec`로 바꿔 해결했다.

## 원인
- JDK 17은 Homebrew(Linuxbrew)로 이미 설치돼 있었음.
- `org.gradle.java.installations.paths`가 `/home/linuxbrew/.linuxbrew/opt/openjdk@17`(포장 폴더)를 가리킴. 진짜 JDK 홈은 그 안의 **`libexec`**라서 Gradle이 JDK로 인식하지 못함.
- 스크린샷의 에러 여러 개는 모두 이 한 원인이고, "New Minor Gradle Version Available"은 무시해도 되는 안내.

## 해결
```
# ~/.gradle/gradle.properties
org.gradle.java.installations.paths=/home/linuxbrew/.linuxbrew/opt/openjdk@17/libexec
```
- 이 파일은 프로젝트가 아니라 사용자 PC 전체 Gradle 설정이라 git 변경에 안 잡힘.
- 검증: 테스트 47개를 `--rerun-tasks`류로 강제 재실행해 전부 통과. 결과가 `UP-TO-DATE` 캐시에서 나오면 통과 증거가 안 되므로 강제 재실행 필요.

## IntelliJ에서
1. Gradle 툴 창 **Reload All Gradle Projects**(설정 변경 전에 실행된 Sync는 옛 결과).
2. 그래도 실패하면 Settings → Build Tools → Gradle → **Gradle JVM**을 Homebrew JDK 17로 지정, 목록에 없으면 `libexec` 경로를 직접 추가.

관련: [[IntelliJ-심볼릭-링크-중복-루트로-New-Java-Class-안-뜸]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/bium-be)
