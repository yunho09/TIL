# Wiki Log

append-only 작업 기록. 과거 항목은 수정하지 않는다.

## 2026-09-01 — init
- 생성: [[index]], [[log]]
- 비고: CLAUDE.md 운영 규칙 작성, raw/ 폴더 생성. 위키 시작.

## 2026-09-02 — ingest (spring-practice 학습 세션)
- 원본: spring-practice 프로젝트 Claude Code 세션 (raw/ 아님 — 세션에서 직접 생성)
- 생성: [[스프링-CRUD-계층구조]], [[JPA-엔티티와-리포지토리]], [[MySQL-연동]], [[프로젝트-현황]], [[학습-진행상황]]
- 비고: 처음에 llm-wiki/ 폴더를 중복 생성했다가 기존 wiki/로 통합. 주제별 하위 폴더(언어/Java/Spring, 프로젝트/spring-practice)는 사용자 요청으로 유지.

## 2026-09-04 09:00 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/spring-practice)
- 생성: [[빈과-DI]], [[스프링-컨테이너]] — 기존 페이지들이 참조만 하고 실제로는 없던 링크였음. 다중 구현체 주입(@Primary/@Qualifier/List<T>), 컨테이너=Map 비유, 컴포넌트 스캔 기준점·src 폴더 구조 등 이 세션에만 있던 내용으로 채움
- 갱신: [[스프링-CRUD-계층구조]] — Postman/curl 테스트 방법 섹션 추가
- 비고: 이 세션은 DI 실습(MessageSender 등)부터 CRUD·JPA·MySQL 연동까지 처음부터 다시 다룬 긴 튜터링 세션으로, 대부분 2026-09-02 ingest와 내용이 겹쳐 새 페이지 생성 없이 기존 페이지 보강 위주로 처리함. spring-practice는 세션 종료 시점까지도 git 저장소가 아님(git init 미완료) — [[학습-진행상황]] 참고.

## 2026-09-04 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[위키-자동-캡처]]
- 비고: 이 vault의 LLM Wiki 골격을 만들고, 이어서 SessionEnd 훅으로 세션 종료 시 자동 ingest + git push되는 파이프라인을 구축한 세션. OMC 내장 wiki 스킬 대신 CLAUDE.md+raw/+wiki/ 직접 구조를 택한 이유, 자동 캡처 동작 조건·스킵 규칙·적용 범위(Claude Desktop 일반 채팅은 미지원 등)를 정리함.

## 2026-09-04 14:00 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 갱신: [[위키-자동-캡처]] — "Notion 연동은 왜 안 붙였나" 섹션 추가 (진실의 원천 분리 위험, 위키링크 미지원, 이미 GitHub push가 웹 미러 역할을 하므로 자동 미러링 미채택)
- 비고: 동일 세션(/home/yunho)의 이어진 대화를 재캡처. 이전 캡처(같은 날짜, 위 항목)와 대부분 중복이라 새 페이지 없이 기존 페이지만 보강.

## 2026-09-05 — ingest (Orca IDE 설치 세션)
- 원본: Claude Code 세션 (/home/yunho, 2026-09-02~05) — raw/ 아님
- 생성: [[Orca-IDE-리눅스-설치]]
- 비고: 사용자가 "이거 llm wiki에 있나"로 조회했으나 vault에 Orca 관련 페이지 0건이었음. 해당 대화는 9/2~9/3로 SessionEnd 자동 캡처 구축(9/4) 이전이라 자동 ingest 대상이 아니었다. 사용자 선택에 따라 사용법 레퍼런스(공식 문서와 중복)는 제외하고 재발 가능성이 높은 리눅스 설치 트러블슈팅만 페이지화. wiki/환경/ 하위 폴더와 index의 "환경/리눅스" 섹션을 이번에 신설.

## 2026-09-06 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
- 생성: [[JOBIS-FE-V2/프로젝트-현황]], [[Figma-Dev-Mode-MCP]], [[S3-Presigned-URL-업로드]], [[CSS-Container-shrink-to-fit]]
- 비고: Figma 디자인(학생 지원하기 화면)을 기존 design-system 컴포넌트만으로 퍼블리싱하고 API 연동까지 진행한 세션. JOBIS-FE-V2는 이 vault에 처음 등장하는 프로젝트라 wiki/프로젝트/JOBIS-FE-V2/와 wiki/프론트엔드/ 폴더, index의 해당 섹션을 신설했다. 프로젝트 고유 정보(퍼블리싱·커밋 컨벤션, staging DNS 오지정, packages/api 공용 코드 결함, 백엔드 API 명세 함정)는 프로젝트 페이지로, 다른 프로젝트에도 재사용 가능한 개념(Figma Dev Mode MCP의 데스크톱/view-only 제약, S3 presigned URL 업로드 규칙과 Playwright 검증 한계, CSS shrink-to-fit과 max-width의 한계)은 프론트엔드/ 하위 별도 페이지로 분리했다. 실제 백엔드 서버가 DNS 문제로 접속 불가였던 것, 컴포넌트 인스펙터 값 요청·아이콘 위치 보류 같은 진행 중 판단은 재사용 가치가 낮아 제외.

## 2026-09-06 23:11 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[GNOME-Wayland-wl-clipboard-포커스-토스트]]
- 비고: 터미널 Claude에 이미지를 붙여넣을 때마다 뜨던 정체불명 GNOME 토스트를 진단한 세션. Mutter 50에 `zwlr_data_control`/`ext_data_control` 프로토콜이 없어 `wl-clipboard`가 숨은 `xdg_toplevel`+`xdg_activation_v1` 포커스 요청 폴백을 타는데, dconf에 남아있던 `focus-new-windows=strict`(과거 Claude Desktop 포커스 탈취 대응용이었으나 이미 `--ozone-platform=x11`로 해결되어 잔재였음) 때문에 Mutter가 요청을 거부해 GNOME Shell 토스트로 대신 뜨는 구조였다. `smart`로 복구해 해결. 겸사겸사 3일 넘게 돌던 `ws-monitor.sh` 디버깅 좀비 프로세스도 정리. wiki/환경/ 하위에 페이지 추가.
