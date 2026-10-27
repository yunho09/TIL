---
tags: [zsh, dotenv, 시크릿, Claude Code, 보안]
updated: 2026-10-27
---

# `.env`를 셸로 `source`하면 `KEY= 값`의 공백 때문에 시크릿이 오류 메시지로 출력된다

AI 에이전트가 `.env`의 키를 쓰려고 `set -a; . ./.env`로 셸에 읽었다가, `GEMINI_API_KEY= AIza…`처럼 `=` 뒤에 공백이 있어 셸이 값을 **명령어로 실행**하려 했고 `command not found: <키 전체>`가 도구 출력에 찍혔다. 그 출력은 세션 기록과 모델 API 전송 내용에 남는다.

## 원인
- 셸 문법에서 `KEY= value`는 "KEY를 빈 값으로 하고 `value` 명령 실행"이다. 값이 그대로 `command not found` 메시지에 들어간다.
- Spring의 `.env`(`optional:file:.env[.properties]`)는 값 앞 공백을 무시해 앱 실행엔 문제가 없어서 오탈자가 숨어 있다. 파일에서 `=` 뒤 공백은 지우는 게 깔끔.

## 남는 곳 / 안 남는 곳
- 남음: 로컬 세션 기록 파일(`~/.claude/projects/.../*.jsonl`), 모델 API로 전송된 대화 내용. 사용자 화면에는 도구 출력이 접혀 안 보일 수 있어 "찍은 적 없다"고 느끼기 쉽다.
- 안 남음: git 커밋·PR·이슈·저장소 파일.
- 위험은 낮지만 안전하게는 키를 재발급. 최종 판단은 키 소유자.

## 예방
- 시크릿 파일은 **셸로 source/eval하지 말고** 파일 파서(예: Python 문자열 파싱)로 읽어 값은 출력하지 않고 마스킹.
- 시크릿은 채팅에 붙여 넣지 말고 레포 밖 파일(`chmod 600`) 또는 `.env`에 직접 저장. 검색은 파일 이름만 대상으로 하고 값을 출력하지 않는다.
- 다른 터미널의 `export`는 에이전트 셸에 안 보이고, `~/.zshrc`/`~/.zshenv`는 에이전트 재시작 후 반영, IntelliJ 실행 구성 값은 `.idea/workspace.xml`(git 제외)에 저장된다.

관련: [[zsh-단어분리-안됨-glob-nomatch-함정]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/bium-be)
