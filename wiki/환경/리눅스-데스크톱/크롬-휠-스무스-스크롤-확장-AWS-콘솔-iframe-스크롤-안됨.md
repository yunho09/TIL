---
tags: [chrome, 확장, 스크롤, aws, iframe]
updated: 2026-11-17
---

# 크롬 휠 스무스 스크롤 확장 때문에 AWS 콘솔 스크롤이 안 됨

AWS 콘솔(EC2 등)에서만 마우스 휠 스크롤이 안 되고 왼쪽 메뉴가 대신 움직이던 원인은 크롬에 깔린 **Chromium Wheel Smooth Scroller** 확장이었다. AWS 콘솔은 본문 전체가 **iframe** 안에 있고, 이 확장이 iframe 안의 휠 입력을 막아 바깥 페이지로 넘기기 때문이다. 확장의 **Black List**에 두 줄을 넣어 해결했다.

## 증상과 원인
- 다른 사이트는 스크롤이 잘 되는데 AWS 콘솔만 본문이 안 내려간다. 본문 위에서 휠 3칸 → 본문 0px, 왼쪽 메뉴 132px 이동.
- 페이지 자체는 `preventDefault`를 부르지 않는데도 휠 이벤트가 막혀 있었다 → 페이지가 아니라 확장이 가로채는 것.
- 다른 사이트는 본문이 iframe이 아니라서 문제가 없었다.
- 터치패드용 `chrome-scroll-boost` 확장은 원인이 아니었다 ([[터치패드-전역-스크롤-속도-libinput-quirks]]).

## 해결: 확장 옵션의 Black List
- 확장 옵션 페이지(`chrome-extension://<확장ID>/options.html`)는 외부 도구로 못 열어 사용자가 직접 연다.
- Black List에 한 줄씩 추가:
  - `https://*console.aws.amazon.com/*` — 저장 시 정규식 `https://.*console\.aws\.amazon\.com/.*$`로 변환됨. 이것만으로는 top 페이지 쪽만 풀리고 iframe은 계속 막힘.
  - `about:blank` — AWS 본문 iframe은 주소 없이 만들어져, 확장이 붙을 때 그 프레임 URL을 `about:blank`로 기억해 도메인 규칙에 안 걸린다.
- 검증: 같은 조건에서 휠 3칸 → 본문 300px 이동, 왼쪽 메뉴 그대로.
- 부작용: AWS 콘솔과 주소 없는 iframe에서는 부드러운 휠 효과가 빠지고 크롬 기본 스크롤로 동작. 나머지 사이트는 동일.

## 진단 요령
- "스크롤이 안 되고 엉뚱한 곳이 움직이는" 증상이 iframe 중심 사이트에서 나오면 휠을 가로채는 확장을 먼저 의심한다.
- 확장이 막는 프레임의 URL이 `about:blank`일 수 있으니 도메인 규칙만 넣고 끝내지 말고 iframe 안에서 실제 휠 입력으로 재확인한다.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-11-17.
