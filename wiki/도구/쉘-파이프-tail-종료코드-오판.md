---
tags: [bash, shell, 종료코드, 파이프]
updated: 2026-09-12
---

`command | tail -N` 형태로 실행하면, 쉘이 관찰하는 종료 코드(`$?`)는 파이프라인 마지막 명령인 **`tail`의 것**이지 `command`의 것이 아니다. `tail`은 입력을 받기만 하면 거의 항상 0을 반환하므로, `command`가 실패해도 파이프 전체는 "성공"으로 보인다.

## 문제 상황

e2e 테스트 전체 실행 결과를 `yarn playwright test | tail -25`로 요약해서 보고, exit code가 0인 것을 근거로 "전체 통과"라고 보고했다. 실제로는 83개가 실패한 상태였는데, `tail`이 실패 목록 일부만 화면에 남기고 exit code는 자기 것(0)을 돌려줘서 생긴 오판이다.

## 대응

- 파이프라인 앞쪽 명령의 실제 종료 코드가 필요하면 `set -o pipefail`을 켜거나 `${PIPESTATUS[0]}`(bash)로 직접 확인한다.
- 결과를 요약해서 보여줘야 하는 경우에도, 먼저 원본 명령을 그대로 실행해 종료 코드와 러너가 찍는 요약 줄(`X passed / Y failed`)을 확보한 뒤 그 출력을 자르는 순서로 한다.
- 테스트 러너처럼 마지막에 요약 줄을 찍는 도구라면, 그 줄이 잘리지 않게 `tail`의 범위를 넉넉히 잡거나 grep으로 요약 줄만 뽑는 편이 더 안전하다.

## 출처
- [[ToyVillage-Admin-FE/프로젝트-현황]] / Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE), 2026-09-12
