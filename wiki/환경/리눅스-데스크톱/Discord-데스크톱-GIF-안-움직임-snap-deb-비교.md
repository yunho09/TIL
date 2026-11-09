---
tags: [discord, snap, deb, wayland, gif]
updated: 2026-11-10
---

# Discord 데스크톱 앱 GIF 안 움직임 — 점검 순서와 snap vs .deb

Discord 데스크톱 앱(Wayland, Intel Lunar Lake 노트북)에서 GIF가 멈춰 있고 웹 디스코드에서는 잘 움직이던 문제. 설정·GPU 끄기·snap→공식 .deb 교체까지 해도 **원인을 못 찾아 미해결**로 끝났다. 점검 순서와 snap/.deb 차이를 남긴다.

## 점검 순서와 결과
- 시스템 쪽 배제: GNOME 애니메이션 설정 켜짐, 시스템 "동작 줄이기" 요청 없음, 로컬 설정 파일에 GIF 차단 항목 없음.
- 앱 설정(계정에 저장): 사용자 설정 → **접근성 → 동작 줄이기**("컴퓨터와 동기화"·"동작 줄이기 활성화") 끄기, "Discord가 포커스됐을 때 GIF 자동 재생"·애니메이션 이모지 켜기(버전에 따라 채팅 메뉴), Ctrl+R 새로고침. 창에 포커스가 없으면 GIF는 원래 멈춘다.
- `snap run discord --disable-gpu`로 GPU 원인 분리 → 같은 증상이라 GPU 아님.
- **웹 디스코드에서 정상**이면 계정 설정 문제가 아니라 데스크톱 앱 쪽 문제로 좁혀진다 (설정이 계정에 저장돼 웹에도 동일 적용되므로).
- snap → 공식 `.deb`(1.0.168) 교체 후에도 동일. 새 설치본의 기기별 "동작 줄이기"도 꺼져 있었음.
- 마우스 올려 보기도 효과 없음. 가설(Wayland에서 창 활성 상태를 못 알아채 GIF를 멈춘다)은 **미검증**이고 `discord --ozone-platform=x11` 시험은 제안만 하고 결과를 못 봤다.
- 결론: 데스크톱은 계속 멈춤, 웹 디스코드로 확인. 이후 앱 업데이트로 고쳐질 수 있음.

## snap vs .deb
| | snap | .deb(공식) |
|---|---|---|
| 배포 | 커뮤니티(snapcrafters) 재포장 | 디스코드 직접 |
| 실행 | 샌드박스, 자체 라이브러리 | 시스템 라이브러리 |
| 업데이트 | 백그라운드 자동 | 수동이라고 설명했으나, 실제 .deb는 작은 설치 도구만 담고 첫 실행 때 앱 본체를 내려받으며 이후 자체 업데이트하는 방식으로 보임(미확인) |
| 호환성 | 영상·화면공유·한글 입력·파일 접근이 가끔 깨짐 | 문제 적음 |
| 보안 | 격리로 조금 안전 | 일반 프로그램 수준 |
- flatpak(Flathub)판도 대안이지만 flatpak 설정이 따로 필요.

## 교체 절차의 함정
- `!` 셸에서 `sudo`는 터미널이 없어 `sudo: A terminal is required to authenticate`로 실패. 비밀번호 창이 뜨는 **`pkexec`**로 실행해 snap 삭제 + `apt install ~/Downloads/discord.deb` 성공.
- 설치 중 실행 중이던 snap판은 꺼진다. 계정 데이터는 유지되고 재로그인만 필요.

## 관련
- [[Discord-영상-로딩-OpenH264-화면공유-뭉개짐]] — 같은 snap판의 영상 문제와 공식 .deb 전환 후보
- [[루트-공간-부족-Docker-디스크-data-이전과-Discord-로그-스팸]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE, 2026-11-10)
