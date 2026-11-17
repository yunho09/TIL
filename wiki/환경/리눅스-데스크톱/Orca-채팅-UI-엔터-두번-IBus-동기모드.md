---
tags: [orca, ibus, 한글, electron, linux]
updated: 2026-11-17
---

# Orca 채팅 UI 엔터 두 번 눌러야 전송 — IBus 동기 모드

Orca 채팅 UI에서 한글을 치고 엔터를 누르면 첫 엔터가 버려져 두 번 눌러야 전송되던 문제. 원인은 리눅스 IBus의 **비동기 키 처리**와 Orca의 엔터 판정 로직이 어긋나는 것이고, Orca 실행 시에만 `IBUS_ENABLE_SYNC_MODE=1`을 주면 우회된다.

## 원인

- IBus는 기본이 **비동기** 방식이다. 엔터를 누르면 입력기가 "조합 끝(composition end)" 신호를 **빈 글자**로 보내고, 마지막 글자(예: "안녕"의 '녕')는 그 뒤에 따로 커밋한다.
- Orca는 조합 끝 신호에 한글이 실려 와야 그 엔터를 전송으로 친다. 빈 글자가 오면 일본어 후보 선택용 엔터로 보고 **버린다**.
- 이 한글 예외 처리는 윈도우용만 고쳐졌다(Orca PR #30192). 확인 시점 최신 1.4.261에도 리눅스용 수정은 없었다.

## 확인 방법

- 숨은 가상 화면에서 Orca와 같은 Electron 버전으로 "안녕"+엔터를 쳐 이벤트 순서를 기록하고, 그 기록을 Orca의 엔터 처리 코드에 그대로 넣어 재현했다.
- 현재 방식(비동기): 엔터 두 개가 모두 버려짐 → 사용자 증상과 일치
- 동기 방식: 엔터 한 번에 "안녕" 전송, 빠르게 쳐도 글자 순서 안 꼬임

## 해결

- `~/.local/share/applications/orca-ide.desktop`의 `Exec=` 앞에 `env IBUS_ENABLE_SYNC_MODE=1`을 붙인다. Orca를 띄울 때만 입력기를 동기 방식으로 쓴다.
- 원본은 `orca-ide.desktop.bak.20261117-ibus`로 백업.
- 적용하려면 Orca를 Ctrl+Q로 완전히 종료 후 독에서 다시 실행(터미널 세션은 유지됨).

## 한계

- **실제 Orca 채팅창에서의 최종 확인은 못 했다.** 코드 재현 단계까지만 검증(재실행 후 확인 필요).
- 근본 수정은 Orca 쪽 몫이라 업스트림 이슈 제안이 남아 있다.

관련: [[Orca-IDE-리눅스-설치]], [[Electron-Wayland-한글-IME-Chromium136-기본-활성화]], [[GTK4-파일선택창-첫글자-영어-ibus-한글]], [[Figma-한글-입력-Wayland-IME]]

## 출처

Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
