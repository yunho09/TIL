---
tags: [gnome, mutter, xwayland, electron, claude-desktop, ld_preload, dock]
updated: 2026-11-14
---

# Claude Desktop이 포커스를 잃으면 스스로 활성화 요청 — 워크스페이스 끌려옴·독 튀어나옴

Claude Desktop(공식 deb 5.1231.2, XWayland로 실행)이 포커스를 잃고 0.02~0.9초 안에 자기 창을 활성화해 달라는 `_NET_ACTIVE_WINDOW` 요청을 스스로 보내, 다른 워크스페이스로 넘어가도 GNOME(mutter)이 그 창이 있는 워크스페이스로 끌고 온다. 요청이 밀리면 창이 "주목 요청" 상태가 되어 독이 튀어나온다. 앱 내부 동작이라 JS를 못 고쳐서 X 요청 층에서 막는 LD_PRELOAD 거름망으로 대응했다 (처음 버전은 한 번도 작동하지 않았고, 11-13에 요청 종류 판독을 고쳐 차단 동작을 로그로 확인함).

## 증상
- Claude Desktop에 입력한 뒤 옆 워크스페이스로 넘기면 다시 Claude 쪽으로 끌려오고 독이 올라온다.
- 채팅을 안 쳐도 생기고, 같은 워크스페이스에서 터미널로 포커스를 옮겨도 다시 가져간다.
- 예전 원인(제목 표시줄 오버레이 재시도 루프)과는 다름: 그 로그는 당일 0건. 11-11 업데이트 이후의 앱 내부 동작.

## 원인 (mutter 50.1 `meta_window_activate_full` 기준)
- `allow_workspace_switch = (timestamp != 0)` — 앱이 보낸 **새 타임스탬프**의 `_NET_ACTIVE_WINDOW`는 `meta_workspace_activate_with_focus`로 이어져 **워크스페이스째 끌려온다**.
- 타임스탬프가 `last_user_time`보다 오래되면 `meta_window_set_demands_attention`으로 가고, dash-to-dock의 `show-dock-urgent-notify=true`가 **독을 튀어나오게** 한다.
- **터치패드 워크스페이스 스와이프는 `last_user_time`을 갱신하지 않아** Claude의 요청이 매번 이긴다.
- 보낸 주체 확인: XRecord로 요청 송신자를 잡고 X-Resource(`XResQueryClientIds`, mask 2)로 pid를 매핑 → `claude-desktop` 본체 프로세스. Chromium X11 `X11Window::Activate`가 `_NET_ACTIVE_WINDOW`를 `[1(app), 서버시간, 0]`으로 보낸다.
- 배제한 용의자: Clawd, clawd-focus 확장(항상 "kept"), notification-right 확장(알림 클릭 후 3초 내에만 활성화), ubuntu-dock(자동 활성화 없음), Electron 포커스-온-내비게이션 래치(`mainViewFocusRuleDisabledLatched` 미설정).
- 어느 JS 함수가 보내는지는 못 찾음: 번들이 압축돼 있고 Electron fuse(`RunAsNode 0`, `EnableNodeCliInspectArguments 0`, `EmbeddedAsarIntegrity 1`, `OnlyLoadAppFromAsar 1`)로 검사·패치가 막혀 있다.

## 해결: `claude-noyank` LD_PRELOAD 거름망
- 위치: `~/.local/lib/claude-noyank/claude-noyank.{c,so}`. 빌드: `gcc -shared -fPIC -O2 -Wall -Wextra -o claude-noyank.so claude-noyank.c -lxcb -ldl -lpthread`.
- Chromium의 모든 X 요청은 libxcb `xcb_send_request`를 지나므로 이 함수만 가로챈다. SendEvent(코어 opcode 25)가 `_NET_ACTIVE_WINDOW` ClientMessage이고 **대상이 자기 연결 소유 창**(`(window & ~resource_id_mask) == resource_id_base`)이며 **현재 활성 창이 Claude가 아닐 때만** event_mask를 0으로 덮어쓴다.
  - event_mask 0인 SendEvent는 대상 창의 생성자에게 전달되는데, 루트 창이면 서버 자신이라 **아무에게도 안 간다**.
  - 다른 창에 대한 요청과 Claude가 이미 활성일 때의 요청은 그대로 통과.
- 생성자에서 `unsetenv("LD_PRELOAD")` — 자식 프로세스(렌더러, Claude Code, MCP)로 번지지 않게 한다.
- 적용: `~/.local/share/applications/com.anthropic.Claude.desktop`의 Exec 3줄(본체 `%U`, NewChat, NewCode) 앞에 `env LD_PRELOAD=…/claude-noyank.so`. 백업 `.bak-20261112`. 업스트림에 새로 생긴 MimeType(`application/vnd.anthropic.mcpb`, `…skill`)도 맞춰 넣고 `update-desktop-database` 실행.
- 모드: `CLAUDE_NOYANK=off`면 차단 없이 `would-block`만 기록. 로그 `~/.cache/claude-noyank.log`(`ready`/`allow`/`block` 줄, 512KB 넘으면 잘림).
- 시험: 수제 SendEvent로 자기 창·남의 창 요청을 보냈을 때 자기 창만 mask 0x0, 남의 창은 0x180000 그대로, 거름망 없는 대조군은 둘 다 0x180000.

## 첫 버전이 안 막던 이유와 수정 (2026-11-13)
- 증상: 낮에 넣은 거름망이 로드는 됐지만 앱에서 온 로그가 **0건**이라 계속 끌려왔다.
- 원인: 거름망이 `xcb_send_request`의 요청 구조체에서 opcode(요청 종류) 칸을 읽어 SendEvent(25)를 걸러냈는데, **Chromium은 그 칸을 항상 0으로 두고 실제 opcode를 요청 본문에만 적는다**(크롬 소스로 확인). 그래서 모든 요청이 "상관없는 요청"으로 통과했다.
- 수정: opcode를 요청 본문에서 읽도록 변경. 크롬과 같은 방식으로 요청을 보내는 시험 프로그램으로 수정 전엔 자기 창 요청이 통과, 수정 후엔 자기 창만 막히고 다른 창 대상은 통과함을 확인.
- 교훈: LD_PRELOAD 거름망은 "로드됨"만으로 검증하지 말고, 앱과 **같은 방식으로 요청을 보내는 시험 프로그램**과 로그의 `block` 기록으로 확인한다.
- 적용 조건: 이미 떠 있는 앱은 옛 거름망을 쥐고 있어 **트레이 아이콘 → Quit으로 완전 종료 후 독에서 재실행**해야 한다(창 X 버튼은 종료가 아님. 앱 내부 Claude 작업도 같이 끊김).
- 실사용 확인: 재실행 후 새 거름망이 로드됐고, 포커스를 잃은 직후 자기 창 재활성화 요청이 `block`으로 기록됨. 켜지는 순간의 같은 요청 3건도 차단됐고, Claude 창을 직접 클릭했을 때는 통과해 정상 활성화.

## 앱 업데이트와의 호환 (5.1231.2 → 5.11492.0)
- 새 버전도 창 요청을 같은 방식으로 보내 거름망을 그대로 쓸 수 있었고, 실행 파일 내용도 기존 `.desktop` 설정과 달라진 게 없어 다시 맞출 필요가 없었다. 트레이 아이콘 컬러도 업데이트 후 자동 재적용([[GNOME-앱목록-독-정리와-상단바-트레이-아이콘-색-입히기]]).
- 업데이트 뒤에도 앱을 한 번 더 완전 종료·재실행해야 새 버전이 되고 거름망 로드를 다시 확인한다. 새 버전의 끌림 여부는 미확인.

## 감수할 점
- Claude가 다른 앱 뒤에 있을 때 트레이 아이콘·Claude 알림 클릭으로 앞으로 못 나온다 (독, Alt+Tab, Clawd 세션 이동은 정상).
- 앱 내부 `app.relaunch()`로 재시작하면 거름망이 빠진다(생성자가 LD_PRELOAD를 지움). 독으로 다시 켜면 복귀.
- 되돌리기: 백업 `.desktop`을 원래 이름으로 복원 후 Desktop 재시작.

## 검증 방법 (재시작 후)
- `/proc/<새 pid>/maps`에 `claude-noyank.so`가 있는지, 로그에 `ready` 뒤 워크스페이스 전환 때 `block` 줄이 찍히는지, 끌려옴·독 튀어나옴이 사라졌는지 확인.
- 켜 둔 채 `.desktop`을 고쳤으면 독 점이 사라지는 건 정상 — [[독-실행-점-사라짐-런처-수정-직후-재실행]].

## 디버깅 팁
- XRecord 콜백의 `category`는 FromClient가 **1**이다(0으로 거르면 아무것도 안 잡힘).
- `pgrep -f '이름'`을 같은 명령줄에서 쓰면 자기 셸이 매칭돼 죽는다(exit 144) — `xrec-sende[r]`처럼 대괄호 패턴을 쓴다.
- 압축된 거대 JS에서 `.{0,160}` 컨텍스트 grep은 타임아웃 → Python 정규식으로.

## 관련
- [[Electron-워크스페이스-고정-창-포커스탈취]] — 같은 mutter 포커스 규칙(MRU, WM_HINTS)을 다룬 다른 원인
- [[스와이프-직후-Alt+Tab-이전-데스크톱-목록]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory), 2026-11-12
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-11-14
