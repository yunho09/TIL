---
tags: [리눅스, XDG, 다운로드, nautilus, 크롬]
updated: 2026-11-23
---

# Downloads 폴더 영문 통합: XDG 기본값·심볼릭 링크·최신순 정렬

한글 `다운로드`와 영문 `Downloads`가 홈에 공존하던 것을 영문 `Downloads` 하나로 합친 기록. 실제 파일은 용량 여유가 있는 `/data/downloads`에 두고 `~/Downloads`는 그 링크로 둔다.

## 구성
- `~/Downloads` → `/data/downloads` 심볼릭 링크. 예전 `~/Downloads`에 있던 파일은 `/data/downloads`로 이동. 이름이 겹친 파일은 **내용이 같은지 확인한 뒤** 사본만 삭제.
- **XDG 기본값:** `xdg-user-dir DOWNLOAD`가 `/home/yunho/Downloads`를 반환하도록 변경. 파일 앱 사이드바 북마크와 Figma 내보내기 위치도 `Downloads`로 맞춤.
- **한글 `다운로드` 제거 순서:**
  - 크롬이 저장 위치로 한글 폴더를 쓰고 있으면 먼저 지우면 안 된다. 그동안은 파일 앱에서만 숨김(터미널 `ls`에는 보임).
  - 크롬 `chrome://settings/downloads` → **위치 → 변경**에서 `Downloads`로 바꾼 뒤 한글 링크 삭제.

## 최신순 정렬
- 파일 앱에서 `Downloads` 폴더만 수정한 날짜 기준 최신 파일이 위로 오게 설정(다른 폴더는 이름순 유지).
- 이미 열려 있던 창은 이전 정렬 그대로일 수 있어 창을 닫고 다시 열어야 반영됨.
- 폴더가 위에 모이는 문제는 [[Nautilus-폴더-먼저-정렬-끄기와-Homebrew-gsettings-함정]] 참고(정렬 키가 전역이라 폴더별 적용엔 한계가 있다는 트레이드오프 포함).

## Notion 앱(별도 크롬 프로필)만 옛 `~/다운로드`로 저장되던 문제 (2026-11-23)
- **증상:** Notion 내보내기 파일이 영문 통합 이후에도 한글 `~/다운로드`에 저장됨.
- **원인:** Notion은 별도 크롬 프로필(`~/.local/share/chrome-webapps/notion`)로 돌아가 프로필마다 다운로드 경로를 따로 가진다. 그 창이 11-16부터 한 번도 꺼지지 않아, 11-18에 폴더를 영문으로 바꾼 것이 반영되지 않았다(실행 중 프로세스가 옛 경로를 쥐고 있음).
- **해결:** 창을 **완전히 닫으면** 그 프로필 `Preferences`의 저장 위치를 `~/Downloads`로 바꿀 수 있다. 실행 중에 Preferences를 고쳐도 종료 때 덮어써질 수 있어 닫은 뒤에 쓴다. 수정 전 설정은 `Preferences.bak-날짜`로 백업. 한글 폴더에 남은 파일은 `~/Downloads`로 옮기고 빈 폴더를 지운다.
- **교훈:** 위 "한글 폴더 제거 순서"의 크롬 점검은 기본 프로필만이 아니라 `chrome-webapps/*` 같은 **웹앱 프로필 전부**에도 해당. (Claude 보충: 세션에서는 사용자가 창을 닫기 전까지라 최종 반영 확인은 못 함.)

## 관련
- [[루트-공간-부족-Docker-디스크-data-이전과-Discord-로그-스팸]] — `/data` 디스크로 큰 데이터를 옮긴 배경 (Claude 보충)

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
- Claude Code 세션 자동 캡처 (/home/yunho)
