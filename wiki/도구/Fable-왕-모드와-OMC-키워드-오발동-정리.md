---
tags: [claude-code, omc, fable, 멀티에이전트, 서브에이전트, 훅]
updated: 2026-11-20
---

# Fable 왕 모드와 OMC 키워드 오발동 정리

멀티에이전트 작업에서 **Fable은 지휘자(왕), 나머지 모델은 일꾼**으로 두는 구조를 `~/.claude/CLAUDE.md` 규칙과 `settings.json` 환경변수로 만든 기록이다. 함께, OMC 키워드(ulw·autopilot 등)가 한국어 질문에도 켜져 Stop 훅이 턴 종료를 막는 문제와 정리 규칙을 남긴다.

## Fable 왕 모드 구성
- **`~/.claude/CLAUDE.md`에 "Fable 왕 모드" 절**을 둔다. OMC 관리 영역 밖에 두면 OMC를 재설치해도 남는다.
  - 왕: 계획 → 작업 분배 → 결과 판정만. 검색은 Haiku(`explore`), 구현·디버깅은 Sonnet(`executor` 등, 어려운 것만 Opus), 설계·리뷰·보안은 Opus, 검증은 **만든 일꾼과 다른 일꾼**.
  - 일꾼에게 `model: "fable"`을 주지 않는다. `fork` 서브에이전트는 Fable을 그대로 물려받아 쓰지 않는다.
  - 규칙은 메인 세션 모델이 Fable(`claude-fable-*`)일 때만 적용, 서브에이전트는 무시.
- **`settings.json`에 `CLAUDE_CODE_SUBAGENT_MODEL=opus`**: 이게 없으면 모델이 정해지지 않은 일꾼이 부모(Fable)를 물려받아 전부 Fable로 돈다.
- 확인 결과: Fable 세션에서 일꾼을 띄우니 지휘자 `claude-fable-5-2`, 일꾼 `claude-opus-5-5`, 왕 규칙 적용됨.
- 켜는 법: 모델 메뉴에서 Fable 선택, 또는 `claude --model fable`. 기본 모델은 Opus 그대로.

## 자동 전환은 안 된다 → 선택창 규칙
- 멀티에이전트를 시작해도 모델 선택창은 안 뜨고 원래 모델로 계속 돈다. 데스크톱 앱이 **세션이 스스로 모델을 바꾸는 것을 막고**, Opus 세션에서 더 비싼 Fable 세션을 띄우는 것도 안 된다.
- 대안으로 CLAUDE.md에 규칙을 추가: Fable이 아닌 세션에서 멀티에이전트 신호(OMC 키워드, `/team`·Workflow, "멀티에이전트"·"병렬로 맡겨")가 오면 일 시작 전 AskUserQuestion으로 "Fable 왕 모드로 할까요?"를 묻는다.
  - "Fable로 바꿀게" → 모델 메뉴에서 고른 뒤 "계속"이라고 보내라고 안내하고 멈춤. "그냥 진행" → 현재 모델로 진행, 세션 내 재질문 없음.
  - 묻지 않는 경우: 이미 Fable, 서브에이전트 한두 개짜리 조회, `claude -p` 같은 비대화형.
- 실제 데스크톱 세션에서 선택창이 뜨는 것은 **미확인**(모델에게 상황을 주고 첫 행동을 묻는 방식으로만 검증).

## 어디서 적용되나
- **적용**: 데스크톱 앱 Code 탭, 터미널 `claude` — `~/.claude`의 CLAUDE.md·settings.json을 읽는 로컬 Claude Code.
- **미적용**: claude.ai 웹 채팅·Chat 탭(파일도 안 읽고 서브에이전트도 없음), 클라우드 세션(로컬 `~/.claude` 못 봄), OMO·Cowork(자체 지휘 체계, [[oh-my-openagent-omo-설치와-OMC-비교]]).

## OMC 모드 요약 (OMC 4.15.7)
- **ulw(ultrawork)**: 독립 작업을 일꾼 여럿에게 동시에. 속도만, 검증·끈질김 없음. OMC 기본 실행 방식.
- **ralph**: ulw + 끝날 때까지 반복. 리뷰어가 통과시킬 때까지 고치고 재검사. 사람이 필요한 상황이거나 stop/cancel에서만 멈춤.
- **autopilot**: 한 줄 요구에서 요구 구체화 → 계획 → 구현(ralph+ulw) → 테스트 반복 → 설계·보안·코드 리뷰 병렬까지 전자동.
- **team**: 공용 할 일 목록에 N명이 달라붙음. 계획→요구서→실행→검증→수정. `/oh-my-claudecode:team 3:executor "작업"`. 관계: ulw ⊂ ralph ⊂ autopilot, team은 별개.
- 사용: 키워드는 `/` 없이 메시지 아무 데나 넣으면 켜진다. 한글(`울트라워크`, `랄프`, `오토파일럿`)로도 켜지고 autopilot은 "build me an app"류 영어 문장도 감지. **team만** 일꾼 지시에 "team"이 들어가 팀이 계속 생기는 사고를 막으려고 단어 감지가 꺼져 있어 슬래시로 써야 한다.
- 중단: `cancelomc` 또는 `/oh-my-claudecode:cancel`(`--force`).

## 키워드 오발동 문제와 정리 규칙
- **증상**: "autopilot 어떻게 써?" 같은 질문에도 상태 파일이 생기고, 답을 끝내려 할 때 Stop 훅이 "Autopilot not complete. Continue working"으로 턴 종료를 막는다. 영어 질문은 질문으로 알아보고 넘어가지만 **한국어 질문은 못 알아본다**.
- **정리**: `state_list_active`로 켜진 모드·세션 ID 확인 → 모드별 `state_clear(mode, session_id)` → 마지막에 `state_clear(mode="skill-active", session_id)`. 이 도구들은 deferred라 먼저 ToolSearch로 불러와야 한다. 사용자가 직접 풀려면 `cancelomc`.
- 순서 주의: cancel 스킬의 bash 폴백은 autopilot·omc-teams에 쓰면 안 된다(autopilot은 `state_write(active=false)`로 재개 데이터 보존 필요).
- **CLAUDE.md 규칙화**: 매직 키워드 훅이 떠도 사용자가 실행이 아닌 질문·이야기만 하면 스킬은 실행하지 않고 답만, 같은 턴 안에 위 정리를 수행하고 답 끝에 한 줄로 알림. 사용자가 실제로 시킨 모드·다른 세션 상태는 지우지 않고, 애매하면 먼저 묻는다.
- 검증 한계: `claude -p` 시험에서는 질문으로 판단해 실행하지 않고 정리를 시도했으나 도구가 자동 차단되어 실제 삭제는 못 봤다([[claude-p-헤드리스-호출과-restricted-읽기-격리]]). 데스크톱 세션에서 같은 도구로 수동 정리한 것은 성공.

## 멀티에이전트 2D 사무실 시각화 (Claude Office Visualizer)
- 메인 에이전트=사장, 서브에이전트=직원 캐릭터로 나와 일하고 말풍선이 뜨는 웹앱(localhost:8000, 리눅스 가능). Claude Code 훅으로 연결. 다른 후보: Pixel Agents(VS Code 확장), Claude Backstage(플러그인).
- 설치 후 `agent-office-on`/`agent-office-off`(systemd 사용자 서비스, 로그 `journalctl --user -u agent-office`)로 켜고 끈다. 기존 훅(Clawd, Orca, 위키 캡처)은 그대로 두고 `settings.json`에 훅만 **추가**했고, 꺼 두면 훅이 아무 일도 안 한다.
- 설치 전에 열린 세션은 훅을 못 읽어 안 보일 수 있다 → 새 세션에서 확인.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.config/Claude/scratch-workspaces/68eef3b8-da94-4db5-b563-9565e29705a3/4cebf249-3d72-4a77-a55b-e742194ef06a/scratch-2026-11-19-47ab78, `.omc/session-capture/20261120-202621-56748899.txt`)
- 관련: [[Claude-모델-선택-Opus-5.5-vs-Fable-5.2와-자동-선택-기능]], [[Claude-Code-서브에이전트-fan-out-토큰-과다소모]]
