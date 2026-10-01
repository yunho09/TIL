---
tags: [github, pull-request, ruleset, codeowners]
updated: 2026-10-01
---

# GitHub: closing keyword 자동 닫힘과 코드오너 룰셋 함정

`resolved #N` 같은 닫기 키워드는 **저장소 기본 브랜치로 머지될 때** 대상 이슈/PR을 자동으로 닫는다. 그리고 main 보호 룰셋의 "코드오너 승인 필수"는 일반 승인 개수와 별개 조건이라, 승인이 있어도 머지가 막힌 것처럼 보인다. 둘 다 PR 본문·룰셋을 읽는 방식 때문에 생기는 오해.

## 닫기 키워드가 다른 PR까지 닫는다
- 사례: 리뷰 반영 PR(#161, 대상 `develop`) 본문의 템플릿 항목 `## 해결 이슈`에 `- resolved #160 리뷰 코멘트`라고 적음 → 이 저장소 **기본 브랜치가 `develop`**이라 #161 머지 2초 뒤 `develop→main` PR #160이 자동으로 닫힘(타임라인 `closer: PullRequest #161`). 닫기 키워드의 대상은 이슈뿐 아니라 PR도 된다.
- 복구: PR 재오픈 + 본문 문구를 단순 참조("#160 리뷰 코멘트 반영")로 교체. 리뷰 스레드는 닫혔다 열려도 resolve 상태 유지.
- 예방: 템플릿을 채울 때 `resolved/closes/fixes #N`을 PR 간 참조에 쓰지 않는다. 이슈를 닫으려는 의도일 때만 쓴다(작업 PR이 develop에 머지돼 이슈가 자동으로 닫히게 하는 용도는 정상).

## 룰셋 읽는 법 (main "Protect Main" 예시)
| 필드 | 의미 |
|---|---|
| `required_approving_review_count` | 필요한 승인 수 |
| `require_code_owner_review` | CODEOWNERS에 적힌 사람의 승인이 **별도로** 필요 |
| `require_last_push_approval` | 마지막 푸시한 사람의 승인은 카운트 안 됨 |
| `required_review_thread_resolution` | 리뷰 스레드 전부 resolve |
| `dismiss_stale_reviews_on_push` | 새 푸시 시 기존 승인 무효화 |

- `* @a @b` 식 CODEOWNERS면 모든 파일의 오너가 그 둘뿐이라, **오너가 아닌 사람이 몇 명 승인해도** `REVIEW_REQUIRED`/`BLOCKED`. 승인 1 + 코드오너 필수일 때 실제로 필요한 사람은 **오너 1명**(오너 승인이 카운트도 같이 채움). 리뷰 "요청"이 걸린 사람 수(PR 화면의 4명처럼 보이는 것)는 승인 의무가 아니다.
- 함정: "아무나 2명"으로 바꾸면(count 2 + 코드오너 해제) 필요 인원이 오히려 **1명→2명으로 늘어난다.** 목표가 "사람 덜 필요하게"면 방향이 반대. 룰셋을 바꾸기 전에 현재 필요한 인원을 먼저 계산할 것.
- 룰셋은 변경 전 JSON을 안전한 곳(임시 디렉터리 말고)에 백업해 두고, 되돌릴 땐 바꾼 필드만 되돌린다. 임시 디렉터리 백업은 세션이 바뀌며 사라졌다.
- **CODEOWNERS는 PR의 base 브랜치(main)에 있는 파일이 기준**이다. develop에서 고쳐 develop→main PR에 태워도 이번 PR의 오너 판정은 바뀌지 않는다 → 변경은 다음 PR부터 적용. 오너로 인정되려면 해당 사람에게 레포 Write 이상 권한 필요. 폴더별 오너는 아래 줄이 우선.
- 급할 때 선택지: 오너 승인 받기 / 관리자 bypass 머지 / 코드오너 옵션 **잠깐 끄고 머지한 직후 즉시 복구**(백업 → 한 필드만 변경 → 머지 → 복구 순서로 진행해 원래 설정과 동일하게 복원).
- 필수 상태 체크가 없는 룰셋이면 실패한 체크(유령 Workers Builds 등)는 머지를 막지 않고 `UNSTABLE`로만 표시된다.

## 같이 알아둘 것
- PR 리뷰 반영·머지는 기본적으로 Assignee 담당. 마지막 푸시자 본인 승인은 `require_last_push_approval` 때문에 승인 수에 안 들어간다.
- 보호된 `develop`은 직접 푸시가 거부(`Changes must be made through a pull request`)되므로 리뷰 반영 커밋은 별도 브랜치+PR로 올린다.
- CodeRabbit이 diff 밖 줄을 지적하면 인라인 스레드가 아니라 리뷰 본문의 "Outside diff range comments"로 달려 **resolve할 스레드가 없다** → PR에 일반 코멘트로 커밋 해시를 남긴다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — [[JOBIS-FE-V2/프로젝트-현황]]
