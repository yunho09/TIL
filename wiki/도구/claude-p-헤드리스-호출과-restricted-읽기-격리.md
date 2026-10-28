---
tags: [claude-code, claude-p, 격리, 헤드리스]
updated: 2026-10-28
---

# claude -p 헤드리스 호출과 `--restricted` 읽기 격리

앱에서 Claude Code 구독을 API 키 없이 쓰려면 헤드리스 CLI `claude -p`를 자식 프로세스로 부르면 된다. 신뢰할 수 없는 레포를 읽게 할 때는 `--restricted`와 읽기 전용 도구 조합으로 레포 밖 접근을 막는다. PR Lens([[프로젝트-현황]])에서 쓴 구성이다.

## 공통 플래그
- `--output-format json|stream-json --json-schema <스키마>`: 결과는 `structured_output`에서 읽음.
- `--system-prompt`, `--tools ""`(도구 없이 분석만), `--setting-sources ""`, `--strict-mcp-config`, `--disable-slash-commands`, `--no-session-persistence`, `--max-budget-usd`.
- 환경변수 `CLAUDE_WIKI_CAPTURE=1 DISABLE_OMC=1`: 사용자의 SessionEnd 위키 훅과 OMC가 앱이 부른 세션에서 또 돌지 않게 함(이 vault의 [[위키-자동-캡처]] 훅 대비).
- `--bare`는 쓸 수 없음 — API 키 인증을 강제하기 때문.

## 깊게 보기(레포 읽기)
- `--restricted --tools "Read,Grep,Glob" --permission-mode dontAsk`, cwd는 레포 tarball을 푼 체크아웃.
- **`--restricted` 없이는 `/etc`도 읽혔다**(실험). 붙이면 레포 밖 읽기가 막히고 막힌 시도는 결과 JSON의 `permission_denials`에 기록됨.
- 악성 설정 파일·`CLAUDE.md`를 레포에 심어도 레포 밖 읽기/쓰기/명령 실행이 불가능함을 검증 에이전트가 확인.
- 함정: ripgrep은 git 레포 밖에서 `.gitignore`는 무시하지만 `.ignore`/`.rgignore`는 따르므로, 신뢰 못 할 레포는 내려받은 뒤 이 파일들을 지워야 Grep/Glob에서 파일을 못 숨김.
- 자식에게 넘기는 환경변수는 허용 목록으로(인증에 필요한 `CLAUDE*`만, `GITHUB_TOKEN`/`GH_TOKEN` 제거). 종료 시 프로세스 그룹째 kill.
- 비용 감각: 실제 PR 한 건 깊게 보기 약 50초, $0.2 상당(구독 사용량).

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
