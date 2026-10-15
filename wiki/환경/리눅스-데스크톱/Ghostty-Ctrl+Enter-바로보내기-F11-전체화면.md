---
tags: [ghostty, claude-code, tmux, keybinding]
updated: 2026-10-15
---

# Ghostty Ctrl+Enter 바로 보내기 + F11 전체화면

Ghostty 기본 키는 **Ctrl+Enter = 전체화면 토글**이라 Claude Code의 "바로 보내기(`chat:sendNow`)"와 충돌한다. Ghostty에서 Ctrl+Enter를 다시 매핑하고 전체화면은 F11로 옮겼다. tmux 안에서 Claude Code를 쓰면 안내 문구가 `Ctrl+X Ctrl+S`로 바뀌는 이유와 해결도 함께 정리한다.

## 최종 설정 (`~/.config/ghostty/config`)
- `keybind = f11=toggle_fullscreen` — F11로 전체화면 토글
- Ctrl+Enter는 기본 전체화면 바인딩을 덮어써야 한다. 시도 순서: ① `ctrl+enter=text:\r`(Enter와 동일) → ② Ctrl+X Ctrl+S 신호를 보내게 함 → ③ **최종: 진짜 Ctrl+Enter 신호를 Claude까지 그대로 전달**(정확한 keybind 줄은 세션에 기록 안 됨, (Claude 보충) `~/.config/ghostty/config` 확인 필요)
- 처음 시도한 `ctrl+enter=text:\r`(= Enter와 동일 신호)은 **Claude가 답하는 중에 누르면 끼어들기가 아니라 대기열에 들어가서** 틀린 해법이었다. `chat:sendNow`는 답변 도중 즉시 끼어들어 보내는 동작이고 `chat:submit`(Enter)은 대기열행.

## 설정 반영
- 실행 중인 Ghostty는 설정 파일을 바꿔도 즉시 안 읽는다 → **Ctrl+Shift+,** (설정 리로드) 또는 재시작. "F11 눌렀더니 `~`가 찍힘 / Ctrl+Enter가 여전히 전체화면"은 이 때문이었다.

## 안내 문구가 `Ctrl+X Ctrl+S`로 뜨는 이유
- Claude Code는 **tmux 안에서는 Ctrl+Enter가 전달된다고 확신하지 못해**, 바로 보내기 키가 둘(`ctrl+x ctrl+s`, `ctrl+enter`)이면 문구에 대체 키 `Ctrl+X Ctrl+S`를 보여준다. tmux 밖에서 실행하면 Ctrl+Enter로 표시된다.
- 기본 `chat:sendNow` 바인딩 = `ctrl+x ctrl+s`, `ctrl+enter` (둘 다 바로 보내기).
- 해결(문구까지 Ctrl+Enter로 통일): `~/.claude/keybindings.json`의 Chat 컨텍스트에서 `"ctrl+x ctrl+s": null`로 기본 바인딩을 빼고, Ghostty는 진짜 Ctrl+Enter(수정자 포함 키 신호)를 tmux에 넘기게 한다. tmux 쪽이 확장 키(extended keys)를 받는 모드여야 한다 (Claude가 도는 pane이 그 모드인 것은 확인함).
- 사용자 keybindings는 **기본값에 추가(additive)**되므로 기본 키를 없애려면 `null`로 명시 해제해야 한다.

## 부작용: Ctrl+S 흐름 제어
- Ctrl+Enter를 `Ctrl+X Ctrl+S` 바이트로 보내는 구성이면 셸에서 잘못 누를 때 Ctrl+S(XOFF)로 터미널이 멈출 수 있어 `~/.zshrc`에 `unsetopt flow_control`을 넣는다. 진짜 Ctrl+Enter를 보내는 최종 구성에서는 불필요해서 다시 뺐다.

## 검증 결과
- 쉬는 중 Ctrl+Enter 전송, 답변 도중 Ctrl+Enter 끼어들기(대기열 아님), Shift+Enter 줄바꿈, F11 전체화면, 안내 문구 `Ctrl+Enter` 표시 모두 사용자가 확인.

## 관련
- [[PC-중복-설치-정리-Homebrew-도입]] — Ghostty(deb)·tmux 설치 구성

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-10-15
