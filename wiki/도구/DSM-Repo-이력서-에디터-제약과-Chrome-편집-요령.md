---
tags: [DSM-Repo, 이력서, Claude-in-Chrome, contenteditable]
updated: 2026-11-03
---

# DSM Repo 이력서 에디터 제약과 Chrome 편집 요령

DSM Repo(repo-v2.dsmrepo.workers.dev)의 이력서 에디터는 markdown으로 저장되는 커스텀 contenteditable이고, **최대 5쪽·보기 화면 overflow 잘림·본문 이미지 높이 128px** 같은 제약이 있다. Claude in Chrome으로 로그인된 사용자 Chrome에서 편집하며 알아낸 동작과 우회법. (Claude 세션 관찰)

## 에디터·보기 화면 제약
- **쪽 상한 5쪽.** 상한 미만이면 넘친 내용이 새 쪽으로 자동 분할되지만, 5쪽에서는 토스트만 뜨고 분할되지 않는다. "+"는 프로젝트 쪽만 추가한다.
- **보기 화면은 조용히 잘린다.** `article` 높이 864px, `overflow:hidden`, padding 24px. 편집 화면에서 멀쩡해도 보기에서 마지막 줄이 잘릴 수 있어 저장 후 `scrollHeight`/요소 bottom으로 확인해야 한다.
- 미리보기가 따로 없다. 저장해야 보기 화면을 볼 수 있고, "피드백 보기" 토글이 저장본을 보여 준다. `저장`과 `제출`은 별개 버튼 — **제출은 사용자가 요청하기 전엔 누르지 않는다.**
- 실제 불릿(`ul`)은 항목마다 별도 `ul`이라 위아래 8px 마진이 겹치지 않아 줄 간격이 넓다(줄 간격 약 31.5px). 문단은 마진 0(약 15.5px). 그래서 글자 `•`로 쓴 목록은 "다닥다닥" 붙어 보인다 → 간격을 주려면 실제 불릿, 분량을 아끼려면 `•` 줄로 **쪽 여유에 따라 선택**.
- 빈 문단은 보존되어 문단 구분용으로 쓸 수 있다. 중첩 목록·`ol`·`li` 안 `<br>`은 보존되지 않는다.
- 초록 강조, 큰 숫자 카드, 2단 구성, 왼쪽 라벨 칸 같은 Figma식 레이아웃은 만들 수 없다.
- **본문 이미지는 CSS `max-height:128px`.** 가로로 긴 이미지는 해상도를 올려도 크기는 그대로(선명도만 개선) → 2x로 올려 또렷하게. 업로드는 숨은 `input[class*="_markdownImageInput"]`에 `file_upload`, 이미지는 캐럿 위치에 들어가므로 `li` 안이면 밖으로 옮겨야 한다. 교체 후 이전 업로드 파일은 저장소에 남을 수 있고 지울 방법이 없다.
- 프로젝트 기간 입력(`input[class*="_projectDateInput"]`)은 "프로젝트 기간 수정"(연필)을 눌러야 렌더된다. 안 누르고 값을 넣으면 `Illegal invocation`.
- 날짜가 `Thu Jun 04 00:00:00 GMT 2026`처럼 깨져 보이면 `2026.06 ~ 진행 중` 형식으로 다시 입력.

## 프로그램으로 편집하는 법
- 블록 클래스 예: `_markdownEditorParagraph_*`, `_markdownEditorBulletList_*`, `_markdownEditorHeading3/4_*`; 속성 `data-editor-block`, `data-markdown-block="paragraph|bulletList|heading|quote"`.
- `innerHTML`을 바꾸고 `InputEvent('input',{bubbles:true,inputType:'insertText',data:' '})` 디스패치 후 blur. **쪽을 넘겨 보면 에디터가 다시 직렬화한 실제 상태**가 보인다.
- React input은 native value setter + input/change 이벤트. 기술스택 칩은 setter 후 `input[class*="_skillInput"]`에 `keydown` Enter.
- `javascript_tool` 출력은 약 1000자에서 잘려 800자 단위 청크로 읽는다. 핸들러 소스를 읽으려다 `[BLOCKED: Cookie/query string data]`가 나오면 우회하지 않는다.
- 4쪽 상태에서 1쪽 칸에 줄을 더하면 마지막 행이 새 쪽으로 자동 분할되어 참조해 둔 요소가 낡아질 수 있다 → 작성 취소 후 다시 하거나 요소를 다시 잡는다.
- 작업 전에 원문을 파일로 백업, 작업 후 새로고침해서 모든 쪽이 잘리지 않았는지 확인.

## 프로젝트 3개 구성 결정 (2026-11-04)
- 토빌만 있던 이력서에 Commonly·JOBIS를 추가. 후보 판단: Commonly 1순위(1쪽 우수상과 연결, 이야기 많음), JOBIS 2순위(가장 오래됨, 토큰 재발급 이슈가 강함), 골목결은 대안, Bium·세미위키는 약해 제외. 프로젝트마다 별도 쪽이 필요해 5쪽 상한에서 **프로필 1 + 토빌 2 + 새 프로젝트 2**가 한계. 토빌을 줄이면 이슈 2개·회고·흐름도가 빠진다는 비용을 먼저 시뮬레이션(임시 복사본으로 높이만 재고 저장 안 함)해서 보여 준 뒤 결정.
- 최종은 토빌 이슈 4개를 모두 남기고 **줄 간격 없는 `•` 목록으로 압축**(2쪽 정보·성과, 3쪽 이슈), 4쪽 Commonly, 5쪽 JOBIS. 하네스 흐름도는 남는 높이(128px 필요, 2쪽 ~85px·3쪽 ~10px)가 모자라 제외.
- 새 프로젝트도 토빌과 같은 형식(배경 → 주요 기능 → 맡은 일 → 주요 성과 → 주요 이슈, 이슈마다 **문제·해결·결과·회고**). 자리가 모자라 결과·회고를 빼면 사용자가 형식 불일치로 바로 지적하므로, 다른 줄(중복되는 성과 줄, 두 줄 해결)을 줄여 자리를 만든다.
- **프로젝트 로고**: 구청 공식 마크가 든 레포 로고는 구청이 만든 서비스처럼 보일 수 있어 Commonly는 GitHub 조직 이미지, JOBIS는 레포 `public/logo.svg` 심볼(이력서에선 80px). 사용자가 직접 설명 못 할 이슈는 빼는 것이 원칙.

## 브라우저 선택
- 앱 내장 브라우저는 로그인이 안 돼 있었고 비밀번호는 Claude가 입력할 수 없다 → 이미 로그인된 **사용자 Chrome(Claude in Chrome)**으로 전환. 작업용 탭은 끝나면 닫되, 사용자가 "열어"라고 하면 닫지 않고 둔다.

관련: [[Claude-in-Chrome-웹한글-양식-입력과-디스코드-읽기전용-검색]], [[이력서-사실-검증-기준-팀원-몫과-커밋-수]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/.config/Claude/scratch-workspaces/68eef3b8-da94-4db5-b563-9565e29705a3/4cebf249-3d72-4a77-a55b-e742194ef06a/scratch-2026-11-02-119612) — 2026-11-02~03 세션
