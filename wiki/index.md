# Wiki Index

Claude가 관리하는 페이지 카탈로그. 페이지당 한 줄 요약. 새 페이지가 생기거나 요약이 바뀌면 여기도 함께 갱신한다.

## 페이지

### 메타 (위키/vault 운영)
- [[위키-자동-캡처]] — Karpathy LLM Wiki 모델 채택 이유, SessionEnd 훅 기반 자동 ingest+push 파이프라인, 스킵 조건, 적용 범위/한계, Notion 자동 연동 미채택 이유

### 환경/리눅스
- [[Orca-IDE-리눅스-설치]] — AppImage type 2가 Ubuntu 26.04에서 안 열리는 이유(libfuse2 부재)와 압축 해제 설치법, `orca`↔GNOME 화면낭독기 이름 충돌, 자동 업데이트 불가 등 한계
- [[GNOME-Wayland-wl-clipboard-포커스-토스트]] — Mutter 50에 data-control 프로토콜 부재→wl-clipboard 폴백 경로→`focus-new-windows=strict`가 겹쳐 뜨는 정체불명 토스트의 원인 체인과 `smart` 복구법
- [[절전-복귀-지연-원인과-zram-도입]] — 스왑 고갈+복귀 직후 systemd 타이머 폭주가 겹쳐 절전 복귀가 9초 넘게 걸리던 원인, zram 압축 스왑 도입(설치 시점 기본값 함정 포함)과 타이머 완화 조치
- [[Ptyxis-터치패드-스크롤-속도-패치]] — GTK4 `-Bsymbolic`으로 LD_PRELOAD 차단, `enable-fallback-scrolling=false`로 VTE 패치가 스크롤백에 안 먹히던 원인, `DBusActivatable=true`가 PATH 래퍼를 우회하는 함정, 스크롤백/앱 배율 분리 + bypass 패치로 해결
- [[GNOME-오버뷰-창-미리보기-사라짐]] — 3손가락 스와이프 후 가끔 창 미리보기만 비는 문제, `_gestureEnd` 예외 가설은 진단 로그로 반증됨(원인 미확정), 자동 감지+덤프 익스텐션 설치해 다음 재현 대기 중, GNOME 확장은 코드 변경 시 핫로드 안 되고 로그아웃/로그인 필요(`ReloadExtension` D-Bus deprecated)

### 언어/Java/Spring
- [[빈과-DI]] — 인터페이스+구현체+생성자 주입 패턴, 다중 구현체 주입(@Primary/@Qualifier/List<T>), 싱글톤 규칙, 실무 사용 빈도
- [[스프링-컨테이너]] — 컨테이너=Map 비유, 빈은 메모리에만 존재, 조립 순서, @SpringBootApplication 구성, 컴포넌트 스캔 기준점과 src 폴더 구조
- [[스프링-CRUD-계층구조]] — Controller/Service/Repository 계층별 역할, 요청 값 추출 3종(@PathVariable/@RequestParam/@RequestBody), URL+메서드 매핑, Postman/curl 테스트 방법
- [[JPA-엔티티와-리포지토리]] — @Entity 규칙(빈 생성자, Long id, setter 대신 update()), JpaRepository가 빈 인터페이스로 동작하는 원리와 쿼리 메서드
- [[MySQL-연동]] — datasource 설정, ddl-auto 선택지, Docker MySQL 컨테이너 명령어 모음

### 프론트엔드
- [[Figma-Dev-Mode-MCP]] — 데스크톱 앱에 파일이 열려 있어야 동작, view-only 파일에서 씬 그래프 API 실패·스크린샷 폴백과 픽셀 계측 대안
- [[Figma-MCP-팀별-호출-한도]] — `claude.ai Figma` 호출 한도는 계정이 아니라 파일 소유 팀의 플랜에 걸림, `whoami`는 항상 성공, 페이지 루트 전체 덤프가 한도를 급격히 소모
- [[S3-Presigned-URL-업로드]] — presign 2단계 흐름, Authorization 헤더가 서명을 깨는 이유, S3 CORS 필요성, Playwright의 Blob 바디 목킹 한계
- [[CSS-Container-shrink-to-fit]] — `max-width`는 상한일 뿐 폭을 확보 못 함, `main` shrink-to-fit 원인과 해결, 전체 배경색 우회법
- [[CSS-fieldset-legend-flex-패딩-무시]] — `display:flex` fieldset에서 `<legend>`가 padding 무시하고 최상단에 붙는 원인과 float+clear 해결법, 고정 px 그리드→`fr` 반응형 전환, 한글 `word-break: keep-all`
- [[Vite-빌드타임-환경변수-인라인]] — `VITE_*`가 런타임이 아니라 빌드 시점에 번들에 문자열로 치환됨, 그래서 비밀값이 될 수 없고 값 변경 시 재빌드 필요
- [[도로명주소-검색-API-신청]] — business.juso.go.kr 신청 절차(검색 API vs 팝업 API, 개발/운영 승인키 흐름 차이), `confmKey`/JSONP 사용법, 승인키가 URL 단위로 묶이는 이유
- [[401을-세션만료로-오인한-로그인-루프]] — 백엔드가 권한 부족도 403 대신 401로 응답하고 프론트가 401을 무조건 세션 만료로 처리할 때 로그인 직후 화면이 스스로 로그아웃을 유발하는 원인 체인과 해결(토큰 유효성 우선 확인)
- [[Vite-모노레포-워크스페이스-패키지-dev서버-캐시]] — 앱 밖 `packages/*` 워크스페이스 패키지 수정이 HMR에 안 걸려 dev 서버가 옛 변환 캐시를 계속 서빙하는 문제, `.vite` 캐시 삭제+`--force` 재시작으로 해결

### 프로젝트/spring-practice
- [[프로젝트-현황]] — spring-practice 구조·API·DB 환경·알려진 허점 스냅샷 (2026-09-02)
- [[학습-진행상황]] — 스프링 로드맵 진행 상태, 겪은 에러들, 다음 단계 후보

### 프로젝트/JOBIS-FE-V2
- [[JOBIS-FE-V2/프로젝트-현황]] — 퍼블리싱·커밋 컨벤션(이슈 1=브랜치 1=PR 1 기본값), `packages/api` 공용 코드 결함(presign/더블슬래시 수정완료/401 이중의미), `updateParams`↔`getParam` 키 표기 불일치로 필터 죽는 버그 클래스, 백엔드 API 확인사항(status enum, acceptances 취소 의미, 조회수, 배너) (2026-09-07)

### 프로젝트/Zaemit-공모전
- [[Zaemit-공모전/Zaemit-MCP-연동]] — 정본 엔드포인트(`zaemit.ai/mcp`, `mcp.zaemit.ai`는 랜딩페이지 함정), 게이트웨이 3종 구조(403툴/57그룹), 게시판·문의폼 등 플러그인 설치 필요 기능, Free 플랜 한도, 공모전 제약 (2026-09-18 마감), OAuth 인증 완료로 연동 이력 요건 충족 확인 (2026-09-08)

### 프로젝트/ToyVillage-Admin-FE
- [[ToyVillage-Admin-FE/프로젝트-현황]] — Figma 퍼블리싱 harness 구조(specs/approvals 분리, 게이트 해시 판정, ②·⑦ 사람 게이트, approvals 재승인 시 cp 누락 함정), 구 파일 폐기 후 `yot` 기준 전환, 업무관리 목록·생성·수정·상세 4개 feature 퍼블리싱·게이트 승인·커밋 완료, 상태 모델(반려→파생 `지연`) 변경, 업무 목록 필터·페이지네이션 구현 패턴(useMemo 3단 파이프라인, 렌더 중 상태 보정으로 페이지 리셋), 레이아웃 버그 수정, task-report S16 재승인 미결 (2026-09-08)

### 프로젝트/Commonly-FE
- [[Commonly-FE/프로젝트-현황]] — 경력증명서 발급 프로토타입, FE↔BE 갭 목록(개별등록/민원인 발급 막힘, 대량등록 데이터 미연결 버그, JWT 클레임 부족), 진행한 이슈·PR(#68–#75), juso 주소검색 키 적용, PR 코드리뷰에서 나온 동시성 disabled 스코프 버그·문서번호 빈값 통과 버그 수정, 로그인 무한 루프(#74/#75) 원인·수정 (2026-09-08)

### 도구
- [[Claude-Code-MCP-서버-등록]] — MCP 서버 목록은 세션 부팅 시에만 로드됨, Desktop 내장 세션은 OAuth 브라우저 승인 불가(터미널 CLI 필요), 터미널을 도중에 닫아 토큰 교환이 빈 값으로 실패하는 패턴과 확인법, `/mcp` 메뉴 커서 위치를 놓쳐 엉뚱한 서버로 들어가는 함정, 자격증명 파일 직접 읽기 우회는 auto mode가 차단
- [[Git-브랜치명-샵-이스케이프]] — 브랜치명에 `#`이 있으면 쉘 주석으로 잘려서 `--delete` 등 뒤 인자가 사라짐, 항상 따옴표로 감싸야 함
- [[Claude-Code-느낌표-bash-접두사-채팅전용]] — `!command`는 채팅 입력 전용 즉시실행 기능, 실제 터미널(bash-input)에 그대로 붙여넣으면 `command not found: !node`로 실패
