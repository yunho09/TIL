---
tags: [GNOME, Nautilus, gsettings, dconf, Homebrew]
updated: 2026-10-01
---

# Nautilus 폴더 먼저 정렬 끄기와 Homebrew gsettings 함정

파일 앱(Nautilus 50)에서 최신순 정렬 시 폴더가 항상 맨 위에 모이는 것은 "폴더를 파일보다 먼저 정렬" 설정 때문이다. 이를 끄는 과정에서 **Homebrew가 설치한 `gsettings`는 성공한 척하면서 실제로는 아무것도 저장하지 않는다**는 함정을 만났고, `dconf write`(또는 시스템 `/usr/bin/gsettings`)로 해결했다.

## 증상과 설정 키
- 최신순으로 정렬해도 폴더가 파일과 섞이지 않고 최상단에 따로 모임.
- 키: `org.gtk.gtk4.Settings.FileChooser sort-directories-first` (`true`=폴더 먼저). `false`로 두면 폴더도 수정 시각대로 파일 사이에 섞임.
- 같은 GTK4 설정을 쓰는 다른 앱의 "파일 열기/저장" 창에도 함께 적용된다.
- GUI로는 파일 앱 메뉴(☰) → 기본 설정 → "폴더를 파일보다 먼저 정렬".

## 함정: Homebrew `gsettings`는 dconf에 연결되지 않는다
- `which gsettings`가 `/home/linuxbrew/...`를 가리키면 시스템 것이 아니라 brew 버전이다. 이 버전은 GNOME 설정 저장소(dconf)에 못 붙어 **메모리 백엔드**처럼 동작한다 → `set` 해도 에러 없이 끝나고, 같은 명령으로 `get` 하면 바뀐 값이 나와서 **성공한 것처럼 보이지만** 실제 저장값은 그대로다.
- 그래서 "설정 바꿨는데 재시작해도 안 변함" → 앱 재시작 문제로 오진하기 쉽다. 검증은 반드시 **다른 경로**로 해야 한다:
  - `/usr/bin/gsettings get ...` (시스템 바이너리)
  - 또는 `dconf read /org/gtk/gtk4/settings/file-chooser/sort-directories-first`
- 해결: 저장소에 직접 쓰기
  ```
  dconf write /org/gtk/gtk4/settings/file-chooser/sort-directories-first false
  ```
  또는 `/usr/bin/gsettings set org.gtk.gtk4.Settings.FileChooser sort-directories-first false`.
- 파일 앱은 이 키 변경을 구독하고 있어 값이 실제로 바뀌면 열린 창에도 바로 반영된다. 안 되면 창을 닫고 다시 열기(`nautilus -q`).
- 되돌리기: 값을 `true`로.

## 일반 교훈
- brew를 도입한 PC에서는 PATH 앞쪽의 brew 도구가 시스템 도구를 가릴 수 있다. GNOME 설정을 만지는 명령은 `which`로 경로부터 확인한다. (Claude 보충: 스키마 디렉터리를 직접 지정해야 하는 확장 설정 사례는 [[Vitals-확장-상단바-시스템-모니터]] 참고)
- "쓴 명령으로 다시 읽어 확인"은 검증이 아니다 — 쓰기와 읽기가 같은 가짜 백엔드를 거치면 항상 통과한다.
- brew 도입 배경은 [[PC-중복-설치-정리-Homebrew-도입]].

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-10-01
