# Wiki Index

Claude가 관리하는 페이지 카탈로그. 페이지당 한 줄 요약. 새 페이지가 생기거나 요약이 바뀌면 여기도 함께 갱신한다.

## 페이지

### 메타 (위키/vault 운영)
- [[위키-자동-캡처]] — Karpathy LLM Wiki 모델 채택 이유, SessionEnd 훅 기반 자동 ingest+push 파이프라인, 스킵 조건, 적용 범위/한계, Notion 자동 연동 미채택 이유

### 환경/리눅스
- [[Orca-IDE-리눅스-설치]] — AppImage type 2가 Ubuntu 26.04에서 안 열리는 이유(libfuse2 부재)와 압축 해제 설치법, `orca`↔GNOME 화면낭독기 이름 충돌, 자동 업데이트 불가 등 한계
- [[GNOME-Wayland-wl-clipboard-포커스-토스트]] — Mutter 50에 data-control 프로토콜 부재→wl-clipboard 폴백 경로→`focus-new-windows=strict`가 겹쳐 뜨는 정체불명 토스트의 원인 체인과 `smart` 복구법
- [[절전-복귀-지연-원인과-zram-도입]] — 스왑 고갈+복귀 직후 systemd 타이머 폭주가 겹쳐 절전 복귀가 9초 넘게 걸리던 원인, zram 압축 스왑 도입(설치 시점 기본값 함정 포함)과 타이머 완화 조치

### 언어/Java/Spring
- [[빈과-DI]] — 인터페이스+구현체+생성자 주입 패턴, 다중 구현체 주입(@Primary/@Qualifier/List<T>), 싱글톤 규칙, 실무 사용 빈도
- [[스프링-컨테이너]] — 컨테이너=Map 비유, 빈은 메모리에만 존재, 조립 순서, @SpringBootApplication 구성, 컴포넌트 스캔 기준점과 src 폴더 구조
- [[스프링-CRUD-계층구조]] — Controller/Service/Repository 계층별 역할, 요청 값 추출 3종(@PathVariable/@RequestParam/@RequestBody), URL+메서드 매핑, Postman/curl 테스트 방법
- [[JPA-엔티티와-리포지토리]] — @Entity 규칙(빈 생성자, Long id, setter 대신 update()), JpaRepository가 빈 인터페이스로 동작하는 원리와 쿼리 메서드
- [[MySQL-연동]] — datasource 설정, ddl-auto 선택지, Docker MySQL 컨테이너 명령어 모음

### 프론트엔드
- [[Figma-Dev-Mode-MCP]] — 데스크톱 앱에 파일이 열려 있어야 동작, view-only 파일에서 씬 그래프 API 실패·스크린샷 폴백과 픽셀 계측 대안
- [[S3-Presigned-URL-업로드]] — presign 2단계 흐름, Authorization 헤더가 서명을 깨는 이유, S3 CORS 필요성, Playwright의 Blob 바디 목킹 한계
- [[CSS-Container-shrink-to-fit]] — `max-width`는 상한일 뿐 폭을 확보 못 함, `main` shrink-to-fit 원인과 해결, 전체 배경색 우회법

### 프로젝트/spring-practice
- [[프로젝트-현황]] — spring-practice 구조·API·DB 환경·알려진 허점 스냅샷 (2026-09-02)
- [[학습-진행상황]] — 스프링 로드맵 진행 상태, 겪은 에러들, 다음 단계 후보

### 프로젝트/JOBIS-FE-V2
- [[JOBIS-FE-V2/프로젝트-현황]] — 퍼블리싱·커밋 컨벤션, staging DNS 오지정, `packages/api` 공용 코드 결함(presign/더블슬래시/401 이중의미), API 명세 함정 (2026-09-06)
