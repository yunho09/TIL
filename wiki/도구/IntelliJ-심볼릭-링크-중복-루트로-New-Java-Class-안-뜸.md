---
tags: [IntelliJ, Gradle, 심볼릭링크, Java]
updated: 2026-10-09
---
# IntelliJ 심볼릭 링크 중복 루트로 New → Java Class가 안 뜸

같은 디렉터리가 프로젝트 트리에 두 번 붙어 있으면(실경로 + 심볼릭 링크 경로), 모듈·소스 루트가 없는 쪽에서는 New 메뉴에 Java Class가 나타나지 않는다. 소스 루트가 마킹된 진짜 Gradle 모듈 루트에서 만들면 해결된다.

## 증상
- 패키지(`entity` 등)에 우클릭 → New 했는데 File / Directory 같은 범용 항목만 있고 **Java Class가 없음**.

## 원인
- `/home/yunho/project` → `/data/project` 심볼릭 링크라 물리적으로는 같은 폴더인데, IntelliJ는 **경로 문자열이 다르면 다른 것으로 취급**한다.
- `JKYMHS-Backend [backend]` (`/data/project/...`) 루트만 Gradle 모듈(`backend.main`, `backend.test`)에 연결되고 `src/main/java`가 **Sources Root**로 마킹됨.
- `~/project/JKYMHS-Backend` 루트는 File → Open 으로 한 번 더 "첨부"된 폴더라 모듈도 소스 루트도 없음. 단서: `.idea/workspace.xml`의 `last_opened_file_path`가 심볼릭 링크 경로.
- IntelliJ는 **소스 루트 안의 디렉터리에서만** New → Java Class를 보여준다.

## 해결
1. `[backend]` 표시가 있는 진짜 모듈 루트를 펼쳐 그 아래 패키지에서 New → Java Class.
2. 중복 루트는 우클릭 → **Remove from Project**.
3. 프로젝트는 심볼릭 링크 경로가 아닌 실경로(`/data/project/...`)로 연다.
4. 그래도 안 뜨면 Gradle 탭의 Reload All Gradle Projects로 소스 루트 마킹 복구.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
