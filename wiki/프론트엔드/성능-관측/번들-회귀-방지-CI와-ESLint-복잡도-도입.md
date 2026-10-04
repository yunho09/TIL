---
tags: [size-limit, lighthouse-ci, eslint, complexity, github-actions, bundle]
updated: 2026-10-04
---

# 번들 회귀 방지 — size-limit·Lighthouse CI·ESLint 복잡도 (baseline 방식)

실사용자 모니터링(Sentry)은 배포 뒤에 알려주므로 "성능을 고쳐놔도 누군가 페이지 하나를 정적 import하면 조용히 원상복구"되는 것은 머지 전 CI 게이트로 막아야 한다. 도입 순서·설정 예시·주의점을 정리한다(ToyVillage-Admin-FE 검토 내용).

## 전제: CI가 없었다
- `.github/`에 PR·이슈 템플릿만 있고 워크플로가 없어 `yarn verify`(lint+typecheck+style-policy+build)가 로컬에서만 돌았다. 두 게이트를 붙이려면 **GitHub Actions 워크플로부터** 만들어야 한다(약 2시간). 모니터링과 별개로 "검증 없이 머지되는 PR" 방지에도 필요.

## size-limit — 번들 크기 예산
- 빌드 산출물 크기에 상한을 두고 넘으면 CI 실패. `.size-limit.js`에 `{ name, path: 'dist/assets/vendor-*.js', limit: '150 kB' }` 식으로 청크별 지정, **gzip 기준**.
- 한도는 스플리팅 후 실제 값에서 **10~15% 여유**로 잡는다. 처음부터 목표치로 조이면 매일 빨개진다.
- 번들은 한 번에 100kB가 아니라 **PR마다 3kB씩** 커져 반년 뒤 다시 1MB가 된다 — PR 코멘트에 `vendor 142 kB (+0)` 같은 증감을 보여준다.
- 서버 없이 빌드 산출물만으로 동작, 결정적(오탐 없음), 30분. **먼저 붙일 것**.
- 실증: 예산 도구 없이 6일 동안 96커밋이 쌓이며 메인 JS가 957.66→984.84 kB로 27 kB 늘었다.

## Lighthouse CI — 성능 점수 하한선
- 헤드리스 크롬으로 `vite preview` 서버의 페이지를 열어 점수·지표(LCP·TBT 등)가 기준 미달이면 실패. size-limit이 "얼마나 받나"라면 이쪽은 "받은 뒤 언제 그려지나" — 번들을 줄여도 렌더 차단 폰트(jsdelivr, preconnect 없음)가 있으면 size-limit은 통과하고 Lighthouse만 잡는다. 접근성 점수도 부수적으로 나온다.
- 러너 성능에 따라 점수가 출렁이므로 `numberOfRuns: 3` 이상 중앙값, **처음 2~3주는 `warn`으로 편차를 본 뒤** `error`로 확정.
- 인증 필요한 화면은 puppeteer로 로그인을 먼저 태워야 해서 1단계는 로그인 화면만.
- **코드 스플리팅 끝낸 뒤에** 붙인다 — 그 전에는 기준선을 정할 수 없다. 반나절~4h.

## ESLint 복잡도 규칙 — baseline으로 신규 유입만 차단
- 기본 임계로 임시 측정한 결과(377개 소스): `max-lines-per-function` 100줄 52건, `complexity` 10 36건(20 이상 5건, 최대 32 `IndividualDetailPage`), `max-lines` 300줄 32건, **`max-depth`·`max-params` 0건**.
- 해석: 중첩·인자 위반이 0이면 코드가 지저분한 게 아니라 **한 컴포넌트가 너무 많은 일**을 해서 복잡한 것 → 처방은 규칙이 아니라 커스텀 훅 추출·컴포넌트 분리, 규칙은 다시 안 뭉치게 막는 장치.
- 주의 3가지: (1) 120건을 한 번에 고치지 말고 **baseline을 기록하고 신규 유입만 차단** (2) 임계값은 **15부터**(JSX 조건부 렌더로 React 컴포넌트 복잡도가 자연히 올라감; 36→약 13건), 안정되면 12→10 (3) `max-lines-per-function`은 마크업이 긴 컴포넌트에 안 맞으니 빼거나 200줄로 느슨하게.
- 이미 `eslint-plugin-boundaries`로 FSD 레이어 의존 방향을 린트로 강제 중이라 "규칙을 자동 검사로 강제"하는 기존 방식의 확장이다. 재측정: `npx eslint src --rule '{"complexity":["error",15]}' -f json`.

## 도입 로드맵(최종 판단)
| 순서 | 항목 | 비고 |
|---|---|---|
| 0 | GitHub Actions | 전제, 2h |
| 1 | Sentry(+ErrorBoundary·소스맵) | [[Web-Vitals와-모니터링-도구-역할-분담]] |
| 2 | size-limit | 30m, 스플리팅 직후 |
| 3 | ESLint 복잡도(baseline, 임계 15) | 2h, 모니터링과는 별개로 판단 |
| 4 | Lighthouse CI | 스플리팅 후 |

## 관련
- [[어드민-SPA-번들-성능-baseline과-최적화-우선순위]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-04 (세션 내용은 2026-09-22~28 논의)
