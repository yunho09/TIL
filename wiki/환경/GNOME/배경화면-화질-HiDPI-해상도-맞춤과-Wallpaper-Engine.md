---
tags: [GNOME, 배경화면, HiDPI, Wayland, Wallpaper-Engine]
updated: 2026-11-11
---

# 배경화면 화질 — HiDPI 해상도 맞춤과 Wallpaper Engine

배경화면이 흐릿하면 원본이 작아서가 아니라 **화면이 매 프레임 실시간으로 축소**해서일 수 있다. 실제 패널 해상도(2880×1800, 배율 150%)에 맞춰 미리 Lanczos로 줄인 이미지를 쓰면 이 뭉개짐을 없앨 수 있다. 영상형 배경(Wallpaper Engine)은 GNOME에서 Hanabi 확장으로 가능하다.

## 해상도 착시: Xwayland가 보고하는 값
- Xwayland는 논리 해상도 **3840×2400**을 보고하지만 실제 패널은 **2880×1800 + 배율 150%**.
- 3840×2160 원본은 "늘어나서 흐려지는" 게 아니라 그릴 때마다 약 0.83배로 **축소**되고 있었다. 실시간 축소는 품질이 낮아 약간 뭉개질 수 있다. (처음에 "늘어나서 흐리다"고 한 설명은 오진이었음)
- 해상도 진단 시 Xwayland 보고값이 아니라 실제 패널 값을 확인할 것.

## 해결: 패널 해상도에 맞춰 미리 줄이기
- 원본을 Lanczos 방식으로 2880×1800에 맞춰 축소(잘리는 부분은 기존 '확대' 설정과 동일)해 `~/.local/share/backgrounds/`에 저장.
  - 라이트: `warty-final-ubuntu-2880x1800.png`, 다크: `ubuntu-wallpaper-d-2880x1800.png`
- `gsettings set org.gnome.desktop.background picture-uri` / `picture-uri-dark`에 `file://` 경로로 지정. 시스템 원본(`/usr/share/backgrounds/`)은 건드리지 않는다.
- 되돌리기: 두 키를 원본 경로(`warty-final-ubuntu.png`, `ubuntu-wallpaper-d.png`)로 다시 set.
- 실제로 선명해졌는지는 사용자 눈 확인 전(미확정).

## 한계
- 원본보다 디테일이 늘지는 않는다. 개선되는 건 축소 뭉개짐뿐이라 차이가 작을 수 있음.
- 다크 모드 원본은 **256색 PNG**라 그라데이션 계단(밴딩)은 이 방법으로 해결 불가. 그래도 흐리면 해상도가 충분히 큰 다른 배경으로 교체하는 게 확실.

## 영상 배경(Wallpaper Engine) on GNOME
- **영상형**: Steam에서 Wallpaper Engine 구매 → 워크숍 배경 다운로드 → **Hanabi** GNOME 확장으로 재생. GNOME 그대로 사용 가능.
- **장면형·웹형**: GNOME에선 어려워 KDE Plasma 등으로 데스크톱 환경을 바꿔야 한다. (조사 기반, 직접 검증은 안 함)
- 이 PC에는 Steam이 설치돼 있지 않았고 설치는 보류함.

## 출처
- [[raw/Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)]] (2026-11-11 세션)
- 외부: linux-wallpaperengine README, jeffshee/gnome-ext-hanabi, achu94/gnome-wallpaper-engine

## 관련
- [[GNOME-오버뷰-창-미리보기-사라짐]]
