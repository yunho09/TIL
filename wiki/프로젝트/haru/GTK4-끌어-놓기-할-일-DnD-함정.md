---
tags: [haru, gtk4, dnd, pygobject]
updated: 2026-10-28
---

# haru 할 일 끌어 놓기(GTK4 DnD) 설계와 함정

하루 앱에서 사이드바 할 일 ↔ 달력 날짜(월/주 보기) 양방향으로 끌어 마감일을 옮기는 기능(커밋 `95a661e`)을 만들며 얻은 GTK4 DnD 지식. 핵심은 **끌어 오는 내용을 문자열이 아닌 전용 GType으로 실어야 한다**는 것과, 클릭·끌기 충돌·저장 경쟁 처리.

## 끌 내용은 전용 GType으로
- 할 일 UID를 `TYPE_STRING`으로 실으면 같은 앱의 `Gtk.Text`/`TextView`(메모칸, 입력칸)가 자체 `DropTarget(G_TYPE_STRING)`으로 받아 **UID 글자가 그대로 삽입**됨. 메모는 600ms 뒤 저장되고 노션 일기에까지 올라갈 수 있었음. 다른 앱의 텍스트칸도 받음
- 해결: `class TaskRef(GObject.Object)`(`__gtype_name__ = "HaruTaskRef"`, `key = (source_uid, uid)`)를 만들어 `Gdk.ContentProvider.new_for_value(GObject.Value(TaskRef.__gtype__, ref))`로 제공, 받는 쪽은 `Gtk.DropTarget.new(TaskRef.__gtype__, MOVE)`. 텍스트 위젯은 G_TYPE_STRING만 받으므로 거부됨. Wayland에서도 같은 앱 로컬 드롭에 GType이 전달됨(GtkNotebook도 같은 방식)
- 드롭 시점에 `(source_uid, uid)`로 `all_tasks`에서 **다시 조회**하고 없으면 False. 끄는 도중 `refresh()`가 돌아도 낡은 Task 객체를 쓰지 않음

## 시그널 규칙
- `DragSource`: `prepare`가 None을 돌려주면 드래그 시작 안 됨(이후 모션마다 prepare 재호출). `drag-begin`에서 `Gtk.DragIcon.get_for_drag(drag).set_child(...)`로 아이콘을 지정하면 유지됨
- `DropTarget`: `motion`이 `Gdk.DragAction(0)`을 돌려주면 거부(GTK 경고 없음). 놓을 곳 색은 CSS `:drop(active)` 또는 직접 클래스 토글(월 칸은 이전 칸 캐시로 42칸 순회 방지)
- 같은 앱 드롭은 DROP_START에서 값을 동기 로드하고 `drag-end`는 `drop_finish` 뒤라, 드롭 핸들러 안에서는 드래그 중 상태가 아직 유효
- 자기 행에서 시작한 드래그에 자기 드롭 영역이 반응(색 들어옴)하지 않도록 `drag-begin`에서 끌던 할 일을 기억해 `due == 선택한 날`이면 motion에서 거부

## 클릭과 끌기 충돌
- 칩을 누르면 완료 토글(클릭)인데 끌기도 같은 위젯에서 시작됨. GTK는 드래그가 시작되면 위젯의 컨트롤러를 리셋(`gtk_widget_reset_controllers`)해 클릭을 취소하므로 정상 끌기는 토글이 안 됨
- 함정: `DragSource`는 **누른 뒤 100ms(`MIN_TIME_TO_DND`)가 지나야** 시작. 그 전에 8px 넘게 움직였다 놓으면 "released"가 발화해 완료가 토글됨 → pressed에서 좌표 저장, released에서 8px 넘게 움직였으면 클릭으로 안 침. `stopped` 시그널은 느린 클릭의 더블클릭 타임아웃에도 발화해 쓸 수 없음
- 월 그리드의 CAPTURE GestureDrag(일정 드래그)는 할 일 칩 위 press에서 `_drag=None`으로 빠지게 하고 `_drag_end`도 `d is None`이면 조기 반환. 기존 "task" 모드를 제거해 할 일은 GTK DnD 한 경로만 사용

## 저장 경쟁 (체크 직후 끌기)
- `_save_task`의 낙관적 override에 변경 필드뿐 아니라 **진행 중 icomp 클론**과 **이전 override 필드**를 병합해 둠: 체크 후 EDS가 반영하기 전(~1초)에 바로 끌면 낡은 `NEEDS-ACTION` 컴포넌트로 덮어써 구글에서 완료가 풀리던 문제와 체크 표시가 잠깐 풀려 보이던 문제 해결

## 기타 확인된 점
- 주 보기: 머리줄과 본문 폭이 다를 수 있어(비오버레이 스크롤바) `day_at(x, y)`가 y로 머리/본문을 구분해 각자 폭으로 x→요일 매핑
- "할 일" 탭 목록은 달력이 안 보이는 화면이라 끌기 제외(`_task_row(draggable=False)` 기본값)
- 달력 → 사이드바 방향은 "지금 선택한 날"로 옮김(칩/주 보기 위 막대를 오른쪽 할 일 묶음에 놓기)

## 작업 방식 메모
- 실행 중인 앱은 옛 코드라 "안 되는데"의 원인이 될 수 있음(이번엔 토요일부터 계속 켜진 프로세스). 정상 종료는 `gdbus call --session ... org.gtk.Actions.Activate 'quit' '[]' '{}'`(zsh에서 `[]`는 따옴표 필수 → [[zsh-단어분리-안됨-glob-nomatch-함정]]) 후 `gtk-launch io.github.yunho09.Haru`
- 다른 세션의 미커밋 작업이 같은 파일에 섞였을 때: diff를 hunk 단위로 나눠 `git apply --cached`로 일부만 스테이징하고, 임시 `git worktree`에서 테스트를 돌린 뒤 커밋·푸시
- 실제 마우스 끌기는 자동 검증이 어려워 핸들러 직접 호출 하니스(EDS 저장은 monkeypatch)로 대신 확인

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-10-27
