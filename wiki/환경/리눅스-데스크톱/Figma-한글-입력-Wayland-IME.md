---
tags: [figma, wayland, ime, ibus, electron, 한글]
updated: 2026-09-25
---

# Figma 한글 입력 — Wayland IME 플래그 vs X11 트레이드오프

비공식 Electron 래퍼 `figma-linux-next`(Electron 42)가 네이티브 Wayland로 떠 있을 때 한글이 안 쳐지는 문제를 다룬다. Wayland IME 플래그를 켜는 쪽과 X11로 내리는 쪽은 서로 트레이드오프가 있고, 최종적으로 Wayland+IME 플래그 조합을 택했다. **실제 한글 입력 성공 여부는 사용자 확인 전이라 미확정.**

## 원인 (Claude 추정 + 일부 확인)
- Figma가 **네이티브 Wayland**로 실행 중인데 Wayland용 입력기 플래그가 없었다.
- Wayland에서 Electron은 IME(ibus) 연동을 켜는 플래그 없이는 한글 입력기를 무시하는 것으로 판단.

## 시도한 세 가지 설정
설정 위치: `~/.local/share/applications/figma-linux-next.desktop`의 `Exec=` 줄 (`.desktop` 오버라이드라 원본 `.bak`을 남기고 수정).

- **1) Wayland + IME 플래그**: `--enable-wayland-ime --wayland-text-input-version=3` 추가.
- **2) X11 모드**: `--ozone-platform=x11`로 바꿔 ibus-x11 경로 사용. 한글 입력은 가장 안정적이라 기대했지만 **터치패드 핀치 줌이 안 됨** → 되돌림.
- **3) 최종: 다시 Wayland + IME 플래그**. Wayland에서 `text-input-v3` 통로가 입력기에 실제로 연결되는 것까지는 확인했다.

## 핵심 교훈
- **X11(XWayland)로 내리면 핀치 줌이 죽는다.** 터치패드 핀치는 Wayland 네이티브 제스처 경로에 의존 → 입력 문제를 X11로 우회하기 어렵다. 관련: [[Figma-터치패드-핀치줌-속도-패치]]
- 한글 입력이 안 될 때는 우선 Wayland IME 플래그(`--enable-wayland-ime`, `text-input-version=3`)로 해결을 시도하고, X11는 핀치 줌을 포기할 때만 폴백.
- 에이전트는 키 입력을 직접 칠 수 없어 **최종 검증은 사용자가 텍스트 상자에서 해야 한다.** 증상별 다음 조치: 영어만 나옴 / 자모 분리(ㅎㅏㄴ) / 글자 중복 / 아무것도 안 쳐짐.
- 앱을 재시작하면 열려 있던 파일 탭이 닫힌다(작업은 Figma 클라우드에 저장돼 있어 최근 파일에서 다시 연다).

## 미해결
- Wayland+IME 플래그 상태에서 한글이 실제로 조합되는지 사용자 확인 대기 중.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.local/share/applications) — 2026-09-25
