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

## 같은 날 겹친 증상: 키맵이 VSCode라 IntelliJ 단축키가 안 먹음 (2026-11-11 보충)
- `Alt+Enter`(빠른 수정·import)와 `Ctrl+Alt+S`(설정)가 안 먹고 `Ctrl+Enter`가 줄만 바꾼 건 **키맵이 VSCode로 설정**돼 있어서였다. 키맵 선택 팝업에 `8 VSCode →`처럼 현재 키맵이 표시된다. VSCode 키맵에서 `Ctrl+Enter`는 "아래에 줄 삽입".
- 같은 기능의 대응(일반적인 매핑, 정확한 키는 Settings → Keymap에서 기능 이름으로 검색): import·빠른 수정 `Alt+Enter` ↔ `Ctrl+.` / 설정 `Ctrl+Alt+S` ↔ `Ctrl+,` / 명령 검색 `Ctrl+Shift+A` ↔ `Ctrl+Shift+P` / import 정리 `Ctrl+Alt+O` ↔ `Shift+Alt+O`. 강의·블로그는 대부분 IntelliJ 기본(GNOME 키맵) 기준이다.
- 설정을 열 수 없을 땐 `Shift` 두 번(전체 검색) → `auto import`/`keymap` 검색, 또는 `☰`(새 UI의 메뉴) → File → Settings. 한국어 키보드의 **오른쪽 Alt는 한/영 키**라 Alt 조합이 안 먹을 수 있어 왼쪽 Alt를 쓴다.
- 자동 import는 Settings → Editor → General → Auto Import → Java의 `Add unambiguous imports on the fly`(후보가 하나일 때만)·`Optimize imports on the fly`. **새로 치는 코드에만** 적용되고 이미 친 코드엔 안 붙는다. 이름이 겹치는 클래스(`Column`, `Configuration` 등)는 자동 선택이 안 되므로 자동완성 목록 오른쪽의 패키지(`jakarta.persistence` 등)를 보고 고른다.
- IntelliJ가 켜진 채 설정·`.idea`를 외부에서 고치면 종료 시 원래 값으로 덮어쓴다 → 완전히 종료한 뒤 수정. 최근 프로젝트 목록(`recentProjects.xml`)에 심볼릭 경로가 남아 있으면 목록에서 열 때마다 그 경로로 열리므로, 그 항목을 지우고 `lastProjectLocation`을 실경로(`/data/project`)로 바꾸고 `.idea`는 삭제 대신 백업해 두었다가 실경로로 직접 입력해 연다. 터미널(Claude)에서 IntelliJ를 실행하면 바로 종료돼 직접 열어야 했다.
- 알림은 무시해도 됨: Full Line Code Completion(로컬 모델이 한 줄 제안, 직접 쳐 배우는 중엔 꺼둘 만함), Reactive Streams 플러그인 추천(Lettuce가 Reactor를 써서 잡힌 것, WebFlux 안 쓰면 불필요 → Don't suggest again), database configurations(Do not suggest again).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JKYMHS-Backend)
