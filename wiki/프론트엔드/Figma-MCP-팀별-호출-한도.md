---
tags: [figma, mcp, api-limit]
updated: 2026-09-07
---

# Figma MCP 팀별 호출 한도

`claude.ai Figma`(호스트형 Figma MCP 커넥터, [[Figma-Dev-Mode-MCP]]와는 다른 도구)의 도구 호출에는 한도가 있다. 이 한도는 로그인 계정이 아니라 **열람 중인 파일을 소유한 팀의 Figma 플랜**에 걸린다.

## 증상과 진단

```
whoami                → 200 OK (인증 정상)
get_metadata(fileKey) → 403 "Figma MCP tool call limit on the Starter plan"
```

`whoami`는 계정 인증만 확인하므로 한도와 무관하게 계속 성공한다. **`whoami`는 되는데 `get_metadata`/`get_screenshot` 같은 파일 조회 도구만 막히면 인증 문제가 아니라 그 파일을 소유한 팀의 쿼터 소진**이다 — 재로그인으로는 풀리지 않는다.

## 한도는 계정 전역이 아니라 파일(소유 팀) 단위

같은 세션·같은 계정에서도 **다른 파일**(다른 팀 소유, 특히 Full 시트 팀 소유)은 영향받지 않고 정상 조회된다. 링크 공유로 열람 중인 파일이 `whoami` 응답의 소속 팀 목록에도 없는 제3의 팀 소유일 수 있고, 그 팀이 Starter 플랜이면 한도가 낮아 금방 걸린다.

## 한도를 빨리 소모하는 실수

nodeId를 모른 채 `get_metadata`를 페이지 루트(예: URL의 `node-id=0-1`)로 호출하면 파일 전체가 통째로 덤프된다(실측 사례: 74만~100만 자). 필요한 프레임 하나만 보려던 것이어도 호출 한 번으로 한도 대부분이 소모될 수 있다. **원하는 섹션/프레임의 nodeId를 먼저 좁혀서 조회**하는 편이 안전하다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
