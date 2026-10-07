---
tags: [claude-code, git, 설정]
updated: 2026-10-07
---
# Claude Code AI 표기(Co-Authored-By) 끄기

Claude Code가 커밋에 붙이는 `Co-Authored-By` 줄과 PR 본문의 `Generated with Claude Code` 줄은 저장소의 `.claude/settings.json`에서 `attribution`으로 끈다. 파일을 git에 올리면 그 저장소에서 Claude Code를 쓰는 팀원 전원에게 같은 설정이 적용된다.

## 설정
```json
{
  "attribution": { "commit": "", "pr": "", "sessionUrl": false },
  "includeCoAuthoredBy": false
}
```
- `commit`·`pr`을 빈 문자열로 두면 해당 표기가 사라진다. 설정 직후 세션의 커밋·PR 표기 지침이 "표기 줄 넣지 않음"으로 바뀌는 것으로 적용 확인.
- **`sessionUrl: false`가 따로 필요하다.** 기본값 `true`라 웹·Remote Control 세션에서는 `Claude-Session:` 트레일러와 PR 본문 세션 링크가 계속 붙는다(CodeRabbit 지적으로 발견, Claude Code 2.1.292에서 확인).
- 예전 버전은 `attribution: false` 한 줄 형식을 거부하므로 **객체 형식**으로 쓴다. `includeCoAuthoredBy: false`는 구버전용.
- 이 설정은 Claude Code만 읽는다. Codex 등 다른 도구는 `AGENTS.md`/규칙 문서에 "표기 줄 금지"를 문장으로 적어야 한다.
- 개인 메모리에 "표기 넣지 않음"을 적어도 리마인더 지침이 표기를 붙이라고 하면 충돌하므로, 설정으로 끄는 쪽이 확실하다.

## 관련
- [[ToyVillage-Admin-FE/팀-작업-규칙-하네스-문서]] — 이 설정을 적용한 사례

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE) — 2026-10-07
