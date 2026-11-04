---
tags: [IntelliJ, Gradle, 심볼릭링크, Java]
updated: 2026-11-04
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

## 같은 원인의 다른 증상: 코드 색이 키워드에만 들어옴 (2026-11-04 보충)
- `public`·`class` 같은 키워드만 색이 있고 `@Getter`·`this.errorCode` 같은 필드·애너테이션은 단색이며, 에디터 우측 상단에 분석 꺼짐 표시(`OFF`)가 보였다. 코드 문제가 아니고 실행에도 영향 없다.
- 단서: 상태바에 `Importing '…' Gradle Project`가 떠 있고, 왼쪽 트리에 같은 프로젝트가 `/data/project/…`와 `~/project/…`로 **두 번** 나왔다. 프로젝트를 심볼릭 링크 경로(`~/project`)로 열었는데 Gradle은 실경로(`/data/project`)를 보고 있어, 링크 쪽에서 연 파일을 프로젝트 소스로 인식하지 못해 의미 분석(semantic highlighting)이 안 됐다고 추정(설정·로그 기반 추정).
- 해결: `File > Close Project` → 시작 화면 `Open`에서 **실경로 `/data/project/JKYMHS-Backend`** 선택 → 상태바 `Importing…`이 사라질 때까지 대기(로그상 약 20초) → 파일 다시 열기. 결과: 트리에 프로젝트 하나, 색 복구.
- 확인법: 메서드 이름(`getMessage`)을 Ctrl+클릭해 정의로 이동하면 분석이 정상.
- 그래도 `OFF`면: `OFF`에 마우스를 올려 이유 확인 → `File`의 Power Save Mode 체크 해제 → Highlight 수준을 `All Problems`로. 안 되면 Gradle 탭 새로고침, `File > Invalidate Caches…` 후 재시작.
- `/data`는 홈(`~`) 안이 아니라 루트(`/`) 바로 아래에 있다. 열기 창 경로 입력칸에 `/data/project/JKYMHS-Backend`를 직접 붙여넣는 게 가장 쉽다.
- 우하단 `Plugin error`(JetBrains AI Assistant 버전 불일치 알림)는 색 문제와 무관.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
