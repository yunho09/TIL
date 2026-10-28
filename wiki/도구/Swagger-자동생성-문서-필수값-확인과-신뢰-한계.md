---
tags: [swagger, openapi, spring, api]
updated: 2026-10-19
---

# Swagger 자동생성 문서 필수값 확인과 신뢰 한계

Spring Boot가 검증 어노테이션에서 자동 생성하는 OpenAPI 문서를 읽을 때 어디까지 믿을 수 있고, 어디서부터는 백엔드에 직접 확인해야 하는지 정리한다.

## 필수 필드는 Schema 탭에서 본다

Swagger UI의 Request body는 기본으로 **Example Value** 탭이 열려 있는데, 이 탭은 예시 값만 보여줄 뿐 필수 표시가 없다. 필수 여부를 보려면:

1. Request body의 `Example Value | Schema`에서 **Schema**를 누른다.
2. 필드 이름 옆에 빨간 `*`가 있으면 필수. 접혀 있으면 펼쳐야 보인다. 맨 아래 **Schemas** 섹션에서 같은 DTO(예: `AnimalKindRequest`)를 펼쳐도 동일하게 확인된다.

JSON 문서(`/v3/api-docs/...`)를 직접 읽으면 `required` 배열과 `minLength`/`minItems`/`maxLength` 같은 제약이 그대로 나온다. 이 값은 실제 서버 검증(`@NotBlank`→`minLength:1`, `@NotEmpty`→`minItems:1`, `@Size`→`maxLength`)과 출처가 같아 믿을 만하다.

**헷갈리기 쉬운 점**: Request body 옆에 붙는 빨간 `required`는 "body 자체를 반드시 보내야 한다"는 뜻이지 필드별 필수 여부가 아니다. path·query 파라미터는 파라미터마다 별도로 `"required": true/false`가 붙는다 — 예를 들어 어떤 파라미터가 `false`라면 아예 안 보내도 된다.

## Swagger가 못 알려주는 것 — 응답 필드 null 여부

응답 DTO에는 보통 검증 어노테이션을 달지 않으므로 응답 스키마에는 `required` 배열 자체가 없다. 즉 Schema 탭에 `*`가 하나도 안 보여도 "전부 필수(항상 옴)"라는 뜻이 아니라 **알 수 없다는 뜻**이다. 어떤 필드가 null·빈 배열로 올 수 있는지는 등록 시점에 그 필드가 선택 입력이었는지로 역추론하거나, 백엔드에 직접 물어야 한다.

## Swagger 표시를 그대로 믿으면 안 되는 곳

- **`pageable`이 `required: true`로 나와도** Spring이 기본값을 채워주므로 실제로는 안 보내도 동작한다.
- **성공 상태 코드는 어노테이션에서 자동 생성되지 않아 전부 200으로만 나온다.** 실제 서버는 POST에 201을 반환해도 Swagger 문서엔 200만 적혀 있을 수 있다 — 실제 호출로 확인이 필요하다.
- 같은 이유로 `page` 파라미터의 `minimum: 0` 같은 표시도 실제 동작(예: 1부터 시작)과 다를 수 있다. 실사례(ToyVillage 업무일지 #158): 명세대로 화면 페이지−1을 보냈더니 서버는 `page=0`과 `page=1`을 둘 다 첫 페이지로 처리해 1·2페이지가 같은 데이터로 나왔다 — "1페이지와 2페이지가 똑같다"는 증상이 나오면 페이지 시작값(0 vs 1)부터 의심하고 `page=0/1/2`를 직접 호출해 비교한다. 첫 페이지만 조회하는 화면(대시보드)은 0을 보내도 문제가 안 드러나 오래 숨어 있을 수 있다. ([[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] 이슈 #158 참고)
- `POST`/`PATCH`가 같은 요청 스키마를 공유하는 경우, 수정 요청에서도 생성 때와 똑같이 전체 필드가 필수로 보인다 — 수정 시 정말 전부 필수인지는 Swagger만으로 판단하지 말고 별도 확인 대상으로 남겨야 한다.

## Basic Auth로 막힌 staging Swagger는 브라우저로 우회

staging Swagger UI에 Basic Auth가 걸려 있으면 `curl`로는 401로 막힌다. 이미 로그인해둔 브라우저(Chrome 등)가 있다면 그 세션으로 Swagger UI를 열어 문서를 확인하는 쪽이 빠르다 — 별도로 자격증명을 알아내거나 인증 우회를 시도할 필요 없이, 사람이 이미 인증해둔 상태를 그대로 재사용하는 것.

Notion 같은 별도 명세 DB에 특정 API가 아예 등록되어 있지 않을 때도 Swagger가 유일한 출처가 될 수 있다. 이 경우 Contract 등 파생 문서의 `source` 필드에 Swagger URL을 남기고, "Notion엔 없고 Swagger가 유일한 출처"라는 사실 자체를 문서에 명시해야 한다 — role·성공 상태 코드 등 이 페이지가 정리한 한계가 그대로 적용되므로, 추후 값이 이상해 보이면 이 출처부터 의심해야 한다.

## 실서버와 어긋나는 표기 실사례 9건과 대조 해석법 (2026-10-14)

ToyVillage에서 계약 65개를 스테이징 Swagger와 자동 대조해 다른 22곳을 실서버로 확인한 결과(계약 쪽 14곳 수정, 나머지 9곳은 Swagger가 틀림). 9건 모두 **성공 응답 코드**만 달랐고 경로·요청·응답 필드 차이는 없었다.
- **생성 POST 8건**: 실서버 201인데 Swagger는 200 — `POST /animal-manage/kind`·`/animal-manage/legal-status`·`/animal-manage`·`/tasks`·`/close-day`·`/open-time`·`/documents`·`/app/admin/employees`. 위 "성공 상태 코드는 전부 200" 항목의 실사례. Spring에서 성공 코드를 문서에 명시하지 않으면 기본값 200이 나오는 흔한 패턴이라 백엔드엔 `@ResponseStatus(HttpStatus.CREATED)` 또는 `@ApiResponse(responseCode = "201")` 지정을 요청하면 된다. 직원 생성만 직접 호출하지 않고 코드 주석에 의존(미재확인).
- **삭제 1건**: `DELETE /notice/{id}`는 실서버 200+message인데 Swagger는 204(본문 없음).
- **Notion 명세도 틀린다**: 휴무일 날짜 조회 경로(Notion `GET /close-day`+`date`, 실제 `GET /close-day/date?date=`), 자료 수정 `PUT /documents/{id}`(Notion·옛 계약 201, 실제 200). "Swagger가 틀리다"와 "명세(Notion)가 틀리다"를 둘 다 의심해 실서버를 직접 호출하는 게 기준.
- **"다르다"를 해석하는 법**: 대조에서 "계약에만 있음"이 뜨면 서버가 그 기능을 지원 안 한다고 읽기 쉬운데, 같은 리소스의 **다른 operation**이 그 기능을 가졌을 수 있다. 휴무일은 `GET /close-day`가 `date`를 무시하고 전체를 주는 대신 `GET /close-day/date`가 따로 있었다(그 날짜 일정만, 없는 날은 `[]`, `date` 빼면 500). 처음엔 `date`를 요청에서 빼는 수정을 만들었다가 "날짜별로 주는 걸로 아는데?"라는 질문에 호출해 보고 철회했다 — 증상(전체가 온다)이 아니라 Swagger의 경로 목록(`/close-day*`)을 먼저 훑는다. 화면이 전체를 받아 직접 거르는 임시 처리가 오래 숨어 있었던 이유도 이것.
- **응답 `sort`**: Spring `Page` 응답엔 `sort`(`empty`·`sorted`·`unsorted`)가 들어와 Swagger엔 있고 계약엔 없는 차이가 3건(개체·관찰 목록 응답, 공지 목록 쿼리 `sort`) — 화면은 안 쓰므로 계약에 추가만 하면 되는 무해한 차이.
- **용어 operation**: "메서드+경로" 한 쌍 = API 한 개. 경로가 같아도 메서드가 다르면 다른 operation이다.
- **실서버 확인 방법**: 로그인된 스테이징 어드민 탭의 콘솔에서 `fetch`로 직접 호출(테스트 데이터를 만들면 제목에 이슈 번호를 붙여 두고 삭제는 사람이 한다). 비동기 호출 결과를 `await` 없이 던지면 응답 코드가 안 찍히므로 top-level `await`로 받는다(안 그러면 같은 테스트 자료가 중복 생성됨). 스테이징 프런트는 `admin-stag.toyvillage.kr`, `admin.toyvillage.kr`은 **운영**이다.
- 이 대조를 하네스에 넣은 구조는 [[ToyVillage-Admin-FE/하네스-점검-결과와-개선-방향]] 참고.

## 2026-10-27 재대조: 9건은 그대로, `floor` 필드가 새로 생김
스테이징 Swagger(`/v3/api-docs/app`)를 다시 받아 계약 65개와 대조한 결과(23:33): 같음 54·다름 11 = **알려진 차이 9건 그대로(해소 0)** + **새 차이 3건**.
- **9건의 위치는 한 곳**: 요청·응답 필드는 다 맞고 Swagger 화면 각 API의 `Responses` 맨 위 줄(성공 코드)만 틀렸다. 같은 서버의 `POST /file`·`POST /reservation`은 Swagger에도 `201 Created`로 정확히 나오므로, 성공 코드를 명시한 메서드(`@ResponseStatus(HttpStatus.CREATED)` 추정)와 빠뜨린 8곳이 갈린 것으로 보인다(백엔드 코드를 못 본 **추정**). 공지 삭제는 문서에 "204 No Content"+응답 본문 스키마가 같이 달려 있어 문서 자체도 앞뒤가 안 맞는다.
- **springdoc 근거**: 스키마 이름이 `com.command.toyvillage_server…`이고 OpenAPI 3.1 → Java 서버에서 springdoc이 코드로 자동 생성한 문서(Spring Boot일 가능성이 매우 높음).
- **새 차이 `floor`**(종 animal-kind): `POST /animal-manage/kind`·`PATCH /animal-manage/kind/{animalKindId}` 요청에 `floor` 정수 **필수(4~5)**, `GET /animal-manage/kind` 응답에 `floor` 선택. 우리 계약·화면엔 없다 → 서버가 실제로 필수 검사를 시작하면 종 생성·수정이 **400으로 실패**할 수 있다. 의미(층?)와 반영 시점은 백엔드에 질문해야 하는 사항.
- **이번에 확인하지 못한 것**: Swagger 문서만 다시 봤고 실서버를 호출하진 않았다. `.env`에 스테이징 로그인 계정이 없어 ① 실서버가 아직 201을 주는지 ② `floor` 없이 종을 만들면 정말 400인지는 미확인(종 생성은 데이터를 만드는 요청이라 하려면 계정과 함께 진행 여부 확인 필요).
- **백엔드에 보낼 메시지 형식**: 처음엔 원인 추정·요청·질문까지 담은 긴 글을 썼으나 사용자는 "안 맞는 것만" 원했다. 표 2개(성공 상태 코드 9행 `API|기능|실제|Swagger`, 새 필드 3행)로 끝내고, 직원 계정 생성의 201은 예전 코드 주석 근거라 재확인 안 했다는 각주 한 줄만 남기는 게 맞다. 슬랙은 표를 표로 렌더링하지 않으니 마크다운 표는 노션·GitHub·디스코드용.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-27 (Swagger 재대조, `floor` 새 차이, 백엔드 전달 메시지)
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-12~14 (#246/#247 Swagger 표기 오류 9건, `/close-day/date`)
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-09-17
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop) — 2026-09-17 밤 (이슈 #106, [[프로젝트/ToyVillage-Admin-FE/프로젝트-현황]] 참고)
