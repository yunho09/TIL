# Wiki Index

Claude가 관리하는 페이지 카탈로그. 페이지당 한 줄 요약. 새 페이지가 생기거나 요약이 바뀌면 여기도 함께 갱신한다.
섹션 제목 = `wiki/` 아래 실제 폴더 경로.

## 페이지

### 메타 (위키/vault 운영)
- [[위키-자동-캡처]] — Karpathy LLM Wiki 모델 채택 이유, SessionEnd 훅 기반 자동 ingest+push 파이프라인, 스킵 조건, 적용 범위/한계, Notion 자동 연동 미채택 이유

### 환경/GNOME
- [[GNOME-오버뷰-창-미리보기-사라짐]] — 3손가락 스와이프 후 가끔 창 미리보기만 비는 문제, `_gestureEnd` 예외 가설은 진단 로그로 반증됨(원인 미확정), 자동 감지+덤프 익스텐션 설치해 다음 재현 대기 중, GNOME 확장은 코드 변경 시 핫로드 안 되고 로그아웃/로그인 필요(`ReloadExtension` D-Bus deprecated)
- [[GNOME-Wayland-wl-clipboard-포커스-토스트]] — Mutter 50에 data-control 프로토콜 부재→wl-clipboard 폴백 경로→`focus-new-windows=strict`가 겹쳐 뜨는 정체불명 토스트의 원인 체인과 `smart` 복구법
- [[Vitals-확장-상단바-시스템-모니터]] — 자체 스키마를 쓰는 확장은 `gsettings --schemadir`로 확장 설치 경로의 schemas를 직접 지정해야 하는 이유(Vitals `hot-sensors`/`position-in-panel` 예시), GNOME 50에서 D-Bus 스크린샷이 `AccessDenied`로 막혀 있음

### 환경/리눅스-데스크톱
- [[GPU-유휴클럭-램프업-지연-프레임드랍]] — 창 드래그·오버뷰 애니메이션만 버벅이고 계속 움직이면 부드러워지는 증상의 원인은 GPU 유휴 클럭 400MHz 고정+최대클럭 램프업 ~1.5초인데 애니메이션은 0.2~0.4초라 못 따라가는 것, 컴포지터/패널 60Hz 하락·Discord `--disable-gpu`라는 오진단 2건은 실측으로 반증, wayland presentation-time 프로토콜·120Hz vs 60Hz 시각 비교자·act_freq 샘플링 진단 기법, `min_freq` sysfs로 임시 해결(재부팅시 원복)과 배터리 트레이드오프, XWayland 프레임 페이싱은 별개 미해결 이슈로 남음
- [[절전-복귀-지연-원인과-zram-도입]] — 스왑 고갈+복귀 직후 systemd 타이머 폭주가 겹쳐 절전 복귀가 9초 넘게 걸리던 원인, zram 압축 스왑 도입(설치 시점 기본값 함정 포함)과 타이머 완화 조치. zram 도입 후에도 재발 — 진짜 급소는 gnome-shell 자신이 zram으로 내보낸 페이지 재적재, `CPUWeight`가 형제 슬라이스끼리만 경쟁한다는 함정, MemoryMin/background.slice/NVMe 스케줄러/fstrim 시간 이동 등 6가지 추가 조치. **최종 확인된 진짜 원인은 복귀가 아니라 전원 버튼 절전 진입 지연** — gnome-shell의 화면 시간(Screen Time) 기록 기능이 sleep 억제자를 놓지 않아 logind가 `InhibitDelayMaxSec` 5초를 다 채운 뒤에야 절전에 들어간 것, `Lid opened`만 측정하던 도구의 사각지대
- [[Ptyxis-터치패드-스크롤-속도-패치]] — GTK4 `-Bsymbolic`으로 LD_PRELOAD 차단, `enable-fallback-scrolling=false`로 VTE 패치가 스크롤백에 안 먹히던 원인, `DBusActivatable=true`가 PATH 래퍼를 우회하는 함정, 스크롤백/앱 배율 분리 + bypass 패치로 해결
- [[터치패드-전역-스크롤-속도-libinput-quirks]] — Chrome 등 모든 앱의 터치패드 스크롤이 느렸던 원인은 `/etc/libinput/local-overrides.quirks`의 `AttrResolutionHint=240x240`(mm당 해상도를 속여 스크롤량 축소), 실제 네이티브 해상도 120/mm 역산, 같은 값이 커서 속도·3손가락 제스처 임계값에도 걸리는 부작용, `pad-scroll-speed` 스크립트로 무로그아웃 적용, 터치패드/마우스휠(scroll-boost)/터미널(Ptyxis) 3개 스크롤 경로가 서로 독립적이라는 정리
- [[Orca-IDE-리눅스-설치]] — AppImage type 2가 Ubuntu 26.04에서 안 열리는 이유(libfuse2 부재)와 압축 해제 설치법, `orca`↔GNOME 화면낭독기 이름 충돌, 자동 업데이트 불가 등 한계
- [[리눅스-메모리-점유-앱별-진단-PSS]] — RSS 합산은 공유 메모리 중복 계산으로 부풀려짐, PSS로 앱별 합산해야 정확, zram/Shmem/slab 등 앱 외 요소까지 더해야 총량이 맞는 이유, Electron 앱은 하나 끄면 런타임째 통째로 회수되는 정리 우선순위
- [[Figma-데스크톱-앱-메모리-중복]] — 비공식 Electron 래퍼(`figma-linux-next`)가 Chromium 런타임을 중복으로 띄우는 원인(브라우저판 대안), 열어둔 탭은 자동 해제 안 됨, `settings.json`을 열린 탭 목록으로 오인하면 안 되는 이유, 탭↔렌더러 PID 매칭 불가로 안전한 정리법은 앱 내 직접 조작뿐
- [[Figma-터치패드-핀치줌-속도-패치]] — `app.asar` 전개+preload 주입으로 핀치줌 배속(`figma-zoom-speed`, CDP로 검증), "확대가 안 되고 그냥 움직여짐" 증상은 핀치 종료 ~190ms 뒤 남은 움직임이 libinput에 새 스크롤 제스처로 재분류되는 것(+`AttrResolutionHint` 스크롤 2배 부작용 중첩)이 원인, 핀치 꼬리 300ms 가드로 대응(사용자 확인 대기 중, 미확정)

### 라즈베리파이/GPIO
- [[GPIO-기초]] — GPIO가 무엇인지(3.3V 출력/입력), 켜짐·꺼짐뿐인 핀으로 PWM이 밝기를 만드는 원리, BCM 번호와 물리 핀 자리 번호가 다른 두 체계, 핀은 한 프로세스만 점유한다는 성질
- [[gpiozero-GPIO-배선-확인]] — gpiozero는 LED가 없어도 에러를 안 내므로 터미널 출력이 무의미, 핀 스윕으로 실제 배선 핀 찾기, BCM 번호와 물리 핀 번호 혼동, `GPIO busy`(같은 핀을 쓰는 프로그램 중복 실행)와 잔여 프로세스 정리, 동시 실행은 가능하고 배타적인 건 핀·포트 단위라는 점과 핀 소유자-클라이언트 구조
- [[gpiozero-PWMLED-밝기-제어]] — 켜고 끄는 LED와 달리 PWMLED는 value 0.0~1.0으로 조광, 슬라이더 0~255를 나누어 매핑할 때 STEPS 불일치 함정, PWM이 핀을 계속 점유하는 성질

### 라즈베리파이/PyQt5-GUI
- [[라즈베리파이-GUI-개발-워크플로]] — PC에서 Designer+pyuic5로 만들고 파이에서 실행하는 전체 흐름, 어느 기계/어느 터미널에서 무엇을 하는지 구분표, 가짜 gpiozero를 끼워 PC에서 로직을 미리 검증하는 법, 자주 막히는 지점 모음
- [[라즈베리파이-PyQt5-설치-ARM64]] — pip PyQt5가 ARM64에서 소스 빌드로 빠지는 문제, apt `python3-pyqt5` + venv `--system-site-packages` 조합, Pi 5용 LGPIOFactory 명시가 Pi 4에서는 불필요한 이유
- [[라즈베리파이-GUI-SSH-VNC-실행]] — GUI는 SSH 터미널(=DISPLAY 없음)에서 못 뜬다, 코드가 import 시점에 `QT_QPA_PLATFORM`을 덮어써 셸 환경변수가 무시되는 함정, Debian 13의 wayvnc 활성화와 세션 없으면 붙을 게 없는 구조, PC 터미널과 파이 터미널 구분(scp는 PC에서, PowerShell은 `&&` 미지원), SSH/SCP/VNC 역할 구분 — VNC는 저장소가 아니라 화면이고 파일은 파이 디스크에만 있다
- [[Qt-Designer-PyQt5-연결]] — objectName이 디자인↔코드의 유일한 연결고리, 폼 objectName을 잘못 바꾸면 `Ui_` 클래스명이 바뀌어 코드가 어긋남, 레이아웃 단축키는 Ctrl+1/2/5, Qt5/Qt6 Designer의 .ui는 호환되지 않음, 반복문 시그널 연결 시 람다 늦은 바인딩, 표시와 적용 분리 패턴, pyuic5가 XML을 파이썬 코드로 번역하는 방식과 생성 파일을 직접 고치면 안 되는 이유
- [[가짜-모듈로-하드웨어-없이-테스트]] — `sys.modules`에 껍데기 모듈을 미리 등록해 하드웨어 라이브러리 import를 가로채는 기법, offscreen 렌더로 GUI 버튼까지 코드로 눌러 검증, 이 방식으로 걸러지는 것과 걸러지지 않는 것

### 언어/Java/Spring
- [[빈과-DI]] — 인터페이스+구현체+생성자 주입 패턴, 다중 구현체 주입(@Primary/@Qualifier/List<T>), 싱글톤 규칙, 실무 사용 빈도
- [[스프링-컨테이너]] — 컨테이너=Map 비유, 빈은 메모리에만 존재, 조립 순서, @SpringBootApplication 구성, 컴포넌트 스캔 기준점과 src 폴더 구조
- [[스프링-CRUD-계층구조]] — Controller/Service/Repository 계층별 역할, 요청 값 추출 3종(@PathVariable/@RequestParam/@RequestBody), URL+메서드 매핑, Postman/curl 테스트 방법
- [[JPA-엔티티와-리포지토리]] — @Entity 규칙(빈 생성자, Long id, setter 대신 update()), JpaRepository가 빈 인터페이스로 동작하는 원리와 쿼리 메서드
- [[MySQL-연동]] — datasource 설정, ddl-auto 선택지, Docker MySQL 컨테이너 명령어 모음

### 언어/JavaScript
- [[Nullish-Coalescing-빈문자열-함정]] — `??`는 `null`/`undefined`만 폴백하고 `""`/`0`/`false`는 안 걸러서, 파싱 결과가 정상적으로 빈 문자열이 되는 경우 의도한 fallback이 발동 안 함(`||`와의 차이), 같은 헬퍼가 앱마다 `??`/`||`로 갈린 divergence 실사례

### 프론트엔드/CSS
- [[CSS-Container-shrink-to-fit]] — `max-width`는 상한일 뿐 폭을 확보 못 함, `main` shrink-to-fit 원인과 해결, 전체 배경색 우회법. 후속 함정: shrink-to-fit을 고쳐 `main`에 `width:100%`를 주면 교차축 `auto` 마진이 무력화돼 중앙 정렬이 깨짐 → 폭을 제한하는 자식(`Container`)에 `margin-inline: auto`를 둬서 해결
- [[CSS-fieldset-legend-flex-패딩-무시]] — `display:flex` fieldset에서 `<legend>`가 padding 무시하고 최상단에 붙는 원인과 float+clear 해결법, 고정 px 그리드→`fr` 반응형 전환, 한글 `word-break: keep-all`

### 프론트엔드/빌드도구
- [[Vite-빌드타임-환경변수-인라인]] — `VITE_*`가 런타임이 아니라 빌드 시점에 번들에 문자열로 치환됨, 그래서 비밀값이 될 수 없고 값 변경 시 재빌드 필요
- [[Vite-모노레포-워크스페이스-패키지-dev서버-캐시]] — 앱 밖 `packages/*` 워크스페이스 패키지 수정이 HMR에 안 걸려 dev 서버가 옛 변환 캐시를 계속 서빙하는 문제, `.vite` 캐시 삭제+`--force` 재시작으로 해결
- [[Vite-공용설정-publicDir-모노레포-경로]] — 공용 `vite.config.common.ts`의 `publicDir`을 `import.meta.dirname` 기준으로 계산하면 레포 루트 `public/`로 고정돼 앱별 `apps/<app>/public/`이 죽은 폴더가 되는 함정

### 프론트엔드/디자인-연동
- [[Figma-Dev-Mode-MCP]] — 데스크톱 앱에 파일이 열려 있어야 동작, view-only 파일에서 씬 그래프 API 실패·스크린샷 폴백과 픽셀 계측 대안. 에셋 export 함정 3가지: `get_design_context` 이미지가 전 픽셀 투명일 수 있음, `download_assets`의 `export`는 부모 프레임에 잘린 합성 렌더일 수 있음, 원본은 `rawImages`로 받아야 함
- [[Figma-MCP-팀별-호출-한도]] — `claude.ai Figma` 호출 한도는 계정이 아니라 파일 소유 팀의 플랜에 걸림, `whoami`는 항상 성공, 페이지 루트 전체 덤프가 한도를 급격히 소모

### 프론트엔드/API-인증
- [[401을-세션만료로-오인한-로그인-루프]] — 백엔드가 권한 부족도 403 대신 401로 응답하고 프론트가 401을 무조건 세션 만료로 처리할 때 로그인 직후 화면이 스스로 로그아웃을 유발하는 원인 체인과 해결(토큰 유효성 우선 확인). Spring Security에서 흔한 원인은 `accessDeniedHandler` 미등록으로 `AccessDeniedException`이 401 entry point로 새는 것. 관련 함정: 인터셉터가 403에도 재발급 후 재시도하면 생성(POST)류 요청에서 중복 등록 위험이 생긴다는 것과, 이 판단이 리팩터링 중 회귀하기 쉽다는 점
- [[S3-Presigned-URL-업로드]] — presign 2단계 흐름, Authorization 헤더가 서명을 깨는 이유, S3 CORS 필요성, Playwright의 Blob 바디 목킹 한계
- [[도로명주소-검색-API-신청]] — business.juso.go.kr 신청 절차(검색 API vs 팝업 API, 개발/운영 승인키 흐름 차이), `confmKey`/JSONP 사용법, 승인키가 URL 단위로 묶이는 이유
- [[TanStack-Query-무효화-키-null-vs-undefined]] — 쿼리 키의 빈 파라미터를 `?? null`로 채우면 prefix 무효화가 안 먹는 이유(TanStack Query는 `undefined`만 와일드카드로 취급), `params`를 그대로 넘겨 `undefined`가 되게 하는 해결

### 프론트엔드/테스트
- [[테스트-픽스처-고정날짜-현재월-필터-노후화]] — "이번 달"만 렌더하는 화면에 고정 과거 날짜 픽스처를 쓰면 작성 당시엔 통과하다 달이 바뀌면서 자동으로 실패하는 시간 의존 테스트 패턴, 판별법과 상대 날짜 전환 해결책
- [[배럴-Import-함정]] — 엔티티 배럴(`index.ts`)이 상수 하나 때문에 재수출 전체를 실행시켜 axios CJS 체인이 Node/Playwright 로더에서 링크 실패하는 원인, `Total: 0 tests in 0 files`로 조용히 수집조차 안 되는 게 가장 위험한 이유, 하위 모듈 직접 import로 우회, 집계 스크립트가 `"Total:"` 문자열만 보거나 `tail -1`로 판정하면 이런 실패를 놓치는 부수 함정. 다른 메커니즘: 배럴 경유 순환 참조가 top-level `styled(X)` 평가 시점에 `undefined`를 주입해 크래시나는 사례
- [[네트워크-레벨-모킹]] — 함수 교체/네트워크 가로채기/진짜 서버 3단계 비교, MSW·Playwright `page.route`·Cypress·nock·WireMock 도구별 구현 방식, fulfill/continue/abort/fallback, 고정·대본형·상태형 응답 패턴, route 등록 순서(나중 등록이 먼저 매칭)·등록 시점·서비스워커 우회·패턴 정확도 함정, 계약 승인+해시 동결로 contract drift를 보완하는 구조
- [[스토리북-전용-Vitest-설정에-묻힌-유닛테스트]] — `storybookTest` 플러그인이 `include`를 스토리 파일로 고정해 `.test.tsx`가 수십 개 있어도 한 번도 안 도는 함정, `vitest.unit.config.ts` 분리 해결, jsdom vs 브라우저 모드 선택 기준(계산된 CSS 값 차이), `globals:true` 없으면 testing-library 자동 cleanup이 등록 안 돼 DOM이 누적되는 문제, Playwright 브라우저 빌드 버전 불일치·리눅스 배포판 미지원 시 캐시 심볼릭 링크 우회

### 프로젝트/spring-practice
- [[프로젝트-현황]] — spring-practice 구조·API·DB 환경·알려진 허점 스냅샷 (2026-09-02)
- [[학습-진행상황]] — 스프링 로드맵 진행 상태, 겪은 에러들, 다음 단계 후보

### 프로젝트/JOBIS-FE-V2
- [[JOBIS-FE-V2/프로젝트-현황]] — 퍼블리싱·커밋 컨벤션(이슈 1=브랜치 1=PR 1 기본값), `packages/api` 공용 코드 결함(presign/더블슬래시 수정완료·단 브랜치별 재발 확인됨/401 이중의미/캐시 무효화 누락), `updateParams`↔`getParam` 키 표기 불일치로 필터 죽는 버그 클래스, 첨부파일 종류를 서버가 구분 못 하는 설계 한계, 파일 업로드 스펙 두 벌 충돌(미해결), 백엔드 API 확인사항(status enum, acceptances 취소 의미, 조회수, 배너), acceptances 전체 조회 API 부재 명세서로 재확인, 어드민 취업관리(`/student`) 화면 — API 연동·자동선택·체크박스 동작 전부 이번 브랜치에서 제외하고 순수 퍼블리싱만 유지하기로 한 결정과 복구 경로, 버그 제보 페이지 Figma 정합화로 디자인시스템 확장(`Input`/`TextArea` underline variant, `FileUpload` 스펙 변경이 어드민 공지 등록에도 영향) 및 학생 앱 라우터 `main` shrink-to-fit 전역 수정. **(2026-09-10 갱신)** `main width:100%` 회귀로 전 페이지 좌측 쏠림 → `Container margin-inline:auto`로 수정, 공지 첨부파일명 `??`/`||` divergence 수정, 공지 12개 페이지네이션은 스테이징 서버 미지원 확인돼 보류, 홈 화면 흰 화면 버그는 토큰 재발급 큐 데드락으로 추가 의심, **어드민 `Footer` 테마 크래시가 재현되며 09-09의 "`--force` 탓" 결론과 상충**(원인 미확정 상태로 재오픈). **(2026-09-11 갱신)** 커밋에 Claude attribution 라인 붙이지 않기로 결정(기존 이력 재작성 완료), 디자인시스템 `.test.tsx` 46개가 storybookTest `include` 고정으로 한 번도 안 돌고 있던 문제 발견·분리 해결(418건 통과), `Modal` 순환참조·`Icon` 강제 aria-label 접근성 부채(59곳 영향, 보류)·죽은 테스트 3건 정리, `createQueryHook` params-누락 URL 버그와 공지 캐시 무효화 `null` 버그 수정, 403 인터셉터 재시도 회귀 원복, 학생 앱 이미지 URL 접두사 미적용·버그제보 API 403(서버측, 미해결) 신규 이슈, PR #147이 스택 PR 7개(#148~154)로 재구성됨

### 프로젝트/Zaemit-공모전
- [[Zaemit-공모전/Zaemit-MCP-연동]] — 정본 엔드포인트(`zaemit.ai/mcp`, `mcp.zaemit.ai`는 랜딩페이지 함정), 게이트웨이 3종 구조(403툴/57그룹), 게시판·문의폼 등 플러그인 설치 필요 기능, Free 플랜 한도, 공모전 제약 (2026-09-18 마감), OAuth 인증 완료로 연동 이력 요건 충족 확인 (2026-09-08)
- [[Zaemit-공모전/Zaemit-사이트-제작-트러블슈팅]] — `board_embed`의 `data-limit`이 제목 div가 아니라 `.wv-board` 루트에서만 읽히는 버그, 검수는 curl 대신 실제 브라우저 렌더로 해야 하는 이유, 툴 인자 한글은 리터럴로(유니코드 이스케이프 조립 금지), `overflow:hidden`만으론 절대위치 장식요소가 안 잘리고 부모에 `position:relative`가 같이 필요한 이유, 외부 이미지는 `image_import`로 저장해야 만료 안 됨, 반투명 카드+밝은 배경 사진 가독성 결함 패턴 (2026-09-08)

### 프로젝트/ToyVillage-Admin-FE
- [[ToyVillage-Admin-FE/프로젝트-현황]] — Figma 퍼블리싱 harness 구조(specs/approvals 분리, 게이트 해시 판정, ②·⑦ 사람 게이트, approvals 재승인 시 cp 누락 함정), 구 파일 폐기 후 `yot` 기준 전환, 업무관리 목록·생성·수정·상세 4개 feature 퍼블리싱·게이트 승인·커밋 완료, 업무 목록 필터·페이지네이션 구현 패턴(useMemo 3단 파이프라인, 렌더 중 상태 보정으로 페이지 리셋), 레이아웃 버그 수정, 업무지시 API 연동 harness(RUNBOOK ①~⑬) 착수부터 승인·구현·검증 완료까지, 단체예약 API(PR #66) 선례 패턴 참고, develop 108커밋 merge. **(2026-09-11 갱신)** 전체 457개 테스트 정밀 재검증(통과 373·실패 84, #58 관련 실패 0건), [[배럴-Import-함정]]으로 3개 스펙이 조용히 안 돌던 문제 확정·수정, task-report S16이 9/8 커밋의 UI 재설계로 깨졌음을 확정(freeze로 실패가 승인 상태로 굳어짐), 실 토큰으로 백엔드 미배포 확인(이 서버는 없는 경로도 404 아닌 500 반환하는 함정), `/team/tree` contract-실배포 불일치 재확인 필요, 커밋 12개로 기능별 분리(단 타입 결합으로 `src/` 마이그레이션 자체는 못 쪼갬), 로그인/토큰 저장 경로 전체 미구현 확인

### 프로젝트/Commonly-FE
- [[Commonly-FE/프로젝트-현황]] — 경력증명서 발급 프로토타입, FE↔BE 갭 목록(개별등록/민원인 발급 막힘, 대량등록 데이터 미연결 버그, JWT 클레임 부족), 진행한 이슈·PR(#68–#77), juso 주소검색 키 적용, PR 코드리뷰에서 나온 동시성 disabled 스코프 버그·문서번호 빈값 통과 버그 수정, 로그인 무한 루프(#74/#75) 원인·수정. **(2026-09-10 갱신)** 노션 API 명세서 전수 확인 결과 민원인 본인 경력 조회 엔드포인트는 애초에 명세에 없었음이 확인돼(#76/#77) "BE가 GET /self를 신설해야 한다"는 이전 진단을 폐기하고 FE를 명세(선택 발급 불가, 항상 전체 발급)에 맞춰 재정렬, 401이 인가 실패까지 삼키는 메커니즘(accessDeniedHandler 부재) 확인

### 도구
- [[Claude-Code-MCP-서버-등록]] — MCP 서버 목록은 세션 부팅 시에만 로드됨, Desktop 내장 세션은 OAuth 브라우저 승인 불가(터미널 CLI 필요), 터미널을 도중에 닫아 토큰 교환이 빈 값으로 실패하는 패턴과 확인법, `/mcp` 메뉴 커서 위치를 놓쳐 엉뚱한 서버로 들어가는 함정, 자격증명 파일 직접 읽기 우회는 auto mode가 차단
- [[Git-브랜치명-샵-이스케이프]] — 브랜치명에 `#`이 있으면 쉘 주석으로 잘려서 `--delete` 등 뒤 인자가 사라짐, 항상 따옴표로 감싸야 함
- [[Claude-Code-느낌표-bash-접두사-채팅전용]] — `!command`는 채팅 입력 전용 즉시실행 기능, 실제 터미널(bash-input)에 그대로 붙여넣으면 `command not found: !node`로 실패
- [[Git-브랜치-뒤처짐-확인-없이-기능-없다고-단정]] — 로컬 브랜치가 base보다 수십~수백 커밋 뒤처지면 이미 merge된 기능도 "없다"고 오판하기 쉬움, `git fetch` 후 `git log HEAD..origin/<base>`로 뒤처짐부터 확인, "기존 구조 분석" 단계 전엔 먼저 merge/rebase
