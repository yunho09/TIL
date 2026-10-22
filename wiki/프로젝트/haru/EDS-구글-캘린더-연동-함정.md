---
tags: [project, haru, eds, evolution-data-server, google-caldav, pygobject]
updated: 2026-10-23
---

# EDS·구글 CalDAV 연동 함정 (PyGObject)

[[프로젝트/haru/프로젝트-현황]]에서 Evolution Data Server를 PyGObject로 쓰며 부딪힌 함정 모음. 대부분 로컬 캘린더에서는 안 보이고 **구글 캘린더로 돌려야 드러났다**.

## 구글 CalDAV 거절
- 따로 떼어낸(detached) 회차가 남은 상태에서 RRULE을 지우면 HTTP **400** → 회차를 먼저 지우고 반복을 끈다
- 오래된 `X-EVOLUTION-CALDAV-ETAG`로 modify하면 HTTP **409** → 여러 단계를 연달아 실행(되돌리기 등)할 때 단계마다 직전에 서버의 최신 X- 속성(ETag)을 복사하고 SEQUENCE를 +1
- 지운 일정을 같은 UID로 되살리는 게 막히면 새 UID로 재생성하고, 이후 단계의 UID도 같이 바꿔야 함
- 구글 주간 규칙은 항상 `BYDAY`를 가짐 → "모든 반복"에서 DTSTART만 옮기면 안 됨

## 구글 일정이 통째로 안 보일 때
- 코드가 아니라 `evolution-source-registry` 서비스가 멈춰 구글 소스 연결이 끝나지 않으면 구글 일정이 **조용히** 사라진다
- 해결: `systemctl --user restart evolution-source-registry` 후 `evolution-calendar-factory` 재시작, 새 SourceRegistry로 재연결. 앱은 20초 안에 연결이 안 끝나면 "다시 연결" 배너를 띄우고 `backend-died` 시그널에서 자동 재연결

## PyGObject/GTK API
- `ECal.Client.connect`가 GObject의 `connect`를 가려 "takes exactly 6 arguments" TypeError → `GObject.Object.connect(client, "시그널", cb)` 사용
- `ECal.Component.set_descriptions`는 None 불가(TypeError) → 빈 메모는 `[]`를 넘겨야 함. 다른 setter(`set_location`, `set_due`, `set_rrules` 등)는 None 허용
- 반복 일정 조회는 `get_object_list_as_comps_sync` + `generate_instances_for_object_sync`; 단순 `generate_instances_sync`는 detached 인스턴스를 무시
- 재연결(`reconnect`) 중 이전 연결의 콜백이 늦게 도착하는 경쟁 → 세대 카운터로 낡은 콜백 무시
- `Adw.Toast` 제목은 기본 `use_markup=True`라 `R&D` 같은 글자가 빈 토스트가 됨 → `use_markup=False`
- `Gtk.Calendar.select_day`는 deprecated → `set_date`
- 키 단축키를 CAPTURE 단계에서 처리할 때 `Gtk.TextView`는 `Gtk.Text`/`Gtk.Entry`의 하위 클래스가 아니므로 포커스 예외에 따로 넣어야 함(안 그러면 메모 입력 중 ↑가 날짜를 바꿈)
- 월 보기 드래그 도중 `refresh()`가 일정 객체를 새로 만들어 프리뷰가 끊김 → 객체 동일성이 아니라 `(source_uid, uid, rid)`로 매칭
- 구글 저장 응답이 ~1초라 드래그 후 막대가 원래 길이로 튀던 것은 렌더링(10ms)이 아니라 저장 지연 때문 → 낙관적 오버라이드(토큰 일치 시에만 해제)

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-10-23
