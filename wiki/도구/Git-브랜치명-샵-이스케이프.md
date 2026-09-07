---
tags: [git, shell]
updated: 2026-09-07
---

# Git 브랜치명의 `#` 이스케이프

브랜치 이름에 `#`이 들어가면(예: 이슈 번호 접두사 컨벤션 `#58/feature-name`) 쉘이 `#` 이후를 주석으로 처리해 명령이 그 지점에서 잘린다.

```sh
git push origin --delete #58/task-management-api    # 잘못됨 — --delete 인자가 통째로 주석 처리되어 사라짐
git push origin --delete "#58/task-management-api"  # 올바름 — 항상 따옴표로 감싼다
```

`#이슈번호/` 형태의 브랜치 네이밍 컨벤션을 쓰는 저장소에서는 `git push`/`git branch`/`git checkout` 등 브랜치명을 인자로 넘기는 모든 명령에서 따옴표로 감싸는 습관이 필요하다.

관련: [[ToyVillage-Admin-FE/프로젝트-현황]] (이 컨벤션을 쓰는 저장소)

## 출처
- Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
