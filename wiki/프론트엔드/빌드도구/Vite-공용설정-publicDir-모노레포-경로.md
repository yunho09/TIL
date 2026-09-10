---
tags: [frontend, vite, monorepo, publicDir]
updated: 2026-09-10
---

# Vite 공용 설정의 publicDir이 모노레포에서 엉뚱한 곳을 가리키는 함정

여러 앱이 공유하는 `vite.config.common.ts` 같은 팩토리에서 `publicDir` 경로를 `import.meta.dirname`(공용 설정 파일 자신의 위치) 기준으로 계산하면, 그 경로는 **레포 루트의 `public/`**로 고정된다. 각 앱(`apps/<app>/public/`)에 있는 `public/` 폴더는 서빙 대상이 아니게 되어 죽은 폴더가 된다.

## 증상

- `apps/student/public/office-building.png`처럼 앱별 `public/`에 파일을 넣어도 `/office-building.png`로 접근이 안 된다(404).
- 앱마다 `favicon.ico` 같은 파일이 앱 `public/`에 있는데 실제 `index.html`은 레포 루트 `public/`의 `logo.svg`를 참조하는 등, 앱 폴더와 실제 서빙 폴더가 어긋나 있어도 겉으로는 잘 동작해서 눈치채기 어렵다.

## 원인

`publicDir`을 상대경로 문자열이 아니라 `import.meta.dirname` 같은 절대경로 계산으로 공용 설정 파일에 박아두면, 그 설정을 어느 앱이 가져다 쓰든 항상 같은 절대경로(공용 설정 파일 기준)로 고정된다. 앱별로 다른 `publicDir`을 쓰려면 각 앱의 `vite.config.ts`에서 공용 팩토리 호출 시 `publicDir`을 오버라이드해야 한다.

## 확인 방법

`vite.config.common.ts`(또는 동급 공용 설정)에서 `publicDir`이 어떻게 계산되는지 먼저 보고, 실제로 새 정적 에셋을 앱 `public/`에 넣기 전에 그 경로가 서빙되는지부터 확인한다. 안 되면 레포 루트 `public/`에 넣거나 공용 설정을 고친다.

관련: [[JOBIS-FE-V2/프로젝트-현황]] — 학생 앱 홈 배너 이미지를 `apps/student/public/`에 뒀다가 안 뜨는 걸 발견하고, 레포 루트 `public/`에 옮겨 해결한 사례.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
