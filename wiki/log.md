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
