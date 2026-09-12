---
tags: [리눅스, 패키지관리, homebrew, linuxbrew, tmux, 중복설치]
updated: 2026-09-12
---

이 PC에 Homebrew(Linuxbrew)가 새로 설치되면서 apt·수동 설치와 겹치는 패키지들을 정리한 기록. tmux/mise는 brew로 통일했고, 로그인 셸(zsh)·essential 패키지(bash)처럼 건드리면 위험한 중복은 의도적으로 보류했다. 그 외에 앱 교체로 죽어 있던 설치본(wezterm, postman/notion snap, .nvm)도 함께 정리했다.

## Homebrew 도입과 정리 방침

- brew는 2026-09-12 설치. 기존 apt/수동 설치와 완전히 같은 패키지가 겹치면 brew 쪽을 남기고 이전 설치를 지우는 방향.
- 단, **로그인 셸로 지정돼 있거나 essential 패키지처럼 시스템이 의존하는 것**은 지우기 전에 위험을 확인해야 한다.

## 정리한 사례

| 대상 | 지운 쪽 | 남은 쪽 |
|---|---|---|
| tmux | apt `tmux 3.6a` (+ 고아 의존성 `libevent-core`) | brew `tmux 3.7c` |
| mise | `~/.local/bin/mise` 수동 설치 (130MB) | brew `mise 2026.9.5` — 완전히 같은 버전이었음 |
| `.zshrc` | `brew shellenv` 중복 로드 줄 | 1줄로 정리 |

### tmux client/server 버전 호환 확인

brew tmux(3.7c)로 교체해도 **이미 떠 있던 apt tmux(3.6) 서버의 세션은 그대로 유지된다** — 클라이언트가 새 버전이어도 기존 서버 세션에 재접속만 하면 되므로, 세션 끊김 없이 클라이언트 쪽만 바꿀 수 있었다. 실제로 서버 프로세스는 그대로 둔 채 클라이언트 바이너리만 brew로 바뀐 뒤에도 기존 세션 3개(1개 attached)가 안 끊기고 유지됨을 확인.

## 의도적으로 보류한 중복

- **zsh** — apt 5.9 vs brew 5.9.2. apt zsh가 `/etc/passwd`에 등록된 로그인 셸이라, brew zsh로 바꾸려면 `/etc/shells` 등록 + `chsh`까지 필요해 로그인이 걸린 작업이므로 보류.
- **bash** — 시스템 bash는 essential 패키지라 제거 불가. brew bash(5.3.15)는 새 버전을 쓰려고 일부러 깐 것일 수 있어 그대로 둠.
- brew tmux/zsh의 의존성인 ncurses/openssl/readline/pcre2/zlib/libevent/bzip2도 `clear`/`tput`/`openssl` 같은 명령을 brew 쪽으로 가리지만, brew 구조상 정상 동작이라 건드리지 않음 (지우면 brew tmux가 깨짐).

## Ghostty 터미널: deb/snap 중복

deb(1.3.0)와 snap(1.3.1)이 동시에 깔려 있던 걸 확인 → snap을 `--purge`로 제거, deb만 유지.

- **Homebrew의 ghostty는 macOS 전용 cask**라 Linux에서는 대체가 안 된다. deb를 지우면 ghostty 자체가 사라진다.

## 발견된 죽은 중복 (~450MB, 실제 실행 경로가 이미 다른 곳으로 교체됨)

| 대상 | 판단 근거 |
|---|---|
| wezterm 2벌 (174MB) | `~/bin/wezterm`(AppImage 원본)이 **libfuse2가 없어 실행 자체가 안 됨** — [[Orca-IDE-리눅스-설치]]에서 확인한 것과 같은 원인. `~/.config/wezterm`도 없어 설정한 적 자체가 없음. 버전도 2년 전 |
| postman snap (183MB) | 실행 항목이 07-28에 `~/.local/opt/Postman` 단독 설치로 교체됨. snap 쪽 데이터가 06-16 이후 갱신 없음 |
| notion-desktop snap (93MB) | 실행 항목이 09-01에 Chrome 웹앱(`--user-data-dir=.../chrome-webapps/notion`)으로 교체됨. snap 데이터 06-14 이후 갱신 없음 |
| `~/.nvm` (3.4MB) | `versions/` 디렉터리가 없어 node가 하나도 안 깔려 있음. `.bashrc`에서만 로드되는데 로그인 셸은 zsh라 애초에 읽히지도 않음. 실제 node는 apt `nodejs` |

**판단 기준**: 앱 아이콘/실행 항목이 최근에 다른 설치 경로로 바뀌었는지, 그리고 옛 설치의 데이터 디렉터리가 그 이후로 갱신됐는지로 "죽은 중복"인지 확인한다. AppImage는 libfuse2가 없으면 실행조차 안 되므로, 설정 폴더 존재 여부가 실사용 이력의 좋은 지표가 된다.

## `.desktop` 오버라이드는 중복이 아니라 의도된 구조

`~/.local/share/applications`에 snap 앱과 같은 파일명의 `.desktop`이 있으면 XDG 우선순위로 그쪽이 이겨서 snap 버전(`/var/lib/snapd/desktop/applications`)을 가린다 — 아이콘이 중복으로 보여도 실제로는 하나만 뜬다. 이 PC에서는 code(ibus), discord(wayland), gitkraken(swiftshader), obsidian(TIL 볼트 래퍼), Claude(x11), Ptyxis가 이 패턴.

## 미해결 / 사용자 결정 대기 (2026-09-12 시점)

- **docker-ce(시스템, systemd active+enabled) vs Docker Desktop 4.80** 이중 설치. 현재 활성 컨텍스트는 `desktop-linux`, 시스템 엔진 쪽은 2개월 전 `hello-world` 컨테이너 2개뿐. Docker Desktop VM 데이터는 4.7GB — 이 VM이 과거 OOM과 절전 복귀 지연을 유발한 이력이 있어([[절전-복귀-지연-원인과-zram-도입]] 참고) 한쪽만 남기는 게 이득일 수 있으나, 어느 쪽을 실제로 쓰는지에 달려 있어 보류.
- **codex CLI 설치 깨짐** (`.codex-eHKM1rRF`): npm 임시 심링크만 남고 `codex` 명령 자체가 없음. 실행 시 `Missing optional dependency @openai/codex-linux-x64` 에러. `npm i -g @openai/codex@latest`로 재설치하거나, 안 쓰면 통째로 제거해야 함.

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-12.
