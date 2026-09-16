---
tags: [swagger, openapi, spring, api]
updated: 2026-09-17
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
- 같은 이유로 `page` 파라미터의 `minimum: 0` 같은 표시도 실제 동작(예: 1부터 시작)과 다를 수 있다.
- `POST`/`PATCH`가 같은 요청 스키마를 공유하는 경우, 수정 요청에서도 생성 때와 똑같이 전체 필드가 필수로 보인다 — 수정 시 정말 전부 필수인지는 Swagger만으로 판단하지 말고 별도 확인 대상으로 남겨야 한다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-09-17
