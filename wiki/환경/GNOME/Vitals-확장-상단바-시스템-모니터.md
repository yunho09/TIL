---
tags: [gnome, extension, vitals, 시스템모니터]
updated: 2026-09-08
---

# Vitals 확장으로 상단바에 CPU/메모리 표시

GNOME 상단바에 CPU·메모리 등 시스템 자원을 실시간으로 보여주는 `Vitals@CoreCoding.com` 확장을 활성화·설정한 기록. 이미 설치는 돼 있었는데 꺼져 있었을 뿐이었고, GNOME 50에서도 호환됐다.

## 확장 자체 스키마는 `--schemadir`로 직접 지정해야 한다

확장이 자체적으로 정의한 GSettings 키(예: Vitals의 `hot-sensors`)는 시스템 스키마 디렉터리에 컴파일돼 있지 않다. 그래서 `gsettings set org.gnome.shell.extensions.vitals ...`를 그냥 실행하면 스키마를 못 찾아 실패하고, 확장 설치 경로 안의 `schemas/` 폴더를 `--schemadir`로 직접 가리켜야 한다:

```bash
gsettings --schemadir ~/.local/share/gnome-shell/extensions/Vitals@CoreCoding.com/schemas \
  set org.gnome.shell.extensions.vitals hot-sensors \
  "['_processor_usage_', '_memory_usage_', '__temperature_avg__']"
```

이 패턴은 Vitals뿐 아니라 **자체 스키마를 쓰는 GNOME Shell 확장 전반**에 적용된다 — 다른 확장을 CLI로 설정할 때도 먼저 그 확장의 `schemas/` 경로를 찾아 `--schemadir`로 넘겨야 한다.

## 자주 쓰는 키

- `hot-sensors` — 패널에 항상 표시할 센서 목록(위 예시는 CPU 사용률 + 메모리 사용률 + 평균 온도). 드롭다운 메뉴에서 항목 옆 핀 아이콘을 눌러도 같은 효과.
- `position-in-panel` — 패널 내 위치. `0`=왼쪽, `1`=가운데, `2`=오른쪽(기본값).
- 갱신 주기(기본 5초)도 설정에서 바꿀 수 있다(예: 3초).

## GNOME 50에서 D-Bus 스크린샷이 막혀 있음

패널 변경 결과를 스크린샷으로 자동 확인하려 했으나 GNOME 50의 스크린샷 D-Bus 인터페이스가 `AccessDenied`로 거부했다. 자동화 스크립트에서 화면을 캡처해 검증하려는 시도는 GNOME 50에서 이 경로가 막혀 있다는 것을 전제하고, 사용자에게 육안 확인을 요청하는 쪽으로 우회해야 한다.

## 출처

- Claude Code 세션 자동 캡처 (/home/yunho/.local/share/gnome-shell/extensions/Vitals@CoreCoding.com)
