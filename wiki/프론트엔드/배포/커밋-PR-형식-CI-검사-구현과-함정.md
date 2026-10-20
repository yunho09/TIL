---
tags: [CI, GitHub-Actions, git-hook, commit-msg, core.hooksPath, setup-node]
updated: 2026-10-20
---

# 커밋·PR 형식 CI 검사 구현과 함정

커밋 메시지·PR 제목 규칙을 **검사 스크립트 한 파일**에 두고, 로컬 commit-msg 훅과 GitHub Actions가 같이 부르게 만든 구현 기록(ToyVillage-Admin-FE #259 / PR #266). 설계 배경은 [[PR-CI-설계-job-분리-e2e-샤딩-훅과-AI-토큰]]의 "훅은 commit-msg만" 판단을 실제로 구현한 것이다.

## 구조
- `scripts/check-pr-format.mjs` — 판정은 여기서만. `--commit-msg <파일>`이면 커밋 1개(훅용), 환경 변수(`PR_TITLE`·`PR_BRANCH`·`COMMIT_RANGE`)면 PR 전체(CI용). 규칙이 두 곳에 있으면 판정이 갈릴 수 있어서 한 곳.
- `.githooks/commit-msg` — 스크립트를 `--commit-msg`로 부르는 3줄짜리 입구.
- `scripts/install-hooks.mjs` + `package.json`의 `postinstall`/`hooks:install` — `git config core.hooksPath .githooks` 자동 설정.
- `.github/workflows/pr-format.yml` — develop 대상 PR에서 실행.
- 실패는 `process.exit(1)` — 훅이면 git이 커밋을 안 만들고, Actions면 빨간불. 통과는 0.

## 커밋 메시지 판정 방식
- 줄 분리 후 2줄 이상이면 본문·트레일러로 보고 실패.
- 첫 줄을 `/^([a-zA-Z]+)(\([^)]*\))?(!)?:(.*)$/`로 조각냄. scope·`!`도 일단 **잡아 둬야** "형식 오류"보다 구체적으로 "scope 금지"를 알려줄 수 있다.
- type 7종 포함 여부, 콜론 뒤 공백 정확히 한 칸(`/^ \S/`), 끝이 `다` 또는 `[해돼어아세네에예]요`면 실패.
- **명사형 종결은 기계가 판단 불가** → 확실한 종결어미만 막는다. `요`는 앞 글자까지 봐서 '필요'·'중요' 오탐을 피함.
- 문제를 배열로 모아 한 번에 전부 보여 준다(첫 오류에서 멈추지 않음).
- 가운뎃점(·)은 team-rules에 금지 규칙이 없고 "메모리와 다르면 team-rules를 따른다"고 정해 두어 막지 않음.

## PR 제목 판정
- 브랜치 이름에서 **하이픈만 공백으로** 바꾼 값과 대소문자까지 같아야 통과(규칙 문구가 "그대로 쓰고 하이픈만 변환"). 브랜치 이름 형식 자체(`#번호/slug`)는 검사 안 함 — `hotfix/...` 같은 예외 브랜치가 있었기 때문.

## 훅 설치와 우회
- git은 기본적으로 `.git/hooks/`만 보고, `.git/`은 푸시되지 않아 팀에 전달 불가 → 훅은 커밋되는 `.githooks/`에 두고 `core.hooksPath`를 각자 한 번 설정해야 함. 이 설정도 `.git/config`라 커밋 불가 → `postinstall`로 자동화. git 저장소가 아니면(압축본 등) 아무 것도 안 하고 성공 종료해 설치를 깨지 않게 함.
- **함정: yarn 4는 `postinstall`을 매번 실행하지 않는다**(의존성 변경 시에만). 이미 설치된 팀원은 머지 후 `yarn hooks:install`을 한 번 직접 실행해야 한다.
- **함정: `core.hooksPath`가 `.githooks`를 가리키는데 폴더가 없으면 훅이 하나도 안 돈다**(오류도 없음). pre-commit 훅을 지울 때 설치 스크립트까지 지워 이 상태가 됐었다.
- 훅이 건너뛰는 경우: 머지 중(`git rev-parse --verify MERGE_HEAD` 성공), `fixup!`/`squash!`, 빈 메시지. 편집기로 쓴 메시지는 git이 주석을 지우기 **전에** 훅을 부르므로 `#` 줄을 직접 제거.
- `--no-verify`로 우회 가능 → CI가 필요한 이유. 반대로 강제 푸시 금지 규칙 때문에 푸시된 위반 커밋은 못 고치므로 **커밋 순간 막는 훅이 핵심**.

## CI 워크플로 결정
- `ci.yml`에 넣지 않고 **파일 분리**: 제목 수정에 재검사하려면 `edited` 타입이 필요한데, 같이 두면 PR 본문만 고쳐도 e2e가 취소·재실행됨. job `name`(`commit·PR format`)이 필수 검사 등록 때 쓰는 이름.
- `concurrency`(PR 번호별 `cancel-in-progress`), `permissions: contents: read`, `timeout-minutes: 5`.
- `fetch-depth: 0` — 기본(최신 1커밋)으로는 `base..head` 범위를 `git log`로 못 읽음. `git log --no-merges --reverse --format=%H%x00%B%x1e`로 해시/메시지/커밋 구분자를 NUL·RS로 둠(메시지에 줄바꿈이 있을 수 있어서). `--no-merges`라 머지 커밋은 git이 처음부터 제외.
- **PR 제목·브랜치는 `env`로 넘긴다.** 명령줄에 `${{ ... }}`로 직접 끼우면 제목에 쉘 명령을 심는 스크립트 인젝션이 가능.
- `yarn install` 생략 — 스크립트가 Node 내장 모듈만 사용.

## 함정: `setup-node`의 패키지 매니저 캐시
- 첫 CI가 검사 스크립트까지 가지도 못하고 **Node 설치 단계에서 실패**: `actions/setup-node`가 yarn 캐시를 기본으로 켜는데, 러너의 yarn은 1.22이고 프로젝트는 4.x를 요구해서 실패. yarn을 안 쓰는 잡이면 캐시 비활성화 설정 한 줄로 해결(옵션명은 세션에 미표기 — Claude 보충: `package-manager-cache: false`로 추정).
- 로컬은 이 단계를 안 거치므로 푸시 전엔 알 수 없었다 → 워크플로는 실제 PR에서 돌려 봐야 확인됨.

## 검증·운영 메모
- 최근 develop 커밋 767개(머지 제외)에 돌려 위반 1개(본문이 있는 커밋), 오탐 0. 임시 git 저장소를 만들어 실제 커밋·머지로 훅과 CI 모드를 검증하는 테스트 9개를 `yarn test:harness`에 포함.
- **필수 검사 등록은 PR 머지 후**: develop에 아직 없는 검사를 필수로 지정하면 열려 있는 PR이 새 커밋을 푸시하기 전까지 대기 상태로 막힌다.
- PR #266에서 e2e 1/4의 `S9`(로그인 401 테스트)가 두 번 연속 실패했으나 실패분만 재실행하니 통과 — 화면 코드를 안 건드린 PR이라 #264의 간헐 실패로 판단(근본 원인 미확정). `ci.yml`을 같이 고치는 #264와 파일 충돌 가능성도 있음.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop-4) — 2026-10-20
