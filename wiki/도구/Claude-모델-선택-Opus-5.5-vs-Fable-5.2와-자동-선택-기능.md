---
tags: [claude-code, 모델, opus, fable, effort, advisor]
updated: 2026-11-11
---

# Claude 모델 선택: Opus 5.5 vs Fable 5.2와 자동 선택 기능

평소 코딩은 **Opus 5.5**, 정말 어려운 작업만 **Fable 5.2 높음**이 세션에서 내린 결론이다. Fable 5.2는 2026-11-09 출시되며 가격이 20% 내려 입력 $8 / 출력 $40(Opus 5.5의 2배)이다. Claude Code에는 작업 난이도를 보고 모델을 알아서 골라주는 기능이 없고, 비슷한 역할은 어드바이저·`opusplan`·서브에이전트별 모델 지정 등으로 대신한다.

## 결정 과정에서 정정된 점
- 처음 답은 "Fable 5.2 높음 추천"이었으나 웹 조사 후 **Opus 5.5 기본, 어려운 일만 Fable**로 정정. 처음 쓴 Fable 가격($10/$50)은 5.1 기준이라 틀렸다.
- 노력 수준(effort)은 같은 모델 안에서 얼마나 꼼꼼히 하느냐만 바꾸고 모델 등급 차이를 메우지 못한다는 전제는 유지.

## 가격 (100만 토큰, 입력/출력)
- Opus 5.5: $4 / $20
- Fable 5.2: $8 / $40 (출시 시 20% 인하)
- 노력 수준이 높을수록 토큰을 더 써서 작업당 비용 차이는 토큰당 가격 차이와 다르다.

## 어려운 작업: Fable 높음이 유리 (Anthropic 자체 측정)
- FrontierCode 2.0(에이전트 코딩): Fable 5.2 엑스트라 49.1%, Opus 5.5 최대 45.5%, Fable 5.2 높음 45.3%.
- Fable 높음이 Opus **최대**와 비슷한 점수를 작업당 약 절반 비용으로 낸다는 해석. Opus **엑스트라** 점수는 공개되지 않았다.

## 평범한 작업: Opus가 싸고 빠름 (독립 테스트 2건, 규모 작음, Opus 엑스트라는 미측정)
- 코딩 과제 1개, 둘 다 높음: Opus 9.6분·$1.70 vs Fable 26.6분·$7.70. 숨은 테스트는 둘 다 전부 통과.
- 실제 TS 모노레포 52회, 둘 다 보통: Opus 44회 통과·$78 vs Fable 40회 통과·$128. Fable 실패 5회는 작성자가 "수정은 맞는데 테스트가 너무 엄격"했다고 판단.
- 같은 테스트에서 **코드 리뷰는 Fable이 우세**: Opus가 놓친 결함을 Fable은 4번 중 4번, Opus는 1번 잡음.

## 그 밖에
- **속도**: Fable은 높음 이상에서 거의 모든 답변 전에 생각해서 간단한 질문도 느리다.
- **구독 한도**: Pro는 출시 프로모션으로 주간 한도의 50%까지만 Fable 사용 가능. Max 조건은 발표에 없음.
- **Anthropic 권장**: Opus 5.5로 시작하고, Opus 노력 수준을 올려도 부족할 때 Fable로 올린다.
- 용도 정리: Opus 엑스트라는 일상 구현·요구가 분명한 티켓·빠른 왕복, Fable 높음은 오래 걸리는 복잡한 작업·틀리면 비싼 수정(결제·라우팅·외부 연동)·코드 리뷰.

## 모델 자동 선택 기능은 있나
**없다.** Claude Code 공식 문서(Model configuration)에 `auto` 모델은 없고 모델·노력 수준은 직접 정한다. 비슷한 기능:
- **어드바이저**(`advisorModel` 설정, `--advisor` 플래그): 주 모델이 필요하다고 판단할 때만 더 센 모델에게 묻는다. Opus 5.5 주 모델 + Fable 5.2 어드바이저가 "평소엔 Opus, 어려울 때만 Fable"에 가장 가깝다. 세부 동작과 데스크톱 앱 지원은 문서에서 **미확인**.
- **`opusplan`**: 계획 모드는 Opus, 실행은 Sonnet으로 자동 전환(Fable 무관).
- **서브에이전트별 모델 지정**: 에이전트 정의의 `model`로 작업별 모델을 고정. OMC의 haiku/sonnet/opus 배분이 이 방식.
- **`ultrathink`** 키워드: 그 턴만 노력 수준을 엑스트라로 올림.
- **적응형 사고**: 정해둔 노력 수준 안에서 단계마다 생각량을 모델이 조절.
- 자동 전환이 일어나는 경우는 모델 과부하·오류 시 대체 모델, 안전 분류기에 걸렸을 때뿐이며 작업 난이도와 무관.
- "자동 모델 선택이 기본값이 됐다"는 8월 뉴스레터 글은 공식 문서에 근거가 없고, 권한 관련 auto mode와 혼동한 것으로 보인다.

## 출처
- Claude Code 세션 자동 캡처 (scratch-2026-11-10-6bebd8, `.omc/session-capture/20261111-144628-7f0f3950.txt`)
- 세션에서 인용된 웹 자료: Anthropic "Introducing Claude Fable 5.2", Claude Platform Docs(What's new in Fable 5.2, Pricing), Claude Code Docs(Model configuration), atomicagent.io·paddo.dev 비교 글
- 관련: [[Claude-Code-서브에이전트-fan-out-토큰-과다소모]], [[oh-my-openagent-omo-설치와-OMC-비교]]
