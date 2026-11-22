---
tags: [gnome, 클립보드, ghostty, paste-on-select, shift-insert]
updated: 2026-11-22
---

# 클립보드 기록(Super+V) 선택 즉시 붙여넣기 + Ghostty Shift+Insert 설정

Super+V 클립보드 목록에서 Enter/클릭만으로 바로 붙여넣게 하려면 클립보드 확장에 이미 있는 **"선택하면 바로 붙여넣기(paste-on-select)"** 설정을 켜면 된다(재로그인 불필요). 이 확장은 Ctrl+V 대신 **Shift+Insert**를 합성하므로 Ghostty에서는 추가 설정이 필요했다.

## 핵심
- 확장이 보내는 키가 **Shift+Insert**인데, Ghostty 기본값은 이 키가 **primary selection(마우스 드래그 선택)** 을 붙여넣음 → 엉뚱한 글자가 들어갈 수 있음. Ghostty 설정에서 Shift+Insert를 **클립보드 붙여넣기**로 재지정(원본은 `~/.config/ghostty/config.bak-20261122`).
- 붙여넣은 뒤 확장이 클립보드를 **원래 내용으로 복원**하는 게 기본 동작. 바로 Ctrl+V를 누르면 방금 고른 항목이 아니라 이전 내용이 붙는다.
- 실제 키 입력 확인은 사용자 몫으로 남겼음(결과는 세션에 없음).

관련: [[Ghostty-Ctrl+Enter-바로보내기-F11-전체화면]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
