---
tags: [Bium, Gemini, Spring, API설계, 레시피추천]
updated: 2026-10-27
---

# 레시피 추천 API: Gemini 호출을 FE에서 서버로 옮긴 설계 (bium-be #3·#8·#10)

v1에서는 프론트가 Gemini를 직접 불러 API 키가 브라우저 번들에 노출됐다. 이를 서버로 옮기면서 v1의 프롬프트·파싱·필터·다양성 선택 로직을 그대로 이식했고, 이후 냉장고 DB 연동·저장·상세 조회·재료 수정/삭제까지 단계별로 붙였다. 결정과 근거, 최종 API 모양, 리뷰에서 나온 함정을 정리한다.

## 단계와 결정 (PR 흐름)
- **#3 (PR #7)**: Gemini 클라이언트 + v1 추천 로직 이식 + 호출 제한. 당시 냉장고 엔티티(#4)가 없어 **재료를 요청 본문으로 받는 임시 API**였다. 팀 공통 오류 형식(`{success,data,error}`)이 먼저 머지된 걸 확인하고 **main 기준으로 rebase해 팀 `ErrorCode`/`GlobalExceptionHandler`를 쓰도록 맞춤**(같은 이름 클래스 두 벌·API마다 다른 오류 형식을 막기 위해). 레시피 전용 핸들러는 지움.
- **#8 (PR #13)**: 요청 본문 재료 → **로그인한 사용자의 냉장고(`fridge_items`)에서 읽기**로 전환, 추천 결과를 `recipes`/`recipe_ingredients`에 저장, `GET /api/recipes/{id}` 추가. 로그인 없이 열어 둔 추천 경로를 제거.
- **#10 (PR #14)**: 재료 수정·삭제. #8과 별개 기능이라 이슈를 분리하고 #8 머지 후 새 브랜치에서 진행 → [[낙관적-락-version-스테일-수정-방지와-PUT-전환]].
- 이슈를 잘게 쪼개 만들었다가 사용자 요청으로 하나(#8)로 묶고 나머지는 "not planned"로 닫음.

## 초기 설계에서 정한 것 (근거)
- **재료 소스**: 클라이언트가 재료를 보내는 방식(A)보다 서버 냉장고 기준(B)이 맞다는 사용자 판단. 로그인·냉장고가 없던 시기엔 "공용 냉장고 하나" 또는 요청 본문 임시 형태로 먼저 만들 수 있고, 주소·응답 형식은 유지된다는 설명 → 결국 #4 머지를 기다리기로 함(FE가 두 번 작업하지 않게, 임시 공개 API는 Gemini 한도 남용 위험).
- **레시피 id·상세 조회**: Gemini 결과는 원래 id가 없고 v1은 화면 상태로 넘겼다. 새 FE 주소가 `/recipes/:recipeId`라 새로고침·링크 진입을 지원하려면 **DB 저장 + 상세 API**가 필요(메모리 캐시 30분·저장 안 함 대안 기각).
- **응답을 숫자 중심으로**: 시간·인분·칼로리·영양은 숫자(단위는 FE가 붙임), `difficulty`는 `EASY/NORMAL/HARD`, 재료 출처는 `source`(`FRIDGE`/`BASIC` 기본 양념). 사용 재료는 D-day가 아니라 **`expiryDate`**(저장된 레시피를 나중에 열면 D-day가 바뀌므로).
- **옮기지 않은 것**: 카드 배경 그라데이션(`RECIPE_COVERS`)은 화면 표현이라 FE에 둠. API 키는 `VITE_GEMINI_API_KEY` → 서버 `GEMINI_API_KEY`.

## 최종 API
| API | 설명 |
|---|---|
| `POST /api/fridges/{fridgeId}/recipes/recommendations` | 본문 `{prioritizeUrgent?}`(생략 가능). 냉장고 재료로 최대 4개 추천·저장 |
| `GET /api/recipes/{recipeId}` | 저장된 레시피 상세 |
| `PUT /api/fridge-items/{itemId}` | 재료 수정(`name`,`emoji`,`expiresAt`,`version`) |
| `DELETE /api/fridge-items/{itemId}?version=` | 소프트 삭제(204) |

- 추천 응답 `data.recipes[]`: `id, name, emoji, summary, cookTimeMinutes, difficulty, servings, caloriesKcal(null 가능), ingredients[{name,amount,source,required}], steps, tips, nutrition{proteinG,carbohydrateG,fatG}, fridgeMatches(중복 제거·최대 6), createdAt`. Gemini 호출 때문에 **5~15초**(실측 약 9초) 걸려 FE 타임아웃은 30초 이상 필요.
- 오류(`{success:false,error:{code,message}}`): 401 `UNAUTHORIZED` / 404 `FRIDGE_NOT_FOUND`·`RECIPE_NOT_FOUND`·`FRIDGE_ITEM_NOT_FOUND` / 406 `NOT_ACCEPTABLE` / 409 `CONCURRENT_MODIFICATION` / 422 `NO_USABLE_INGREDIENT` / 429 `TOO_MANY_REQUESTS`(+`Retry-After`) / 502 `AI_REQUEST_FAILED`·`RECIPE_NO_RESULT` / 503 `AI_QUOTA_EXCEEDED`.
- 쓸 수 있는 재료가 없으면 **Gemini를 부르지 않고 422**, 그리고 호출 제한 횟수도 쓰기 전에 거절(처음엔 제한부터 검사해 빈 냉장고 요청이 횟수를 소모했다 → 리뷰에서 지적돼 순서 변경).

## 추천 파이프라인 (v1 이식)
1. 유통기한 지난 재료 제외, 임박순 정렬(유통기한 없는 재료는 사용 가능으로 취급, 맨 뒤).
2. `RecipePromptBuilder`: 재료 최대 **30개**만 프롬프트에 포함, `- 이름 (이모지) / D-n`.
3. `GeminiClient`: 모델 `gemini-2.5-flash-lite`, temperature 0.55, JSON 응답 모드, 키는 URL 쿼리가 아니라 **`x-goog-api-key` 헤더**(URL은 로그에 남을 수 있음).
4. `RecipeCandidateParser`: 코드블록 제거 후 `{…}` 추출, AI가 준 숫자는 범위 검사(시간 1–600, 인분 1–20, kcal 0–5000 등, 벗어나면 null)·`"1,200kcal"` 파싱·`"required":"true"` 문자열 인정.
5. `RecipeCandidateSelector`: 냉장고에 없는 필수 재료가 든 레시피·디저트/음료 제외 → 비슷한 레시피를 빼고 4개 선택.
- **재료명 매칭 버그 수정**(CodeRabbit이 지적, v1에도 있던 문제): 부분 일치로 비교하면 냉장고의 "파"가 "양파"를, "김"이 "김치"를, "배"가 "배추"를 가진 것으로 판정돼 없는 재료가 필요한 레시피가 안 걸러졌다. → **한 글자 이름은 완전 일치만**, 두 글자 이상만 부분 일치 허용.

## 접근 제어·운영 결정
- **냉장고 멤버 확인**: 냉장고는 여러 명이 공유(`fridge_members`)하므로 주소의 `fridgeId`만 바꿔 남의 냉장고를 보지 못하게 멤버십을 확인. 멤버가 아니면 **403이 아니라 404**(존재 여부 자체를 숨김). 레시피 상세도 같은 방식.
- **호출 제한**: IP 기준(5/분, 서버 전체 15/분) → 로그인 후 **사용자 기준**(5/분)으로 전환. 입력 검증에 실패한 요청은 횟수에 넣지 않음 → [[LLM-프롬프트-입력-검증과-공개-API-호출-제한]].
- **Gemini 대기 중 DB 연결 안 잡기**: [[Spring-Boot-4-JPA-실전-함정-메모]].
- 엔티티(#4 담당자 소유)에 칸 추가: `FridgeItem.emoji`, `Recipe`(이모지·팁·난이도·칼로리·영양, `steps` JSON), `RecipeIngredient`(`amount` 문자열·`source`). AI가 "1/2모"·"1큰술"처럼 문자열로 주는 분량은 팀 설계의 `quantity`/`unit` 숫자 구조에 안 맞아 문자열 칸을 추가. PR 본문 "확인 부탁드립니다"에 명시.

## 협업·범위에서 배운 것
- 처음엔 냉장고 API·엔티티까지 만들었다가 **남의 담당 범위(냉장고·도감, #4 DB 설계)를 침범**한 걸 사용자가 지적 → 냉장고 API·엔티티·공통 예외 핸들러·SecurityConfig 변경을 되돌리고 추천 로직만 남김. 팀 DB 설계(`fridge_items`, `recipes` 구조)와 어긋나 있어 FE에 보낼 명세도 다시 맞춰야 했다. **착수 전에 담당 범위와 팀 설계(이슈·DBML)를 먼저 확인**할 것.
- 공통 파일(`SecurityConfig`, `ErrorCode`, `GlobalExceptionHandler`)을 고치는 변경은 PR 본문에 팀 공지 항목으로 적는다.
- 팀 규칙: PR 대상은 `develop`(로그인이 main에만 머지됐다가 develop으로 옮겨진 일 있음), 커밋·PR에 Claude 표기 금지, CodeRabbit 지적은 수정 커밋 해시를 답글로 달고 스레드 Resolved. 머지는 머지 커밋 방식.
- 커밋을 여러 개로 나눌 때 **각 커밋이 단독으로 빌드되는지** 확인(중간 커밋이 옮긴 enum import 누락으로 깨진 적 있음).
- 후속: 팀원의 냉장고 목록·재료 목록 API(PR #18)가 나오면 FE에서 `fridgeId`·임박 재료 패널·수정 모달을 연결. 재료 목록 응답의 `version`을 수정·삭제 API에 그대로 보냄. 칸 종류가 `CENTER/LEFT_DOOR/RIGHT_DOOR` 3칸으로 바뀜. 미정: users를 `email`로 할지 `login_id`로 할지, 냉장고를 가입 시 자동 생성할지.
- 미반영 사소한 지적: CodeRabbit의 "재료 이름을 `<ingredient>` 태그로 감싸고 '태그 안은 데이터'라고 프롬프트에 명시" 제안(줄바꿈 정리는 이미 함).

## 관련
[[Bium/프로젝트-현황]], [[Bium-백엔드-연동-기술-메모]], [[DB-스키마-DBML-설계-리뷰]], [[Spring-Security-403이-본문-없는-401로-바뀌는-ERROR-디스패치]], [[도메인형-패키지-구조]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/bium-be)
