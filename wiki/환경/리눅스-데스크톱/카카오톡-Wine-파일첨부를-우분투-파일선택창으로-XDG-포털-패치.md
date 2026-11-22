---
tags: [카카오톡, wine, xdg-portal, 파일선택창, comdlg32, 패치]
updated: 2026-11-22
---

# 카카오톡(Wine) 파일 첨부를 우분투 파일 선택 창으로 — XDG 포털 패치

Wine 카카오톡의 첨부 버튼이 Wine 자체 파일 창 대신 우분투(GTK) 파일 선택 창을 띄우도록, Wine 개발자가 올렸다 닫힌 **XDG Desktop Portal 초안 패치**(Wine MR 10060)를 카카오톡 전용 Wine 11.10에 붙이고 보완했다. 재실행 후 카카오톡 창에 붙은 우분투 창이 뜨는 것까지 사용자가 확인했다("잘됨").

## 적용 내용
- 패치 대상은 `comdlg32`(파일 대화상자). 카카오톡 전용 Wine 안의 **파일 3개만** 바뀌어 다른 앱엔 영향이 없다.
- 초안 패치에 더한 보완 세 가지:
  - 카카오톡이 **32비트**라 wow64 표에 연결(`__wine_unix_call_wow64_funcs`). 32/64비트 구조체 레이아웃이 같아 64비트 진입점 재사용 가능(리뷰어가 오프셋 대조로 확인).
  - 파일 창이 카카오톡 창의 **부모(`x11:<XID>`)** 로 붙어 뜨게 함.
  - 포털 호출 동안 메시지 루프를 돌려 카카오톡이 **"응답 없음"** 으로 안 멈추게 함.
- 이미 떠 있는 카카오톡은 옛 파일을 로드한 상태라 **트레이에서 완전 종료 후 재실행**해야 적용된다.

## 코드 리뷰로 고친 문제
- **동시에 두 번째 대화상자**(다른 채팅창): libdbus 공유 연결(`dbus_bus_get`)에서 두 호출이 서로 막혀 창이 굳을 수 있음 → 진행 중이면 `STATUS_NOT_SUPPORTED`를 돌려 **두 번째는 기존 Wine 창으로 폴백**(busy 플래그). 의도된 동작이다.
- 포털이 응답 안 하거나 앱 종료 시 WM_QUIT를 삼켜 프로세스가 안 끝나는 문제 → 종료 신호·owner 창 소멸 시 대기에서 빠져나오게 함.
- 닫힐 때 무조건 `SetForegroundWindow`를 부르면 다른 작업 공간으로 끌려갈 수 있어 제거.
- 큰 params 구조체(약 147KB)를 스택 대신 힙으로(카톡 주 스레드 스택 1MB 대비).

## 검증·운영 메모
- 빌드 타깃 이름은 `dlls/comdlg32/all`. 원본은 건드리지 않고 시험용 Wine 사본+Xvfb 시험 prefix에서 **가짜 포털**로 다중 선택(한글 이름)·새 방식 열기·저장 경로 반환을 검증.
- 포털 호출 감시로 실제 카톡이 포털을 부르는지 확인(재실행 전 잡힌 호출은 크롬 등 다른 앱 것).
- **되돌리기**: `~/.local/share/wine-kakao/backup-comdlg32-20261122`의 원본 두 파일을 제자리에 복사하고 추가한 `comdlg32.so` 삭제.
- 소스·빌드물은 `/data/build/wine-portal`(1.3G)에 보관(수정 시 약 5분 재빌드). 지워도 카톡엔 영향 없음.
- (Claude 보충) 카카오톡 Wine이 교체되면 패치가 사라질 수 있음.

관련: [[카카오톡-Wine-스플래시-멈춤-한글-네모-크래시]], [[카카오톡-Wine-메시지-올-때-독-올라옴-차단과-내용-알림-실패]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
- [Wine MR 10060 — comdlg32: Use XDG Desktop Portal for native file dialogs](https://gitlab.winehq.org/wine/wine/-/merge_requests/10060)
