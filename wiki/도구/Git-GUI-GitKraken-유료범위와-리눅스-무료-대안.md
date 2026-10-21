---
tags: [git, gitkraken, gui, tauri, electron, 리눅스]
updated: 2026-10-21
---

# Git GUI — GitKraken 유료 범위와 리눅스 무료 대안

GitKraken 무료 플랜은 공개·로컬 레포만 되고 프라이빗 레포는 유료(Pro 이상)다. 프라이빗 접근 자체는 특별한 기술이 아니라 일반 git 인증이라서, 무료 대안이나 직접 만든 앱으로도 된다. Tauri vs Electron 선택 기준도 함께 정리한다. (대부분 Claude의 웹 검색·기존 지식 기반이며 가격·최신 상태는 미확인 — 공식 페이지 확인 필요)

## GitKraken 요금
- **무료**: 공개 레포 + 로컬 레포만. GitHub/GitLab의 **프라이빗 레포는 Pro 이상**.
- Pro는 연간 결제 기준 사용자당 월 약 4.95달러로 보도되나 출처마다 다름. 카드 없이 7일 체험 가능.
- GitHub Student Developer Pack에 GitKraken Pro가 포함돼 있었으나 현재도 그런지는 미확인.
- 유료 잠금은 기술 한계가 아니라 사업 모델. Electron 앱을 뜯어 결제 확인을 우회하는 건 크랙(약관 위반, 기술적 보호조치 무력화 소지)이라 비추천.

## 리눅스(Ubuntu)에서 쓸 만한 무료 대안
- **Gitnuro** — GitKraken과 생김새가 가장 비슷, 커밋 그래프, GPL 오픈소스, 리눅스 지원 *(최신 상태 미확인)*
- **Sublime Merge** — 그래프·속도 우수, 결제 없이 계속 사용 가능(가끔 구매 권유 창)
- **GitButler** — Tauri/Rust/Svelte 제작, 활발히 개발 중이나 여러 브랜치 동시 작업 방식이라 사용감이 다름
- **gitg** — GNOME 기본, 그래프 있으나 기능 단순
- **WebStorm** — 비상업용 무료, 내장 git 그래프·충돌 해결이 GitKraken급
- **GitHub Desktop은 공식 리눅스 버전 없음**(커뮤니티 포크만). Sourcetree·Fork도 Windows/Mac 전용.

## 직접 만들 때
- 프라이빗 레포는 `git` CLI나 `simple-git`/`isomorphic-git`/libgit2 인증만으로 자연스럽게 됨.
- "기능 전부"는 범위 과다. 순서: 레포 열기+커밋 그래프 → 변경 보기·stage·commit → push/pull/fetch → 브랜치·머지·충돌 해결.
- 비슷한 앱이 이미 많으므로 "GitKraken에서 불편했던 X를 해결"처럼 차별점을 하나 정해야 포트폴리오로 설득력 있음. 먼저 Gitnuro 등을 써보고 불편함을 찾는 방법.
- 기술 포인트: 커밋 그래프 렌더링(브랜치 레인 배치, 수천 커밋 가상 스크롤), diff 뷰어, 드래그 rebase/merge.

## Tauri vs Electron
- **Electron**: Chromium 내장, Node.js 백엔드, 앱 100MB+, 메모리 많이 씀. JS만으로 완결(`simple-git`)이라 시작이 빠름.
- **Tauri**: OS 웹뷰 사용(리눅스는 WebKitGTK), Rust 백엔드, 앱 몇 MB, 메모리 적음. 2.0부터 모바일 지원. Rust `git2`(libgit2) 사용 가능.
- Tauri 단점: 시스템 작업을 Rust로 짜야 해 진입장벽, OS별 웹뷰 차이(특히 리눅스 WebKitGTK는 Chromium보다 렌더링 성능·호환성 문제가 잦음).
- 빨리 결과를 보려면 Electron, 가볍게/Rust 학습 겸이면 Tauri. Electron 메모리 특성은 [[리눅스-메모리-점유-앱별-진단-PSS]] 참고.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-10-21
