---
tags: [zaemit, mcp, 공모전, oauth]
updated: 2026-09-08
---

# Zaemit(재밋) MCP 연동

재밋(Zaemit)은 AI 웹사이트 빌더이고, MCP 서버로 사이트 제작·운영 기능 대부분을 노출한다. "제2회 Zaemit AI 웹사이트 공모전" 출품을 위해 이 MCP를 붙이는 과정에서 엔드포인트 함정, 게이트웨이형 툴 구조, OAuth 인증 이슈를 확인했다.

## 공모전 제약 (2026-09-07 기준)

- 마감: 2026-09-18
- 제출물: 재밋으로 만든 사이트 URL + 향후 이용 계획 + 만족도 설문
- 심사 키워드: "이후에도 실제로 계속 쓸 수 있는 사이트"
- **절대 제약**: 사이트는 반드시 재밋 MCP로 만들어야 한다. React/Next/HTML을 직접 작성해 배포하면 출품 무효. 재밋 MCP 연동 이력이 1회 이상 있어야 유효 출품작.

## 엔드포인트 함정 — `mcp.zaemit.ai` ≠ MCP 서버

재밋 공식 안내에서 흔히 보이는 `https://mcp.zaemit.ai`는 **MCP 엔드포인트가 아니라 연결 안내용 랜딩 페이지**다. 이 URL로 `initialize`를 보내면 JSON이 아니라 HTML(`<title>Zaemit MCP 연결 안내</title>`)이 돌아와서 핸드셰이크가 깨진다.

**정본 엔드포인트는 `https://zaemit.ai/mcp`**다. 여기는 무인증 `initialize`가 정상 응답한다:
```
serverInfo: {"name":"zaemit-mcp","title":"재밋(Zaemit) 홈페이지 제작·운영","version":"1.0"}
```

등록 명령:
```bash
claude mcp add --transport http zaemit https://zaemit.ai/mcp
```

## 게이트웨이 3종 구조

`tools/list`는 인증 없이 공개돼 있지만, 노출되는 툴은 딱 3개뿐인 **검색형 게이트웨이** 패턴이다. 실제 기능(공모전 시점 확인: 403개 툴, 57개 그룹)은 그 뒤에 숨어 있고 AI가 검색해서 찾아 쓰는 방식이다.

| 툴 | 역할 |
|---|---|
| `tool_categories` | 기능 그룹 목록 + 그룹별 툴 개수. 뭘 할 수 있는지 모를 때 제일 먼저 호출 |
| `tool_search` | 한국어/영어로 실제 툴을 검색해 `inputSchema`(호출 규격) 획득. `tool_invoke` 전 필수 |
| `tool_invoke` | 실제 실행. `site_id`로 대상 사이트 지정, 삭제·환불 등 비가역 작업은 서버가 `confirm_token`을 돌려주고 `confirmed=true` 재호출을 요구 |

`initialize`와 `tools/list`는 무인증이지만 **`tools/call`(=`tool_invoke` 등 실제 실행)은 전부 OAuth 2.1 필수**다:
```
{"error":"invalid_token","error_description":"OAuth 2.1 access token required"}
```

## 기능 맵 (공모전 조사 시점, 403툴/57그룹)

- **바로 사용 가능 — 41그룹 187툴**: `page`(페이지 생성·수정·발행, 블록 조립이 기본이고 커스텀 HTML은 `page_publish` 예외 경로), `member`, `sheet`(폼 제출물·목록 데이터 저장소), `component`(헤더/푸터/GNB), `seo`, `stats`, `menu`/`domain`/`language`, `block`/`design`/`frontend`, `popup`/`floating`/`widget`, `image`/`file`/`media`, `systempage`(로그인/마이페이지 표준 페이지) 등.
- **플러그인 설치가 필요 — 16그룹 216툴**: **`board`(게시판, 23툴, `plugin_install(boardsys)`)**, **`inquiry`(문의·신청 폼, 22툴, `plugin_install(inquirykit)`)**, `product`/`order`/`shipping`/`coupon`/`payment`(쇼핑몰 일체, 81툴), `reservation`(예약), `lecture`(강의·수강), `aichatbot`, `notify`/`webhook`/`snslogin`/`autotranslate`/`map`/`instafeed`.
- 폼 제출물은 `sheet`(데이터 시트)에 쌓이는 구조라 폼+목록 조합은 설계상 지원된다.
- 미확인(공모전 조사 세션 종료 시점): Free 플랜에서 `boardsys`/`inquirykit` 실제 설치 가능 여부, `design_guide`→`design_blocks`로 확인하는 CSS/디자인 커스터마이징 한계.

## Free 플랜 제약

- 페이지 10개 상한
- 내 도메인 연결 불가(유료 전용) → 출품 URL은 zaemit 서브도메인
- 트래픽 300MB / 저장공간 500MB
- SSL·검색엔진 노출·다국어는 Free에도 포함
- 가입 시 AI 크레딧 20만 지급, 신용카드 불필요

심사 키워드가 "이후에도 계속 쓸 수 있는 사이트"라, 10페이지 제한은 오히려 주제를 좁게 잡는 게 유리하다는 신호로 해석됨(Claude 보충).

## OAuth 인증 관련 이슈

세션 부팅 후 등록·터미널 vs Desktop 차이·토큰 교환 실패 패턴은 재밋에 국한된 문제가 아니라 Claude Code의 일반적인 MCP 등록 동작이라 별도로 [[Claude-Code-MCP-서버-등록]]에 정리했다.

**2026-09-08: 인증 완료 확인됨.** 터미널 세션에서 재시도해 `accessToken`이 753자로 정상 저장됐고, 부여된 `scope`는 `mcp pii newsite`다 — `newsite`가 포함돼 있어 사이트 생성 권한까지 확보됐다. 이로써 공모전 절대 제약 중 "재밋 MCP 연동 이력이 1회 이상 있어야 유효 출품작"은 충족됐다. 이후 3단계(멀티 페이지·이미지 업로드·CSS 조정·폼/게시판·반응형·발행 URL 실기능 테스트)는 터미널 세션으로 인계되어 이 페이지 작성 시점엔 아직 미완료.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
