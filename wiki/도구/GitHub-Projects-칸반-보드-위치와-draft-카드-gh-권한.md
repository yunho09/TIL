---
tags: [GitHub, Projects, 칸반, gh-cli]
updated: 2026-11-01
---

# GitHub Projects 칸반 보드 위치와 draft 카드, gh 권한

팀 공지의 "칸반(kanvan)"은 GitHub **Projects** 기능으로 만든 칸반 보드를 말한다. 이슈가 카드가 되어 `Todo → In Progress → Done` 같은 컬럼 사이를 옮겨 다닌다. 나눠 준 태스크가 레포 Issues에 안 보이면 보드 안의 draft 카드이거나 비공개 보드일 가능성이 크다.

## 보드 위치
- **조직**: 상단 탭 `Overview · Repositories · Projects` 중 Projects (`github.com/orgs/<org>/projects`)
- **레포**: 상단 탭 `Code · Issues · Pull requests · Actions · Projects` 중 Projects. 연결된 보드가 없으면 "No projects linked"로 나온다.
- 이슈 페이지에서는 오른쪽 사이드바 **Projects** 칸에서 `Status`를 바꿔 컬럼을 옮길 수 있다.
- 일반적인 사용: 시작 전 `Todo`, 작업 시작 `In Progress`, PR 머지 후 `Done`.

## 내 태스크를 못 찾을 때
- **draft 카드**: Projects 보드에서는 이슈를 만들지 않고 카드(Draft)만 바로 추가할 수 있다. 이런 카드는 레포 Issues 탭과 이슈 검색에 나오지 않는다. 보드에서만 보인다.
- **비공개 보드**: 로그아웃 상태로는 조직 Projects 탭에 옛 템플릿만 보일 수 있다. 팀원 계정으로 로그인해야 새 보드가 보이고, 로그인해도 안 보이면 보드에 초대되지 않은 것이므로 만든 사람에게 링크·권한을 요청한다.
- 보드에서 내 카드만 보기: 상단 필터에 `assignee:@me`. 이슈 탭에서도 같은 필터를 쓴다.
- 레포 이슈에 담당자로 걸려 있어도 내가 직접 만든 이슈일 수 있으니 작성자를 확인한다. 팀이 배분한 태스크와 구분된다.

## gh CLI로 보드를 읽으려면
- 기본 gh 로그인 토큰에는 프로젝트 읽기 권한이 없어 Projects를 조회할 수 없다.
- `gh auth refresh -s read:project` 로 스코프를 추가한다. 터미널에 일회용 코드가 나오고, Enter를 누르면 열리는 `github.com/login/device`에 코드를 입력해 Authorize하는 과정이 필요하다. (Claude 보충: 쓰기까지 필요하면 `project` 스코프)
- 이 명령은 Enter 입력과 브라우저 승인을 기다리며 계속 실행 상태로 남는다. 세션에서는 앱 터미널 탭에서 계속 실행 중이라고 표시됐다. (Claude 보충: 별도 터미널에서 직접 실행하는 편이 낫다. 관련: [[Claude-Code-느낌표-bash-접두사-채팅전용]])

## 출처
- Claude Code 세션 자동 캡처 (2026-11-01, raw 파일 없음)
