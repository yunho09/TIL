---
tags: [claude-code, mcp, oauth]
updated: 2026-09-07
---

# Claude Code MCP 서버 등록·인증 동작

`claude mcp add`로 MCP 서버를 등록할 때 흔히 겪는 3가지 함정: 세션 부팅 후 등록 시 안 보이는 문제, Claude Desktop 내장 세션에서 OAuth가 안 되는 문제, 터미널을 도중에 닫아 토큰 교환이 조용히 실패하는 문제.

## MCP 서버 목록은 세션 부팅 시점에만 로드된다

실행 중인 Claude Code 세션 도중에 `claude mcp add`로 서버를 추가해도, **그 세션의 `/mcp` 목록에는 뜨지 않는다.** MCP 서버 목록은 세션이 시작될 때 한 번만 읽힌다. 설정 파일(`~/.claude.json`)에는 정상적으로 들어가 있어도 마찬가지다. 해결법은 세션을 완전히 새로 시작하는 것뿐이다.

스코프를 `User config`로 올려두면 등록 이후에 시작하는 모든 세션·모든 프로젝트 폴더에서 잡힌다.

## 독립 CLI vs Claude Desktop 내장 세션

같은 머신에 둘 다 있을 수 있다:

| 위치 | 용도 |
|---|---|
| `~/.local/bin/claude` (독립 설치) | 터미널용 CLI |
| `~/.config/Claude/claude-code/<version>/claude` (Desktop 내장) | Desktop 앱 안의 Code 탭 |

**둘 다 같은 `~/.claude.json`을 읽어서 등록은 공유된다.** 하지만 **OAuth 브라우저 승인은 독립 터미널 CLI 쪽에서 해야 한다.** Desktop에 내장된 세션은 시작 시점에 non-interactive로 표시되는 경우가 있어(다른 MCP 서버들도 같은 이유로 인증 대기 상태로 멈춰 있을 수 있다), 로그인 창을 띄우지 못한다. Desktop 앱 자체를 끌 필요는 없고, 터미널 창을 별도로 하나 열어 그쪽에서 `/mcp` → 인증을 진행하면 된다.

## OAuth 토큰 교환이 조용히 실패하는 패턴

`/mcp`에서 서버를 선택해 브라우저 승인까지 마쳤는데도 상태가 계속 `Needs authentication`으로 남는 경우가 있다. 확인 방법: `~/.claude/.credentials.json`의 `mcpOAuth` 항목에서 해당 서버의 `accessToken` 길이를 본다(값 자체는 출력하지 않고 길이만 확인). **`accessToken`이 0자(빈 문자열)로 저장돼 있으면 인증이 중간까지만 진행된 것**이다:

| 단계 | 결과 |
|---|---|
| 서버 discovery | 성공 |
| 클라이언트 등록 | 성공 |
| 브라우저 authorize | 도달 (그래서 "승인한 것처럼" 보임) |
| **코드 → 토큰 교환** | **실패 — accessToken 빈 값으로 저장** |

OAuth는 `http://localhost:<port>/callback`으로 콜백을 받는데, 이 서버는 **터미널의 claude 프로세스가 떠 있는 동안만 존재하는 임시 수신 서버**다. 흔한 실패 원인:

- 브라우저 승인 왕복이 끝나기 전에 **터미널 세션을 닫거나 다른 명령으로 넘어감** (가장 흔함)
- 브라우저가 승인 후 콜백 페이지("연결할 수 없음" 등)를 띄웠는데 그 전에 닫음
- 승인까지 시간이 오래 걸려 타임아웃

재시도할 때는 터미널 창을 그대로 두고, 브라우저가 `localhost:<port>` 콜백 페이지에서 성공 메시지를 띄울 때까지 기다린 뒤, `/mcp`를 다시 열어 상태가 `connected`로 바뀌었는지 확인해야 한다. 정상 완료되면 `accessToken`이 수백 자 길이로 채워지고 `mcp-needs-auth-cache.json`에서 해당 서버 항목이 제거된다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
