---
tags: [xquare, deploy, ssh, nginx, github-actions, self-hosted-runner]
updated: 2026-11-11
---

# xquare 프론트 배포: SSH 게이트웨이 VM과 self-hosted runner 결정

xquare 프론트(user 앱)를 `XQUARE-frontend` VM에 nginx로 올리고, 자동 배포는 SSH 대신 GitHub self-hosted runner로 하기로 한 과정과 확정 사항. admin 앱은 아직 Vite 기본 템플릿 상태라 배포 대상은 user 앱 하나. 정적 서빙 원리는 [[Vite-SPA-nginx-정적-서빙과-try_files-캐시]].

## 서버 진단 (접속 결과 해석)
- `xquare.xyz:2222`는 **AsyncSSH(파이썬) 기반 중계 게이트웨이**이고 비밀번호만 허용 → `ssh-copy-id` 소용없음. 접속하면 VM(root, systemd, 내부에 OpenSSH :22)으로 연결.
- VM에는 rsync·curl만 있고 nginx·git·bun·node 없음, 80/443 미개방. 빌드는 PC/CI에서 하고 산출물만 올리면 되므로 node 불필요.
- `xquare.xyz` 자체는 앞단 openresty가 받는 다른 서버(테스트 문구 응답, TLS 없음) → 이 VM에 nginx를 깔아도 **앞단이 포트를 넘겨주기 전엔 외부 접근 불가**.
- 구분법: SSH 배너가 `OpenSSH`가 아니면 중계기 의심, PID1이 systemd면 VM·bash/sh면 컨테이너.
- 접속 정보·비밀번호는 채팅에 올리지 않고 `~/.ssh/config` 별칭으로만 관리.

## 확정 사항 (2026-11-11)
| 항목 | prod | stag |
|---|---|---|
| 프론트 | `xquare.dsmhs.kr` | `xquare-stag.dsmhs.kr` |
| API | `openapi.dsmhs.kr` | `openapi-stag.dsmhs.kr` |
| 서버 포트(제안) | 80 | 8080 |
- 도메인 연결·HTTPS 인증서는 앞단 담당자가 처리 → VM은 http만 열면 됨. 우리는 **포트를 정해 알려주는** 쪽. 방화벽 막힘 없음, VM은 프론트 전용(nginx 설치 OK).
- GitHub App 값(`VITE_GITHUB_CLIENT_ID`·`APP_SLUG`)은 stag만 수령, prod는 추후. 콜백 URL에 각 프론트 도메인 등록 필요.

## 자동 배포 설계 결정: SSH/rsync 대신 self-hosted runner
- 이유: 담당 측이 SSH 방식 자동 배포를 원치 않음. runner는 서버가 GitHub로 먼저 연결하므로 인바운드 접속이 필요 없음(서버 외부 인터넷 가능).
- 구조: 빌드 job은 `ubuntu-latest`(env는 GitHub Secrets — `.env`는 gitignore라 CI에 없음), 배포 job은 `runs-on: self-hosted`에서 `upload/download-artifact`로 산출물만 받아 폴더 교체 → 서버에 bun/node 불필요, 외부 코드가 서버에서 실행되지 않음.
- 배포 job은 **`push` 이벤트에서만** 실행(`pull_request`로 돌면 외부 PR 코드가 서버에서 실행될 위험). 브랜치 매핑 예: `develop`→stag, `main`→prod.
- runner 설치: 레포 Settings → Actions → Runners → New self-hosted runner(**레포 관리자 권한 필요**). `./run.sh`는 터미널 종료 시 꺼지므로 `./svc.sh install && ./svc.sh start`로 서비스 등록.
- 초기 nginx 세팅은 SSH로 수동, 자동 배포에서만 SSH 배제.

## 출처
- Claude Code 세션 자동 캡처 (xquare-admin-application, 2026-11-11) — raw 파일 없음
