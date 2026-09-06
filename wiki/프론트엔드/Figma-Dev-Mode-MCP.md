---
tags: [figma, mcp, design-to-code]
updated: 2026-09-06
---

# Figma Dev Mode MCP

Figma 파일을 Claude가 직접 읽어 디자인 스펙(레이아웃·색상·타이포)을 추출할 수 있게 해주는 MCP 서버(`127.0.0.1:3845`에 로컬로 뜬다). 두 가지 전제 조건이 안 맞으면 조용히 실패하는 게 아니라 명확한 에러를 낸다.

## 동작 조건

- **Figma 데스크톱 앱**에 대상 파일이 열려 있어야 한다. 웹 브라우저 탭에 열려 있는 것과는 무관하다 — MCP는 데스크톱 앱의 플러그인 API를 통해 씬 그래프에 접근한다.
- 열려 있어도 파일이 **view-only**(편집 권한 없음, 예: 팀 소속 인증이 안 된 상태) 상태면 `get_design_context` 같은 씬 그래프 조회 API가 `Figma Plugin API not available — ensure a file is open and fully loaded` 에러를 낸다.

## view-only일 때의 대안

- `get_screenshot`은 view-only에서도 `capturePage` 방식으로 폴백 동작해서 화면 캡처 자체는 가능하다.
- 다만 레이어별 정확한 속성(정확한 px 수치, 색상 hex, 텍스트 스타일 이름)은 못 가져오므로, 캡처된 스크린샷을 **픽셀 단위로 직접 계측**해서 수치를 역산해야 한다. 이때:
  - 프레임 폭이 Figma 우측 패널에 표시돼 있으면 그 값으로 캡처 이미지의 스케일(px/design-px)을 정확히 잡을 수 있다.
  - 폰트 크기는 감으로 추정하지 말고, 후보 크기별로 실제 폰트를 렌더링해 잉크 바운딩박스 크기를 계측값과 대조하면 정확히 특정할 수 있다.
  - 그래도 ±1~2px 오차는 남을 수 있어, 편집 권한이 풀리면 인스펙터 값으로 재검증하는 게 좋다.

관련: [[JOBIS-FE-V2/프로젝트-현황]] — 이 조건을 실제로 겪은 사례(학생 지원하기 화면 퍼블리싱).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
