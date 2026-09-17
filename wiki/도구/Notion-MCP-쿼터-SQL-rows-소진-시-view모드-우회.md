---
tags: [notion, mcp]
updated: 2026-09-17
---

# Notion MCP 쿼터: SQL/rows 모드 소진 시 view 모드로 우회

Notion data source에 대한 SQL 쿼리(`COUNT` 등)와 rows 조회는 **같은 쿼터를 공유**하며, 반복 호출하면 소진돼 더 이상 쓸 수 없게 된다.

반면 데이터베이스의 **"view" 조회는 이 쿼터를 소모하지 않는다(quota-free).** SQL/rows 경로가 막혔을 때는 view 모드로 전체 행을 읽어 개수 세기·중복 여부 확인 같은 걸 대신 처리할 수 있다.

## 대응

- Notion data source를 반복적으로 조회해야 하는 작업(전수 확인, 개수 검증 등)에서 SQL/rows 쿼터가 소진되면 바로 실패로 끝내지 말고 view 모드 조회로 전환할 수 있는지 확인한다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
