---
tags: [claude.ai, 크롬확장, 자동화, 대화기록]
updated: 2026-11-20
---

# claude.ai 대화기록을 Claude in Chrome으로 읽기

claude.ai에 로그인된 크롬에서 페이지 JS로 내부 API를 호출하면 대화 목록·본문을 통째로 읽을 수 있다. JS 도구 출력 한도(약 1,500자)는 DOM에 쓰고 페이지 텍스트로 읽어 우회한다. 내부 API라 공식 문서가 없고 바뀔 수 있다(Claude 보충: 실측은 한 세션뿐).

## 내부 API (페이지 안 fetch)
- `/api/organizations` → org ID
- `/api/organizations/{org}/chat_conversations?limit=500` → 대화 목록(이 계정 238개)
- `/api/organizations/{org}/projects?limit=200` → 프로젝트
- `/api/organizations/{org}/chat_conversations/{uuid}?tree=True&rendering_mode=messages` → 대화 본문

## 출력 한도 우회
- `javascript_tool` 반환값은 ~1,500자에서 잘린다 → 결과 텍스트를 `<pre id="dd">`에 넣고 `get_page_text`(최대 ~50k자)로 읽는다.
- 페이지 URL에는 쿼리 문자열을 붙이지 않는다(원 세션의 운용 규칙).
- 날짜 필터는 문자열 슬라이스 위치를 확인한다(`m.t.slice(5,10)`; 잘못 자르면 0건이 나왔음).

## 진행 순서
1. `tabs_context_mcp` → 새 탭 생성(기존 탭 재사용 금지) → claude.ai 이동
2. 목록 받아 제목·날짜로 관련 대화 선별 → 마스터 대화부터 본문 읽기
3. 끝나면 만든 탭 닫기

## 로컬 보조
- 구글 캘린더 일정은 EDS(gi: EDataServer·ECal·ICalGLib)를 `/usr/bin/python3`로 읽는다 — 함정은 [[EDS-구글-캘린더-연동-함정]]
- 대화 기록 검색 시 ugrep이 "정규식이 너무 복잡"하다며 거부하면 파이썬으로 대체

## 사용 예
- 공모전 29개 집계: [[공모전-출품-현황과-발표일-정리-2026-11-20]]

## 출처
- [[raw/Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)]]
