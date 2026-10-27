---
tags: [omo, oh-my-openagent, herdr, omc, claude-code]
updated: 2026-10-27
---

# oh-my-openagent(omo) 설치와 OMC 비교

oh-my-openagent(omo)는 Claude Code와 별개인 독립 에이전트 앱이다. 공식 설치 스크립트로 `~/.local/bin/omo`에 깔고, 흐름도(DAG) 확장은 Herdr 창 안에서만 뜬다. Claude Code용 omo는 없고, Claude만 쓸 거면 이미 설치된 OMC와 겹치는 부분이 크다.

## 설치 (omo 5.1.47 → 5.1.48)
- **omo**: 공식 설치 스크립트(체크섬 검증 포함). `.zshrc`는 건드리지 않는다.
- **Herdr 0.9.5**: Homebrew 공식 저장소에서 `brew`로 설치. 첫 실행 때 뜨는 연동 설정 창의 **install**은 누르지 않는다(Claude Code 설정이 바뀜, 흐름도 확장에는 불필요).
- **omo-herdr-dag 1.7.0** 확장: `~/.omo/agent/extensions/` 안에만 복사된다. 먼저 dry-run으로 파일 위치를 확인할 수 있다.
- 사용 통계 끄기는 `~/.omo/omo.jsonc`에 설정. `omo doctor`에는 통계 상태가 안 나와 실제로 꺼졌는지는 확인하지 못했다.
- 로그인: omo 창에서 `/login` → Claude 구독. 기본 모델 Claude Opus 5.5. `omo doctor` 기준 작업 종류 10개 중 7개 사용 가능, deep-low·deep-high·ultrabrain 3개는 GPT 전용이라 ChatGPT 구독 로그인(`/login chatgpt-subscription`)이 필요하다.
- Claude 구독은 토큰을 빼 쓰는 게 아니라 설치된 Claude Code(`~/.local/bin/claude`)를 뒤에서 실행하는 방식이다. 다른 회사 도구가 구독을 자동으로 돌리는 셈이라 약관상 애매한 부분이 남는다.

## 업데이트
- `omo update`는 체크섬 확인 없이 바로 내려받으므로 쓰지 않고, 처음 쓴 공식 설치 스크립트에 버전을 지정해 다시 돌렸다(미러 서버의 최신이 한동안 이전 버전이었음).
- 버전을 지정해 설치하면 `~/.omo/install.json`이 그 버전에 고정되므로 '최신 따라가기'로 되돌려야 한다.
- 이미 떠 있는 omo는 옛 버전이라 Ctrl+D로 끄고 다시 실행해야 한다.

## 쓰는 법
- 평소 작업은 한국어로 그냥 입력. 어려운 작업은 `ulw`(코드 읽기→계획→검증까지, 메시지마다 붙여야 유지), 큰 작업은 `mass ulw`(여러 에이전트 병렬 + 흐름도).
- **흐름도 창은 Herdr 창 안에서 omo를 실행할 때만** 뜬다. Orca 터미널 탭에서도 `omo`는 잘 돌지만 흐름도는 없다.
- Orca의 에이전트 목록·상태 표시에는 omo가 없다(Orca의 터미널 래핑은 `omp`에만 적용). Pi 항목 실행 명령을 `omo`로 바꾸는 우회는 Pi 전용 설정 폴더 때문에 로그인이 꼬일 수 있어 비권장. [[Orca-IDE-리눅스-설치]] 참고.
- 같은 Claude 구독 한도를 공유하므로 `mass ulw`는 한도를 빨리 소모할 수 있다.

## omo vs `claude` + OMC
- omo는 `omo`로 켜야만 동작한다. `claude`에서는 omo 기능이 전혀 없다. omo 형태는 독립 앱·OpenCode 플러그인·Codex 플러그인(Light)이고 **Claude Code용은 없다**.
- omo만의 장점: 작업 종류별 Claude/GPT 자동 분배와 모델별 프롬프트, `mass ulw` 흐름도, Kibitzer(작은 모델이 과거 실수 반복을 미리 경고, 기억은 git의 마크다운), 독립 도구 호출 일괄 실행·잘못된 도구 인자 사전 수정·세션 이어하기.
- 겹치는 부분: `ulw`는 OMC의 ultrawork·ralph와 같은 아이디어. Claude만 쓸 거면 차이가 작다. `claude`에서 `ulw`는 OMC ultrawork가 켜진다.
- OMC로 모델 섞기는 `ccg`(Claude+Codex+Gemini에 각각 묻고 Claude가 종합)로 어느 정도 가능하나, 작업 종류별 자동 분배는 아니다.
- 결정: 평소엔 `claude`(OMC, 세션 자동 위키 기록 등 기존 구성은 `claude` 기준), omo는 큰 작업 비교용으로 시험. 먼저 OMC로 해보기로 했다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho)
