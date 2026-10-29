---
tags: [linear, github, 이슈관리, 워크플로]
updated: 2026-10-29
---

# Linear 도입과 GitHub org 연동 (Linear Code 앱)

GitHub Issues 중심이던 팀에 Linear를 도입할 때의 개념 대응, 워크스페이스/팀 설계, GitHub 연동 후 사용법을 정리한다. 핵심 함정은 **GitHub org 하나는 Linear 워크스페이스 하나에만 연결**되는데, Linear가 GitHub App을 **두 개**(`Linear`, `Linear Code`) 가지고 있어 이미 예전 `Linear` 앱이 깔린 org에도 `Linear Code`로 새 워크스페이스를 붙일 수 있었다는 점이다.

## 개념 대응 (GitHub Issues → Linear)
- **Workspace**=조직, **Team**=이슈 묶음(팀 키가 이슈 ID 접두사, 예 `OPN-12`), **Label**=분류(제목의 `[FEAT]` 접두사를 대체), **Project**=기능 단위 묶음, **Cycle**=1~2주 스프린트(못 끝낸 이슈 자동 이월), 상태 Backlog→Todo→In Progress→In Review→Done.
- Free 플랜은 **팀 2개·활성 이슈 250개** 제한이 **워크스페이스마다** 따로 붙는다(멤버 수 제한은 자료마다 달라 공식 요금 페이지 확인 필요). 다른 프로젝트와 한도를 나눠 쓰기 싫으면 프로젝트별 워크스페이스가 유리하나, 워크스페이스 간 이슈 이동 기능은 없어 합칠 땐 내보내기/가져오기.
- 팀은 제품 단위로 하나만 만들고 FE/BE는 라벨로 나누는 구성을 택함(FE·BE 따로 이슈 관리가 필요할 때만 팀 둘).
- 가입은 Google/이메일 로그인이라 새 계정이 필요 없고 한 계정으로 여러 워크스페이스를 오간다. GitHub 계정은 로그인 수단이 아니라 로그인 후 따로 연결. 관리자(Admin)는 한 명 더 지정해 둔다.
- 워크스페이스 생성 시 Region(미국/EU)은 만든 뒤 바꾸기 어렵다. 한국 옵션은 없고 EU는 유럽 데이터 규정용.

## Claude(MCP) 연결의 범위
- MCP 커넥터는 로그인 시 고른 **워크스페이스 하나에만** 연결된다. 새 워크스페이스를 만든 뒤 커넥터를 다시 연결해야 선택 화면에 뜬다.
- 연결된 도구로는 이슈·프로젝트·라벨·댓글은 다루지만 **팀 이름/키 변경, 연동(GitHub) 설정은 못 한다** → 설정 화면에서 직접(또는 브라우저 자동화로) 해야 한다. 팀 키는 브랜치명·PR에 남으므로 브랜치를 만들기 전에 정한다.
- `claude mcp add --transport http linear-server https://mcp.linear.app/mcp` 후 `/mcp`에서 로그인. 관련: [[Claude-Code-MCP-서버-등록]].

## GitHub 연동에서 막혔던 것과 해결
- 증상: `team-xquare` org에 Linear 앱이 이미 설치돼 있어 새 워크스페이스는 "No organizations connected".
- 오진: "같은 org는 다시 못 붙인다"고 결론 → 틀림. 기존 설치는 `Linear`(예전 앱, 레포 전체 접근, 마지막 흔적 1년 반 전)이고, 새 워크스페이스의 **Enable**(코드 접근) 버튼은 **`Linear Code`**라는 별개 앱을 설치하는 흐름이라 충돌하지 않았다. GitHub는 *같은 앱*을 org에 한 번만 설치할 수 있다.

  | 앱 | 용도 | 비고 |
  |---|---|---|
  | `Linear` | 예전 GitHub 연동 | 어느 워크스페이스 소유인지 GitHub에서 안 보임 |
  | `Linear Code` | 코드 접근용 새 앱 | 레포를 골라 설치 가능 |
- member 권한이면 설치가 **Request**(owner 승인 대기)로 걸리고 대기 요청이 남는다. owner가 된 뒤 설치 창의 org 옆 **Cancel Request**로 취소하고 **Only select repositories**로 필요한 레포만 골라 **Install**.
- 기존 앱을 건드리지 않으려면: 설치 창 제목이 `Install Linear Code`인지 확인하고, 기존 앱의 "Update Linear's repository access" 화면이면 아무것도 누르지 않고 닫는다(승인하면 *다른 워크스페이스*에 레포 접근이 열림). 지우면 그 앱을 쓰던 쪽 연동이 끊긴다.
- GitHub 승인/본인확인(Confirm access) 창은 에이전트가 조작할 수 없어 사람이 눌러야 한다. 연결되면 개인 GitHub 계정도 같이 Connected로 나온 경우가 있었다.

## 연결 후 설정과 사용법
- 브랜치명 형식은 GitHub 연동 설정에서 `feature/identifier`로 바꾸면 `feature/opn-5`처럼 나온다(기본은 `메일아이디/opn-5-한글제목…`). 이슈 화면에서 `Ctrl+Shift+.`로 복사(Alt를 누르면 In Progress 자동 전환 회피).
- 팀 Workflow의 PR 자동화 기본값: 드래프트 PR=변화 없음, PR 열기=In Progress, 리뷰 요청=In Review, 머지=Done(대상 브랜치가 `develop`이든 `main`이든 동일). 연결 전에는 "Connect" 버튼만 보이다가 연결 후 상태 선택 칸이 생긴다.
- PR 본문 매직 워드: `Closes OPN-5`(머지 시 닫음, 쉼표로 여러 개), `Part of OPN-5`(연결만), `Ignore OPN-5`(브랜치명 자동 연결 차단). 커밋 메시지 `feat: …` 형식은 그대로.
- **공개 레포에 Linear 링크 댓글**은 기본 꺼짐 — 켜면 이슈 제목·내용이 공개 PR 댓글로 노출된다. 레포가 public이면 켜지 않는다.
- 커밋 메시지로 이슈 연결은 GitHub webhook을 따로 추가해야 해서 생략, 브랜치명/PR 본문 연결로 충분.
- 코드 접근이 켜져 있으면 Linear 안에서 PR diff를 보고 리뷰 가능(`G` 다음 `R`).
- 팀원은 초대 후 **Settings → Connected accounts에서 각자 GitHub 계정을 연결**해야 PR 작성자가 Linear 사용자에 매핑된다.
- 가져오기: 열린 GitHub 이슈만 옮기면 되고, 옮긴 뒤 GitHub Issues는 쓰지 않기로 정한다(양쪽 관리는 꼬임).
- 팀 결정 항목: 팀 키, 사이클 사용·주기, 브랜치 규칙, Triage 담당, GitHub Issues 폐지 여부, PR 템플릿에 `Closes OPN-` 한 줄.

## 일정 관리·상태 보드·팀원 초대
- **일정**: 이슈에는 **마감일(Due date)만** 있고 시작일이 없다 → 작업별 시작~종료 간트는 불가, 프로젝트 단위 **Timeline 뷰**(프로젝트 기간+마일스톤 막대)로 대신한다. 주차별 계획은 **Cycles**(1~2주 자동 반복, 미완료 이슈 자동 이월), 단계별 목표는 프로젝트 안 **마일스톤**, 진행률은 프로젝트 %·사이클 그래프. 마감일은 급한 이슈에만 쓰는 편이 관리하기 쉽다. 프로젝트 기간·마일스톤은 MCP로 넣을 수 있지만 사이클 활성화는 팀 설정(브라우저)에서 해야 한다. (Claude 보충: 위는 Linear 일반 기능 설명이며 GitHub Project와의 직접 비교는 권한 문제(404)로 못 했다.)
- **상태 흐름**: `Backlog → Todo → In Progress → In Review → Done` (+ Canceled/Duplicate). Backlog/Todo/Canceled는 수동, In Progress·In Review·Done은 PR 이벤트로 자동. 보드 보기는 **팀 → Issues → Display → Board**(칸반, 드래그 또는 이슈 선택 후 `S`로 상태 변경), 내 이슈만은 My issues. QA 같은 중간 상태는 팀 설정 Issue statuses에서 추가.
- **초대**: 사이드바 **Invite people**(또는 Settings → Members)에서 이메일(쉼표 구분)·역할 **Member**·팀 선택 후 전송. Guest는 유료 플랜 전용. 팀이 공개 팀이면 초대 때 팀을 안 골라도 들어와서 직접 참여 가능. 초대 후 Admin 한 명 더 지정.
- **워크스페이스 이름 결정**: 조직명(`xquare`)이 다른 프로젝트에 이미 쓰이면 프로젝트 전용 이름(`xquare-openapi`)으로 따로 만든다 — Free 한도를 나눠 쓰지 않는 장점.

## PR 자동 연결 조건과 GitHub 이슈 동기화 (2026-10-29)
- **PR 연결 규칙**: Linear는 브랜치명·PR 제목·PR 본문 **세 곳 중 한 곳**에서 `OPN-숫자`(대소문자 무관)만 찾는다. 인식: `feature/opn-17`, `opn-17-login`, `yunho/opn-17` / 불인식: `feature/17`(팀 키 없음), `feature/#17`(GitHub 이슈 번호), `feature/opn17`(하이픈 없음). `feature/`는 팀 규칙일 뿐. 연결은 브랜치 생성이 아니라 **PR을 열 때** 일어나고, PR을 연 뒤에 제목·본문을 고쳐도 그때 연결된다. `Closes #10`은 GitHub 이슈 닫기이지 Linear와 무관.
- 브랜치명을 못 바꾸면: PR 본문·제목에 `OPN-36`만 적기, 본문에 Linear 이슈 URL(`Closes https://linear.app/…`), Linear 이슈 `⋯` → **Add pull request…**에 PR 주소 붙여넣기. 안 붙이면 상태는 직접 옮기면 되고 자동화만 안 된다(연동의 핵심 이득이 자동화).
- 커밋 메시지로 이슈 연결은 webhook을 따로 달아야 해서 꺼져 있고, 커밋에 `OPN-10`을 써도 아무 일 없다.
- **GitHub 이슈 동기화(GitHub issue sync)**: 한 방향(GitHub→Linear)은 레포 여러 개를 한 팀에 연결 가능, **양방향은 팀 하나에 레포 하나만**. 연결된 이슈는 제목·상태·담당자·라벨·댓글이 양쪽에서 같이 바뀐다. **켠 뒤에 새로 만든 이슈에만 적용**되어 기존 GitHub 이슈는 안 넘어온다. 양방향을 레포 2개에 쓰려면 팀을 둘로 나눠야(Free 팀 2개 소진, 이슈 번호·브랜치 키 변경, GitHub 라벨 두 벌, 제목 `[FEAT]` 접두사 소실) 해서 한 방향을 택했다.
- **동기화 구성에서의 상태 전환**: `opn`이 없으면 In Progress·In Review는 수동. Done은 PR 머지 시 GitHub가 연결된 이슈를 닫고 그 상태가 동기화되어 Linear도 Done이 될 것으로 보이나 **실측 전**. 단 GitHub 자동 닫기는 PR이 **레포 기본 브랜치**로 머지될 때만이라, 기본 브랜치가 `main`인데 PR을 `develop`/feature 브랜치로 보내는 레포(백엔드)는 안 닫혀 Done도 수동이다.
- **정리 시 교훈**: 동기화를 켜면 Linear에만 만든 이슈는 GitHub에 없어 팀원이 같은 이슈를 GitHub에 다시 만들면 Linear에 중복이 생긴다. 이슈는 사람이 만들게 하고 AI는 만들지 않기로 했으며(사용자 지시), 임의로 만든 이슈 26개는 일괄 삭제(30일간 "Recently deleted"에서 복구 가능). PR 연결 확인용 CI(lint·build + Linear ID 체크)도 상태를 수동 관리하기로 하면서 의미가 없어져 PR을 머지 없이 닫았다 — 빌드·린트 CI 자체는 Linear와 무관하게 유용.
- **Free 플랜 초대는 전원 Admin**으로 들어가므로 관리자를 따로 더 지정할 필요가 없다. 초대 목록은 Settings → Administration → Members의 **Invited** 칸(수락하면 Active), 다른 이메일로 로그인하면 초대가 연결되지 않는다.
- **보드에서 칸이 사라져 보일 때**: 이슈 0개인 상태 칸은 기본 숨김(Hidden columns). Display(오른쪽 위 가운데 슬라이더 아이콘) → **Show empty columns**로 고정. Active 탭은 Todo·In Progress·In Review만, Backlog·Done까지 보려면 **All issues** 탭.

## 관련
- [[GitHub-closing-keyword-기본브랜치-PR-자동닫힘과-코드오너-룰셋]] — GitHub 쪽 닫기 키워드 동작
- [[xquare-openapi-frontend/프로젝트-현황]] — 이 도입이 적용된 프로젝트

## 출처
- Claude Code 세션 자동 캡처 (/data/project/xquare-openapi-frontend, 2026-10-28, 2026-10-29)
