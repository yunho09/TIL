---
tags: [zsh, bash, shell]
updated: 2026-09-17
---

# zsh의 단어 분리·glob 함정 (bash와 차이)

bash 스크립트를 zsh 대화형 셸에 그대로 복붙하면 자주 걸리는 두 가지 차이.

## 단어 분리(word splitting)를 자동으로 안 한다

bash는 `for f in $F`에서 `$F`가 공백으로 구분된 문자열이면 자동으로 단어 단위로 쪼개 반복한다. **zsh는 기본적으로 이 분리를 하지 않는다** — `$F` 전체가 한 단어로 취급돼 루프가 한 번만 돈다.

해결: `bash -c '...'`로 감싸 bash 의미론으로 실행하거나, zsh에서 명시적으로 `=(${(s: :)F})` 같은 분리 문법을 쓴다.

## 매치 안 되는 glob에 기본적으로 에러를 낸다

`--include=*.ts`처럼 실제로 매치되는 파일이 없는 glob 패턴을 쓰면, zsh는 `setopt nonomatch`가 설정돼 있지 않은 한 **"no matches found" 에러를 내고 명령 자체를 실행하지 않는다.** bash는 매치가 없으면 패턴 문자열 그대로를 인자로 넘긴다.

## 대응

- 다른 곳(문서, AI가 생성한 스크립트, CI 로그)에서 가져온 셸 명령을 zsh 대화형 터미널에 그대로 붙여넣을 때는 이 두 차이부터 의심한다.
- 확실히 bash 의미론이 필요하면 `bash -c '...'`로 감싸는 게 가장 빠른 우회다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
