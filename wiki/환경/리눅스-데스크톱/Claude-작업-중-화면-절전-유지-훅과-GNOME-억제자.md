---
tags: [GNOME, claude-code, 절전, 훅, systemd, inhibitor]
updated: 2026-10-21
---

# Claude 작업 중 화면·절전 유지 (훅 + GNOME 억제자)

Claude Code가 일하는 동안만 GNOME에 "화면 끄기·절전 금지"를 걸고, 모든 Claude가 끝나면 풀어 주는 구성이다. Claude Code 훅이 표지 파일을 남기고, 백그라운드 유저 서비스가 표지가 하나라도 있으면 GNOME 세션 억제자를 쥔다.

## 출발점: 작업이 멈추는 진짜 원인
- 화면 자동 끄기(`idle-delay`)는 이미 0(안 함)이었다. 사용자가 겪던 "화면 꺼지면 멈춤"은 **배터리 상태 입력 없이 15분 후 자동 절전(suspend, 900초)** 이었다.
- 충전 중(AC)에는 자동 절전이 `nothing`이라 안 멈춘다.
- 뚜껑 닫기는 별도 경로: `lid-grace` 서비스가 잠금 후 10초 뒤 절전(도킹 시 예외). 사용자가 **그대로 두기로 결정** (가방 안 발열 우려, 2026-10-21).

## 구성
- 훅 `~/.claude/hooks/claude-awake-hook.sh` — `settings.json`의 14개 이벤트(UserPromptSubmit, PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest, Notification, Elicitation, PreCompact, PostCompact, Stop, StopFailure, SubagentStart, SubagentStop, SessionEnd)에 **동기** command 훅으로 등록.
  - 표지 파일은 tmpfs `/run/user/1000/claude-awake/`에 `S_<session>` / `A_<session>_<agent>`(서브에이전트는 따로 셈).
  - 작업 중 이벤트는 표지 생성, `Stop`·`PermissionRequest`·`Elicitation`·`AskUserQuestion`/`ExitPlanMode`의 PreToolUse는 삭제. 사용자 응답을 기다리는 동안은 아무것도 안 돌므로 "끝난 상태"로 본다.
  - 표지에는 조상 프로세스 중 comm이 `claude`인 pid를 적는다(`/proc/<pid>/stat` 순회).
  - 어떤 경우에도 `exit 0`(Claude를 막지 않음), session_id는 `tr -cd 'A-Za-z0-9-'`로 정리(경로 탈출 방지). 호출당 약 22ms.
  - 동기 훅으로 둔 이유: 비동기면 Stop과 PostToolUse 순서가 뒤집혀 표지가 잘못 남을 수 있음.
- 서비스 `~/.local/bin/claude-awake`(python) + `~/.config/systemd/user/claude-awake.service` (`graphical-session.target`에 묶음, `lid-grace.service`와 같은 패턴).
  - `Gio.FileMonitor`(inotify) + 300ms 디바운스 + 60초 점검.
  - 표지가 있으면 `org.gnome.SessionManager.Inhibit('claude-awake', 0, 'Claude 작업 중', 12)` → 쿠키, 없으면 `Uninhibit`.
  - 죽은 표지 정리: pid가 살아 있는 `claude`가 아니거나, 훅 신호 없이 `CLAUDE_AWAKE_STALE_SEC`(기본 1800초) 지나면 버림.

## 핵심 개념
- **GNOME `Inhibit` 플래그**: idle=8, suspend=4 → 12면 둘 다 막음. 억제자는 D-Bus 연결에 묶여 프로세스가 죽으면 자동 해제된다.
- `gsd-power`는 idle 억제 중 dim/blank/sleep 감시를 끈다. mutter idle watch는 `마지막 입력 + timeout`이라, **해제 시점에 이미 timeout 이상 놀았다면 즉시 절전**에 들어간다.
- `Stop` 훅은 **Esc 중단 때 발동하지 않는다** → 표지가 남아 최대 30분(stale 한도) 동안 억제.
- 시스템 변경은 `/usr/bin/gsettings`(linuxbrew gsettings는 no-op), `pkexec`(sudo 아님)를 쓴다 (Claude 보충: 사용자 환경 선호).

## 검증과 한계
- 확인함: 억제 on/off 전환, 죽은 표지 정리, 훅 실제 발동(현재 세션·다른 세션), 억제 중 절전 안 함 — AC 절전 시간을 60초로 줄이고 idle 305초 방치해도 절전 없음. 설정은 원복.
- 첫 테스트는 도중에 충전기가 꽂혀 무효(AC는 원래 절전 안 함)였고, AC 설정(`sleep-inactive-ac-type suspend`, 60초)으로 재실험해 통과.
- **미확인**: 끝난 뒤 실제로 잠드는 해제 쪽(노트북을 재워야 해서).
- 한계: `run_in_background`만 남기고 턴이 끝나면 감지 못 함 / Esc 중단 후 최대 30분 유지 / 작업 중 화면이 밝은 채라 배터리 더 씀 / Claude Desktop 채팅(Code 아님)은 대상 아님.

## 끄는 법·로그
- `systemctl --user disable --now claude-awake`, 훅 제거는 `~/.claude/settings.json.bak-20261021-awake` 백업 참고.
- 로그: `journalctl --user -u claude-awake` ("inhibit on (N busy)" / "inhibit off").

## 관련
- [[절전-복귀-지연-원인과-zram-도입]] — 같은 노트북의 절전 진입·복귀 문제, 억제자가 절전 지연을 만든 사례

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
