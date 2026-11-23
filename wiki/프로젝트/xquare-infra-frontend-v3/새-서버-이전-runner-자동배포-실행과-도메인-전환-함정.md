---
tags: [xquare, deploy, nginx, github-actions, self-hosted-runner, 도메인, 보안]
updated: 2026-11-23
---

# xquare 프론트 새 서버 이전: runner 자동 배포 실행과 도메인 전환 함정

[[SSH-게이트웨이-VM-배포-결정과-self-hosted-runner]]에서 정한 설계를 실제로 실행한 기록(2026-11-14~20). 이 작업은 새 사이트 배포가 아니라 **이미 같은 도메인으로 돌던 사이트의 실행 서버를 바꾸는 이사**다. stag는 새 서버 8080, prod는 80으로 옮겼고 둘 다 develop/main push 자동 배포가 됐다. 정적 서빙 원리는 [[Vite-SPA-nginx-정적-서빙과-try_files-캐시]].

## 이사 개념과 순서
- 도메인은 한 번에 한 곳만 가리킨다. "연결해 달라"는 건 두 서버에 같이 붙이는 게 아니라 **앞단 프록시의 도메인→서버 규칙 한 줄을 예전 서버에서 새 서버로 바꾸는 것**이다.
- 순서: ① 새 서버 준비(nginx·runner·워크플로) → ② 실제 빌드를 새 서버에 넣고 `curl localhost:포트`로 확인 → ③ 프록시 규칙 변경 요청 → ④ 예전 서버·예전 자동 배포 내리기.
- ②를 먼저 하는 이유: 빈 방(`stag ok` 테스트 페이지)으로 안내되지 않고, 문제가 생기면 규칙을 예전 서버로 되돌려 즉시 복구할 수 있다. 같은 커밋이면 빌드 해시(`index-xxxx.js`)까지 같아 화면으로는 구분이 안 된다.
- **prod 함정(실제 발생)**: 요청하지 않았는데 담당자가 prod 도메인을 새 서버 80으로 먼저 연결 → prod 폴더엔 nginx 테스트용 `prod ok`만 있어 실서비스가 한 줄 페이지가 됐다. develop→main PR 머지로 첫 prod 배포(약 45초)를 돌려 복구. 교훈: **도메인 연결을 요청하기 전에, 요청하지 않아도 연결될 수 있다고 가정하고 prod 폴더에도 실제 빌드를 먼저 넣어 둔다.**

## 도메인이 새 서버를 가리키는지 확인하는 법
- 빌드가 같아 화면·JS 해시로는 구분 불가 → nginx가 파일마다 붙이는 **`Last-Modified`(ETag)와 `Cache-Control` 헤더**로 대조한다. 새 서버에서 먼저 `curl -sI localhost:8080/`로 값을 기록해 두고 공개 주소 값과 비교.
- 예: 예전 서버는 `/`에 Cache-Control이 없고 새 서버는 `no-cache`. 일치하면 전환 완료.
- 공개 주소의 JS `Last-Modified`가 옛날 날짜인 건 Cloudflare가 예전 서버 때 저장한 사본을 주기 때문이며, 해시가 같으면 내용도 같아 문제없다.
- 예전 stag 자동 배포가 develop push에 계속 반응하는 것도 PR 체크 목록(`infra-frontend-stag` 같은 우리가 안 만든 체크)으로 드러난다. 도메인 이전 뒤 내려 달라고 요청해야 두 곳 이중 배포를 피한다.

## 증상 구분: 521 vs 502 (Cloudflare 뒤)
- **502**: 프록시까진 됐는데 뒤의 백엔드가 응답 없음. **521**: Cloudflare가 뒤의 프록시에 연결 자체를 거부당함(한 단계 앞). 무관한 프로젝트 도메인(openapi)까지 같이 521이면 우리 서버가 아니라 공용 프록시 문제.
- 로그인·데이터가 안 되는 건 백엔드 502일 때 예전/새 서버 상관없이 똑같으므로, 프론트 이사 검증과 분리해서 본다.

## runner 설치 실행 절차와 함정
- **전용 계정**: root로 runner를 돌리면 워크플로가 서버 전체를 건드릴 수 있고 설치 프로그램도 root를 거부한다. `useradd -m -s /bin/bash runner`(비밀번호 없음 → 외부 로그인 불가, sudo 그룹 없음) 후 `chown -R runner:runner /var/www/xquare-prod /var/www/xquare-stag`로 **사이트 폴더 두 개만** 쓰게 한다.
- 프롬프트로 어디에 있는지 구분: `❯` 로컬 PC, `#` 서버 root, `$` runner 계정. 로컬에서 서버용 명령(`useradd`)을 치는 실수가 실제로 있었다(Permission denied로 무사).
- GitHub 화면의 명령을 복사할 때 앞의 `$`를 같이 붙이면 `$: command not found`. 서버에 `shasum`이 없으면 `echo "<sha256>  파일" | sha256sum -c`.
- `./config.sh`가 실행 전 필요한 라이브러리를 스스로 검사하므로 `installdependencies.sh`(apt, DNS 필요)가 DNS 때문에 실패해도 **등록부터 시도해 보고**, `Libicu ... missing`류 에러가 날 때만 DNS를 고친다. DNS(`getent hosts archive.ubuntu.com`)가 간헐적으로 실패하는 서버는 runner가 GitHub와 계속 통신해야 하므로 배포도 가끔 실패할 수 있어 담당자에게 문의한다(핫스팟을 써도 소용없다 — 서버 쪽 네트워크 문제).
- `config.sh` 답: 그룹 Enter, 이름 Enter, **라벨 `xquare-frontend`**, 작업 폴더 Enter. `./run.sh`는 돌리지 않고 root로 `./svc.sh install runner && ./svc.sh start` → `systemctl list-units --no-pager 'actions.runner*'`에 `active running`. `svc.sh`는 `config.sh` 후에 생성된다. 등록 토큰은 1시간 만료(`gh api -X POST repos/<owner>/<repo>/actions/runners/registration-token --jq .token`).
- 워크플로의 deploy job은 `runs-on: [self-hosted, xquare-frontend]`. actionlint는 직접 만든 커스텀 라벨을 모른다고 경고하는데 정상.

## 서버 셸 함정 모음 (이 VM에서 겪은 것)
- 붙여넣은 첫 명령이 `^[[200~`로 깨짐(bracketed paste) → **접속할 때마다** `bind 'set enable-bracketed-paste off'`를 직접 타이핑.
- `systemctl status`의 pager가 화면을 깨면 `q` 후 `reset`, 이후엔 `--no-pager`·`is-active`·`is-enabled` 사용.
- heredoc(`cat > 파일 <<'EOF'`)이 안 끝나면 끝의 `EOF`에 공백이 붙은 것 → `EOF`는 맨 앞 공백 없이 직접 타이핑. 홑따옴표 `'EOF'`는 `$` 치환을 막는다. nano가 깨질 땐 heredoc으로 쓴다.
- zsh에서 반복문 변수 이름을 `path`로 쓰면 PATH와 연동돼 명령어를 못 찾는다. glob(`--include=*.ts`)은 따옴표로 감싼다.

## 워크플로 설계 세부 (PR로 올린 것)
- 빌드 job(`ubuntu-latest`)이 push 브랜치로 모드를 고른다: develop→`build:stag`(`vite build --mode staging`), main→`build`. 산출물은 upload-artifact.
- 가드 둘: **빌드 결과에 그 환경의 주소가 안 들어 있으면 실패**(env 누락 시 `.env`로 조용히 폴백하는 걸 잡음), **산출물이 비어 있으면 파일을 지우기 전에 중단**. 교체 뒤 nginx가 새 `index.html`을 응답하는지 확인.
- 서버는 빌드 결과물만 받으므로 레포 코드를 실행하지 않고 bun/node도 필요 없다.
- 머지 시점 주의: deploy job이 runner가 없으면 멈춰 있고, **머지하는 순간 첫 배포가 시작**되므로 runner 등록 후 머지(약 30초 소요). GitHub 레포 Deployments 탭에 이력이 쌓인다.
- PR 리뷰 지적(CodeRabbit, 미반영): "7일 넘은 옛 파일 삭제" 기준이 *배포에서 빠진 날*이 아니라 *파일 mtime*이라, 9일 전 배포한 A를 계속 서비스하다 B를 올리면 A 전용 JS가 즉시 삭제될 수 있다. 개선안은 직전 배포 파일 목록을 저장해 그 파일 mtime을 갱신한 뒤 7일 규칙 적용. 우리 앱은 JS가 한 파일이고 Cloudflare가 1년 캐시해 영향이 작아 후순위로 둠.

## public 레포 Actions 보안 설정 (self-hosted runner 전제)
| 설정 | 기본(조회 결과) | 변경 | 이유 |
|---|---|---|---|
| Fork PR 워크플로 승인 | 처음 기여자만 | **외부 기여자 전부 승인 필요** | 한 번 머지된 외부인의 두 번째 PR부터 승인 없이 돌아, PR로 워크플로를 바꿔 우리 서버 runner에서 임의 명령 실행 가능 |
| `GITHUB_TOKEN` 권한 | write + PR 승인 허용 | **read, PR 승인 끔** | 서드파티 Action이 침해돼도 레포에 push·PR 승인 못 하게 |
- 위치: Settings → Actions → General. **항목마다 Save 버튼이 따로**라 ① 저장 안 하고 ②로 가면 ①이 사라진다. `gh api`로 `all_external_contributors`·`read`·승인 기능 `false`가 저장됐는지 조회해 확인했다.
- 팀원 PR은 영향 없음. 쓰기가 필요한 워크플로는 그 파일에만 `permissions:`로 연다. 배포 job은 `push`에서만 실행(위 설계 페이지).

## nginx 보강: `index.html`만 no-cache
- 배포가 옛 JS를 지우므로 브라우저가 캐시한 옛 `index.html`이 지워진 JS를 요청하면 흰 화면이 난다([[배포-후-열린-탭-옛-청크-요청이-index-html로-돌아오는-문제]]). `location = /index.html { add_header Cache-Control "no-cache"; }`를 추가하면 매번 서버에 재검증(변동 없으면 304류로 가벼움). `/assets/`는 그대로 1년 immutable.
- 확인: `curl -sI localhost:8080/ | grep -iE 'cache-control|last-modified'`(`no-cache`), 없는 경로 `/some/page`도 `index.html`로 가므로 `no-cache`, `/assets/...js`는 `max-age=31536000`+`public, immutable` 두 줄(`expires`가 한 줄, `add_header`가 한 줄 → 정상). `cp`로 `/root/`에 백업 후 `nginx -t && systemctl reload nginx`(`&&`라 문법 오류면 reload 안 하고 기존 설정 유지).

## 환경변수 파일 커밋 결정
- `.env.staging`·`.env.production`을 커밋하고 `package.json`에 `build:stag` 추가. 근거와 값별 안전성은 [[Vite-빌드타임-환경변수-인라인]]의 "공개값 env 파일 커밋" 절.
- 레포 코드 메모: `index.tsx`는 `import.meta.env.PROD`면 `console.error`를 no-op으로 만드는데 staging 빌드도 PROD라 에러 로그가 사라진다. 배포 상세 로그 컴포넌트(`log.tsx`, `summary.tsx`)의 `localhost:5173`은 하드코딩된 가짜 샘플 로그다(env 유출 아님).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/xquare-infra-frontend-v3, 2026-11-14~20) — raw 파일 없음
