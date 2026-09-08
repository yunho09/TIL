---
tags:
  - orca
  - appimage
  - ubuntu
  - linux
  - fuse
  - 설치
updated: 2026-09-05
---

# Orca IDE 리눅스 설치

Orca는 여러 AI 코딩 에이전트를 git worktree 단위로 병렬 실행하는 IDE(ADE)다. 공식 배포는 **AppImage** 단일 파일인데, Ubuntu 26.04에서는 다운로드해도 **더블클릭으로 열리지 않는다**. 원인은 AppImage type 2가 요구하는 FUSE 2가 최신 우분투에 없기 때문이고, 압축을 풀어 설치하면 root 권한 없이 해결된다.

## 왜 안 열렸나

두 가지가 겹쳤다.

- **실행 권한 없음** — 브라우저로 받은 AppImage는 `+x`가 안 붙어 있다. 이것만으로도 더블클릭이 무반응이다.
- **libfuse2 부재 (진짜 원인)** — 파일 오프셋 8에 매직 바이트 `AI\x02`가 있으면 **AppImage type 2**다. type 2는 실행될 때 자기 자신을 임시로 마운트하는데 여기에 **FUSE 2.x**가 필요하다. Ubuntu 26.04에는 `fuse3`만 설치되어 있고 `libfuse2`는 없다(`libfuse2t64` 패키지가 universe에 있지만 기본 미설치).

```bash
xxd -s 8 -l 4 어떤앱.AppImage      # "AI.." (41 49 02) → type 2
dpkg -l | grep -E 'libfuse2|fuse3'  # fuse3만 있으면 type 2는 그냥 안 열림
```

해결 경로는 두 갈래다.

| 방법 | 장점 | 단점 |
|---|---|---|
| `libfuse2t64` 설치 | AppImage 원본 그대로 사용, **인앱 자동 업데이트 동작** | sudo 필요 |
| 압축 해제 후 설치 | **root 불필요**, FUSE와 무관하게 동작 | 자동 업데이트 불가, 용량 큼 |

여기서는 압축 해제 방식을 택했다.

## 압축 해제 설치 절차

`--appimage-extract`는 FUSE 없이도 동작한다. 이건 Orca뿐 아니라 **모든 type 2 AppImage에 공통으로 쓸 수 있는 우회법**이다.

```bash
chmod +x orca-linux.AppImage
./orca-linux.AppImage --appimage-extract        # → squashfs-root/ (Orca는 약 518MB)
mv squashfs-root ~/.local/opt/orca-ide
```

앱 목록에 등록하려면 아이콘과 `.desktop` 항목을 손으로 넣어줘야 한다. 추출된 트리 안에 이미 다 들어 있다.

- 아이콘: `usr/share/icons/hicolor/{16x16..512x512}/apps/orca-ide.png` → `~/.local/share/icons/hicolor/` 로 복사
- 실행 항목: `~/.local/share/applications/orca-ide.desktop` 작성, `Exec=` 는 추출 폴더의 **`AppRun`** 을 가리킨다 (`AppRun`이 `LD_LIBRARY_PATH` 등 환경을 잡아준다)
- 마무리: `update-desktop-database ~/.local/share/applications`, `gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor`

## ⚠️ `orca` 이름 충돌 — 반드시 `orca-ide`

**우분투에는 `/usr/bin/orca`(GNOME 화면낭독기, screen reader)가 이미 있다.** `~/.local/bin/orca`로 링크를 걸면 PATH 우선순위상 화면낭독기를 가려버린다(접근성 기능 고장).

Orca도 이걸 알고 있어서 **리눅스 빌드는 실행파일 이름 자체를 `orca-ide`로 쓴다.** 앱에 내장된 CLI 런처 스크립트 주석에 이유가 그대로 적혀 있다("avoids Ubuntu GNOME Orca conflict").

```bash
# CLI는 앱 안에 들어 있다. 이 경로를 링크한다.
ln -sf ~/.local/opt/orca-ide/resources/bin/orca-ide ~/.local/bin/orca-ide
which -a orca      # /usr/bin/orca 만 나와야 정상 (화면낭독기)
```

**공식 문서와 번들 스킬은 전부 `orca ...`로 쓰여 있으므로, 리눅스에서는 `orca-ide ...`로 바꿔 읽어야 한다.** (단 Orca 앱 *내부* 터미널에서는 앱이 자체 shim을 PATH에 넣어줄 수 있어 `orca`가 그대로 될 수도 있다.)

## 압축 해제 설치의 한계

- **인앱 자동 업데이트가 동작하지 않는다.** 실행 로그에 `[autoUpdater] APPIMAGE env is not defined, current application is not an AppImage`가 찍힌다. AppImage 런타임이 설정하는 `APPIMAGE` 환경변수가 없어서 업데이터가 갱신 대상 파일을 못 찾는다. 새 버전은 직접 받아 같은 절차로 덮어써야 한다.
- **Wayland 경고** — Electron 앱이라 실행 시 `'--ozone-platform=wayland' is not compatible with Vulkan` 및 vaapi 오류가 뜬다. 실행 자체는 정상이지만, 창이 튀거나 워크스페이스가 끌려가면 `.desktop`의 `Exec=`에 `--ozone-platform=x11`을 붙인다. (Claude Desktop에서 겪은 것과 같은 계열의 문제)
- `.desktop`의 `Exec`에는 `--no-sandbox`가 기본으로 들어간다(공식 AppImage 내장 desktop 파일이 그렇게 되어 있음).

## 검증

- 설치본: Orca **1.4.195**, Ubuntu 26.04 LTS, GNOME Wayland에서 실행 확인.
- 앱 내장 CLI `orca-ide --help`로 명령 목록(worktree/terminal/tab/automations/skills 등) 출력되는 것까지 확인.

## 출처

원본 파일 없음 — Claude Code 세션(`/home/yunho`, 2026-09-02~05)에서 직접 설치·검증하며 정리. 공식 문서는 <https://www.onorca.dev/docs>, 저장소는 <https://github.com/stablyai/orca>.

관련: [[위키-자동-캡처]]
