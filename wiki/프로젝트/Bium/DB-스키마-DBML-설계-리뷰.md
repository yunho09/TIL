---
tags: [Bium, DBML, ERD, MySQL, JPA]
updated: 2026-10-21
---
# Bium DB 스키마(DBML) 설계 리뷰

Bium 전체 데이터 모델을 **DBML**(dbdiagram.io에서 ERD로 그려 주고 MySQL/PostgreSQL DDL로 변환하는 스키마 표기)로 작성한 설계서를 리뷰한 결과다. 그대로 구현하면 MySQL에서 실제 버그가 되는 항목, 구현 전에 정해야 할 항목, 현재 코드(MVP)와의 차이를 정리했다.

## DBML 문법 요약
- `Enum`은 값 목록, `Table`은 테이블. `[pk, increment]` 자동 증가 PK, `[unique, not null]` 제약.
- `ref: > fridges.id`는 다대일 FK, `ref: -`는 일대일.
- `indexes { ... }`에서 복합 unique·복합 PK 지정, `note`는 주석.

## 스키마 구성 (영역별)
- 회원·공유: `users`, `fridges`, `fridge_members`(OWNER/MEMBER), `fridge_invites`(초대 코드)
- 도감·영양: `ingredient_catalog`(`owner_id` NULL=공용, 값=개인 커스텀), `nutrition_info`
- 냉장고 재료: `fridge_items` — 낙관적 락 `version`, 소프트 삭제 `deleted_at`, 정렬 `sort_order`
- 장보기: `purchased_items`, `shopping_list_items`
- 레시피·차감: `recipes`, `recipe_ingredients`, `cooking_logs`, `cooking_log_items`
- 활동·알림·피드: `activity_logs`(재료 삭제 후에도 보이게 `item_name` 스냅샷), `notification_settings`(`last_sent_date`로 중복 발송 방지), `notification_logs`, `feed_likes`, `feed_saves`

## 🔴 구현 전에 고쳐야 할 것
- **공용 도감 unique 무력화**: MySQL은 unique 인덱스에서 NULL끼리 서로 다른 값으로 본다. `(owner_id, name)` unique는 `owner_id=NULL`인 공용 도감에서 같은 이름이 중복 삽입된다. 해결: `owner_id=0` 고정값(FK 불가) 또는 생성 컬럼 `owner_key = IFNULL(owner_id, 0)`에 `(owner_key, name)` unique.
- **되돌리기를 `quantity_before`로 복원하면 안 됨**: 공유 냉장고라 요리 후 되돌리기 전에 다른 멤버가 수량을 바꿀 수 있다(3개→차감 2개→타인이 +2로 4개→되돌리면 3개로 덮어써 추가분 소실). 복원은 **현재 수량 + `deducted_quantity`**, `quantity_before`는 기록용. 수량 0으로 소프트 삭제된 재료의 되살리기 규칙도 함께 정할 것.
- **장보기 테이블이 `user_id` 기준**: 공유 냉장고면 `shopping_list_items`는 `fridge_id` 단위여야 하고, `purchased_items`에는 어느 냉장고로 옮겼는지 알 `fridge_id`와 `moved_at`(또는 생성된 `fridge_item_id`)이 필요.
- **알림 대상 정보 부재**: 언제 보낼지만 있고 어디로 보낼지가 없음. 푸시/FCM이면 `user_devices(user_id, token, platform, created_at)` 필요. (이메일 알림 채택은 [[Bium/프로젝트-현황]] 참고)

## 🟡 구현하면서 개선
- 인덱스: `fridge_items`는 `deleted_at IS NULL`로 항상 거르므로 주 인덱스에 `deleted_at` 고려, 만료 알림 배치는 `(expires_at, deleted_at)`, 냉장고별 임박 목록은 `(fridge_id, expires_at)`. `feed_likes`/`feed_saves`는 PK가 `(user_id, fridge_id)`라 냉장고별 카운트가 풀스캔 → `fridge_id` 인덱스 또는 `fridges.like_count` 비정규화.
- 피드 대상이 냉장고인지 레시피인지 확정(레시피면 `recipe_id` 참조).
- 단위 변환: 레시피 `g` vs 재료 `개`면 차감 불가 → 도감에 환산 값을 두거나 단위가 다르면 차감하지 않는 규칙.
- varchar로 둔 `recipes.type`, `purchased_items.source`, `activity_logs.action`, `notification_logs.status`는 enum으로 통일(JPA `@Enumerated(STRING)`).
- 냉장고 OWNER 1명 보장이 DB에 없음(앱에서 보장하거나 `fridges.owner_id`), OWNER 탈퇴 시 권한 이양 규칙, 회원 탈퇴를 소프트 삭제/익명화 중 무엇으로 할지.
- `created_at`/`updated_at` 누락 테이블, `notify_time`의 시간대 기준.

## 현재 코드(MVP)와의 차이
- `FridgeIngredient`(≈`fridge_items`)는 `name/emoji/zone/expiryDate`뿐 — `fridge_id`, 수량·단위, `version`, 소프트 삭제 없음.
- `FridgeZone` 6값(`SHOPPING_BAG, LEFT_DOOR, RIGHT_DOOR, PANTRY, FRIDGE, FREEZER`) vs 스키마 `storage_type`(FRIDGE/FREEZER/ROOM)×`section_type`(DOOR/CENTER) 두 축. 좌/우 문 구분은 `DOOR` 하나로 표현 불가, 장바구니 단계는 스키마에선 `purchased_items`로 분리.
- `Recipe`의 `difficulty`, `tips`, `emoji`, 영양 정보는 스키마에 없고, 스키마의 `fridge_id/requested_by/type`은 코드에 없음. `recipe_ingredient.amount`가 `"2큰술"` 문자열 → 차감 기능 때는 `quantity`+`unit`으로 분리(Gemini 프롬프트·파서도 함께 수정).
- 필드명 차이: `name`↔`title`, `cookTimeMinutes`↔`cook_minutes`, `expiryDate`↔`expires_at`.

## 결정: 레시피 추천 브랜치는 기다리지 않고 진행
- 위 문제는 모두 아직 안 만든 테이블 이야기이고, 브랜치(`3-feat-recipe-recommendation-api`)는 `FridgeIngredient`·`Recipe`만 쓰는 독립 MVP라 막히는 게 없음.
- 로그인 브랜치(`5-feat-login-singup`)와도 겹침 없음, `ddl-auto: update` 개발 단계라 옮길 데이터 없음 → 스키마 확정 뒤 엔티티를 맞추면 됨.
- 오히려 **DBML 쪽을 먼저 보강**: `emoji`, `difficulty`, `tips`, 레시피 영양 정보, 좌/우 문 구분 추가 + 위 🔴 항목 반영.

관련: [[ERD-to-JPA-엔티티-변환-팁]], [[MySQL-연동]], [[도메인형-패키지-구조]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/bium-be)
