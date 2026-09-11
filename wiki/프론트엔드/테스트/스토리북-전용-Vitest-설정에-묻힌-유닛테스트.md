---
tags: [testing, vitest, storybook, playwright, jsdom]
updated: 2026-09-11
---

# 스토리북 전용 Vitest 설정에 묻힌 유닛 테스트

`storybookTest` 플러그인으로 Storybook 컴포넌트를 Vitest 브라우저 모드에서 돌리도록 설정한 레포에서는, 같은 설정 파일에 일반 `.test.tsx` 유닛 테스트를 끼워 넣을 수 없다. 별도 설정으로 분리해야 하며, 분리 과정에서도 몇 가지 함정이 있다.

## 증상

`vitest.config.ts`가 다음과 비슷하게 스토리북 전용으로 설정된 레포:

```ts
export default defineConfig({
  plugins: [storybookTest({ /* ... */ })],
  test: { browser: { enabled: true, provider: 'playwright' } },
})
```

여기에 `include: ['src/**/*.test.tsx']`를 아무리 추가해도 무시된다. **`storybookTest` 플러그인이 `include`를 스토리 파일(`*.stories.tsx`)로 자체적으로 덮어쓰기 때문**이다. 그 결과 저장소에 `.test.tsx` 파일이 수십 개 있어도 **단 한 번도 실행되지 않은 채 방치**될 수 있다 — CI도 통과하고 로컬도 조용해서, 이 사실 자체를 오랫동안 아무도 눈치채지 못한다(실제 사례: 46개 파일이 그런 상태였음).

## 해결 — 설정 분리

`storybookTest`가 없는 별도 설정 파일(`vitest.unit.config.ts`)을 만들어 유닛 테스트 전용으로 돌린다. `package.json`의 `test` 스크립트에서 두 설정을 모두 돌리도록 연결한다(예: `vitest run --config vitest.config.ts && vitest run --config vitest.unit.config.ts`).

## 환경 선택 — jsdom vs 브라우저 모드

유닛 설정의 `environment`는 **기존 테스트가 어떤 값을 기대하고 작성됐는지**로 판단한다. 브라우저 모드(playwright)로 먼저 돌려봤더니 `toHaveStyle` 단언이 대거 실패했다 — `repeat(3, 1fr)`을 기대했는데 실제 브라우저는 계산된 값 `138px`를 돌려주는 식이다. 이 테스트들은 애초에 **jsdom을 전제로 작성**된 것이었다(레포가 `jsdom` 패키지를 이미 의존성으로 갖고 있었는데 설정이 연결 안 된 상태). `environment: 'jsdom'`으로 바꾸자 관련 실패가 사라졌다.

**판단 기준**: 브라우저 모드는 실제 계산된 CSS 값(px)을 주고, jsdom은 소스 그대로의 값(`%`, `fr`, 변수명 등)을 준다. 기존 테스트 코드의 기대값 형태를 보면 원래 의도한 환경을 알 수 있다.

## `globals: true` 없으면 DOM이 안 지워진다

환경을 맞춘 뒤에도 실패가 176건이나 남아 있었는데, 대부분 `"multiple elements found"` — 테스트 간 DOM이 안 지워지고 누적되는 증상이었다. **testing-library의 자동 `cleanup()`은 Vitest `globals: true` 없이는 등록되지 않는다.** `globals: true`를 켜자(또는 `afterEach(cleanup)`을 수동 등록) 실패가 **176건 → 13건**으로 줄었다. 원인 불명의 대량 실패, 특히 "여러 개 찾음(multiple elements)" 계열 에러가 몰려 있으면 이것부터 의심할 것.

## 남은 실패는 진짜 버그일 수 있다

환경 문제를 다 걷어내고 나면 남는 실패는 컴포넌트/테스트 자체의 문제인 경우가 많다. 실제로 겪은 패턴:

- **이미 제거된 기능을 검사하는 죽은 테스트** — 오래 안 돌던 테스트가 살아나면, 컴포넌트에서 이미 삭제된 prop/기능을 여전히 기대하는 assertion이 나올 수 있다. 지우기 전에 `git log -p`로 그 기능이 의도적으로 제거됐는지부터 확인한다(실수로 빠진 거라면 복구가 맞는 대응).
- **테마 불일치** — 테스트 헬퍼(`renderWithTheme` 등)가 다크 테마로 렌더하는데 테스트는 라이트 테마 값을 기대하는 식의 사소한 불일치.
- **순환 참조로 인한 수집 자체 실패** — 별개 함정, [[배럴-Import-함정]]의 "다른 메커니즘" 절 참고.

## Playwright 브라우저 설치가 막힐 때

브라우저 모드 테스트(스토리북 쪽)를 돌리려면 Playwright chromium이 필요한데, `yarn playwright install chromium`이 실패하는 경우가 있다.

- **Playwright 버전이 리눅스 배포판을 아직 지원하지 않음** — 예: Playwright 1.57이 Ubuntu 26.04 미지원. 최신 배포판일수록 최신 Playwright 릴리즈를 기다려야 할 수 있다.
- **캐시에 있는 브라우저 빌드 번호와 lock의 요구 번호가 다름** — 예: 캐시엔 `chromium_headless_shell-1228`(Chrome 149)만 있고 설치된 playwright 1.57.0은 `-1200`을 요구. 실제로는 상위 호환되는 경우가 많아, `~/.cache/ms-playwright/`에 요구 번호로 심볼릭 링크를 걸어 우회할 수 있다(레포는 안 건드림, `rm` 두 번이면 원복). 환경변수(`PLAYWRIGHT_BROWSERS_PATH` 등)로 우회하려 해도 Vitest 브라우저 모드에는 안 먹힐 수 있으니 심볼릭 링크가 더 확실하다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — [[프로젝트/JOBIS-FE-V2/프로젝트-현황]]
