# Wiki Log

append-only 작업 기록. 과거 항목은 수정하지 않는다.

## 2026-09-01 — init
- 생성: [[index]], [[log]]
- 비고: CLAUDE.md 운영 규칙 작성, raw/ 폴더 생성. 위키 시작.

## 2026-09-02 — ingest (spring-practice 학습 세션)
- 원본: spring-practice 프로젝트 Claude Code 세션 (raw/ 아님 — 세션에서 직접 생성)
- 생성: [[스프링-CRUD-계층구조]], [[JPA-엔티티와-리포지토리]], [[MySQL-연동]], [[프로젝트-현황]], [[학습-진행상황]]
- 비고: 처음에 llm-wiki/ 폴더를 중복 생성했다가 기존 wiki/로 통합. 주제별 하위 폴더(언어/Java/Spring, 프로젝트/spring-practice)는 사용자 요청으로 유지.

## 2026-09-04 09:00 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/spring-practice)
- 생성: [[빈과-DI]], [[스프링-컨테이너]] — 기존 페이지들이 참조만 하고 실제로는 없던 링크였음. 다중 구현체 주입(@Primary/@Qualifier/List<T>), 컨테이너=Map 비유, 컴포넌트 스캔 기준점·src 폴더 구조 등 이 세션에만 있던 내용으로 채움
- 갱신: [[스프링-CRUD-계층구조]] — Postman/curl 테스트 방법 섹션 추가
- 비고: 이 세션은 DI 실습(MessageSender 등)부터 CRUD·JPA·MySQL 연동까지 처음부터 다시 다룬 긴 튜터링 세션으로, 대부분 2026-09-02 ingest와 내용이 겹쳐 새 페이지 생성 없이 기존 페이지 보강 위주로 처리함. spring-practice는 세션 종료 시점까지도 git 저장소가 아님(git init 미완료) — [[학습-진행상황]] 참고.

## 2026-09-04 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[위키-자동-캡처]]
- 비고: 이 vault의 LLM Wiki 골격을 만들고, 이어서 SessionEnd 훅으로 세션 종료 시 자동 ingest + git push되는 파이프라인을 구축한 세션. OMC 내장 wiki 스킬 대신 CLAUDE.md+raw/+wiki/ 직접 구조를 택한 이유, 자동 캡처 동작 조건·스킵 규칙·적용 범위(Claude Desktop 일반 채팅은 미지원 등)를 정리함.

## 2026-09-04 14:00 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 갱신: [[위키-자동-캡처]] — "Notion 연동은 왜 안 붙였나" 섹션 추가 (진실의 원천 분리 위험, 위키링크 미지원, 이미 GitHub push가 웹 미러 역할을 하므로 자동 미러링 미채택)
- 비고: 동일 세션(/home/yunho)의 이어진 대화를 재캡처. 이전 캡처(같은 날짜, 위 항목)와 대부분 중복이라 새 페이지 없이 기존 페이지만 보강.

## 2026-09-05 — ingest (Orca IDE 설치 세션)
- 원본: Claude Code 세션 (/home/yunho, 2026-09-02~05) — raw/ 아님
- 생성: [[Orca-IDE-리눅스-설치]]
- 비고: 사용자가 "이거 llm wiki에 있나"로 조회했으나 vault에 Orca 관련 페이지 0건이었음. 해당 대화는 9/2~9/3로 SessionEnd 자동 캡처 구축(9/4) 이전이라 자동 ingest 대상이 아니었다. 사용자 선택에 따라 사용법 레퍼런스(공식 문서와 중복)는 제외하고 재발 가능성이 높은 리눅스 설치 트러블슈팅만 페이지화. wiki/환경/ 하위 폴더와 index의 "환경/리눅스" 섹션을 이번에 신설.

## 2026-09-06 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
- 생성: [[JOBIS-FE-V2/프로젝트-현황]], [[Figma-Dev-Mode-MCP]], [[S3-Presigned-URL-업로드]], [[CSS-Container-shrink-to-fit]]
- 비고: Figma 디자인(학생 지원하기 화면)을 기존 design-system 컴포넌트만으로 퍼블리싱하고 API 연동까지 진행한 세션. JOBIS-FE-V2는 이 vault에 처음 등장하는 프로젝트라 wiki/프로젝트/JOBIS-FE-V2/와 wiki/프론트엔드/ 폴더, index의 해당 섹션을 신설했다. 프로젝트 고유 정보(퍼블리싱·커밋 컨벤션, staging DNS 오지정, packages/api 공용 코드 결함, 백엔드 API 명세 함정)는 프로젝트 페이지로, 다른 프로젝트에도 재사용 가능한 개념(Figma Dev Mode MCP의 데스크톱/view-only 제약, S3 presigned URL 업로드 규칙과 Playwright 검증 한계, CSS shrink-to-fit과 max-width의 한계)은 프론트엔드/ 하위 별도 페이지로 분리했다. 실제 백엔드 서버가 DNS 문제로 접속 불가였던 것, 컴포넌트 인스펙터 값 요청·아이콘 위치 보류 같은 진행 중 판단은 재사용 가치가 낮아 제외.

## 2026-09-06 23:11 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[GNOME-Wayland-wl-clipboard-포커스-토스트]]
- 비고: 터미널 Claude에 이미지를 붙여넣을 때마다 뜨던 정체불명 GNOME 토스트를 진단한 세션. Mutter 50에 `zwlr_data_control`/`ext_data_control` 프로토콜이 없어 `wl-clipboard`가 숨은 `xdg_toplevel`+`xdg_activation_v1` 포커스 요청 폴백을 타는데, dconf에 남아있던 `focus-new-windows=strict`(과거 Claude Desktop 포커스 탈취 대응용이었으나 이미 `--ozone-platform=x11`로 해결되어 잔재였음) 때문에 Mutter가 요청을 거부해 GNOME Shell 토스트로 대신 뜨는 구조였다. `smart`로 복구해 해결. 겸사겸사 3일 넘게 돌던 `ws-monitor.sh` 디버깅 좀비 프로세스도 정리. wiki/환경/ 하위에 페이지 추가.

## 2026-09-06 23:14 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[절전-복귀-지연-원인과-zram-도입]]
- 비고: 절전 복귀가 가끔 9초 넘게 걸리던 문제를 진단한 세션. 커널 자체 복귀는 항상 0.2~0.8초로 빠르고, 디스크 스왑 완전 포화 상태에서 절전 중 놓친 `Persistent=true` 타이머(devlog-sync, snap firmware-updater 등)가 복귀 즉시 몰려 실행되며 메모리를 요구해 gnome-shell 페이지 재적재가 지연되는 구조였다. zram(8G/zstd/prio 100) 도입, `vm.page-cluster=0`/`vm.swappiness=100` 튜닝, firmware-updater 타이머 mask, devlog-sync에 RandomizedDelaySec 추가로 조치. 세션 종료 시점까지 재부팅 전이라 디스크 스왑 잔여분 때문에 효과는 재부팅 후 다음 복귀에서 검증 필요 — 이 미검증 상태를 페이지에 명시함.

## 2026-09-07 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho/.claude/projects/-home-yunho/memory)
- 생성: [[Ptyxis-터치패드-스크롤-속도-패치]]
- 비고: Ptyxis(GTK4+VTE) 터치패드 스크롤이 너무 빠른 문제를 진단·해결한 세션. 원인이 세 겹으로 겹쳐 있었음 — ① GTK4가 `-Bsymbolic`으로 빌드돼 LD_PRELOAD 델타 가로채기 불가, ② `enable-fallback-scrolling=false`라 VTE가 스크롤백 이벤트에 관여하지 않고 GtkScrolledWindow가 무보정 델타로 처리(어제 세션에 만든 패치가 무효했던 이유), ③ 마우스 보고 앱 경로는 휠 노치→줄 단위 이중 양자화로 프로토콜상 부드럽게 만들 수 없음. 특히 `DBusActivatable=true`인 앱은 GNOME이 D-Bus로 `/usr/bin/ptyxis`를 직접 띄워 PATH 래퍼/.desktop Exec를 전부 우회한다는 점은 다른 GNOME 앱 커스터마이징에도 재사용 가능한 함정이라 판단해 페이지화. VTE 패치 확장(스크롤백 직접 처리) + 스크롤백/앱 배율 분리 + 마우스 보고 앱에서 터치패드만 스크롤백으로 우회시키는 bypass로 해결, 0.4/0.05로 사용자 확인 완료. 터치패드 속도 조절 자체보다 원인 진단 체인(D-Bus 활성화 우회, GTK4 심볼 인터포지션 차단, VTE 렌더링 단위)이 재사용 가치가 높아 그 부분 위주로 정리함.

## 2026-09-07 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[Zaemit-공모전/Zaemit-MCP-연동]], [[Claude-Code-MCP-서버-등록]]
- 비고: "제2회 Zaemit AI 웹사이트 공모전" 출품을 위해 재밋 MCP를 연결하던 세션. 재밋 고유 지식(정본 엔드포인트가 `zaemit.ai/mcp`이고 안내에 자주 쓰이는 `mcp.zaemit.ai`는 랜딩페이지라 핸드셰이크가 깨지는 함정, 검색형 게이트웨이 3툴 뒤에 403툴/57그룹이 숨은 구조, 게시판·문의폼이 플러그인 설치형인 점, Free 플랜 한도, 공모전 절대 제약)은 wiki/프로젝트/Zaemit-공모전/에, 다른 프로젝트에도 재사용 가능한 Claude Code 자체의 MCP 등록·인증 동작(세션 부팅 시에만 MCP 목록 로드, Desktop 내장 세션은 OAuth 브라우저 승인 불가, 터미널을 도중에 닫으면 accessToken이 빈 문자열로 저장되며 조용히 실패하는 패턴)은 wiki/도구/에 분리해 페이지화했다. index에 두 섹션(프로젝트/Zaemit-공모전, 도구) 신설. 세션은 OAuth 인증 성공과 툴 카탈로그 확보(3단계 기능 테스트 착수 직전)에서 끝났고, 실제 기능 한계 테스트 결과는 아직 없어 페이지에 미확인 항목으로 남김.

## 2026-09-07 19:37 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
- 생성: [[ToyVillage-Admin-FE/프로젝트-현황]], [[Figma-MCP-팀별-호출-한도]], [[Git-브랜치명-샵-이스케이프]]
- 비고: Figma `업무관리 · 목록` 화면을 Figma 스펙에 맞춰 퍼블리싱하던 세션. 도중에 사용자가 새 스크린샷을 보내와 작업 중이던 구 파일(`toyvillage-dev`)이 폐기되고 `yot` 파일로 전체 화면이 재정리된 사실이 드러나 spec을 재작성하고 게이트를 재승인하는 과정으로 이어졌다. 이 저장소 고유의 harness 퍼블리싱 파이프라인 구조(specs/approvals 분리, 시나리오·e2e 해시로 게이트 판정, ②·⑦ 사람 승인 지점, Figma 파일 전환, 커밋/브랜치 컨벤션, 알려진 e2e·폰트 이슈)는 프로젝트 페이지로 묶었다. 다른 프로젝트에도 재사용 가능한 두 가지 — `claude.ai Figma` MCP의 호출 한도가 계정이 아니라 파일 소유 팀의 플랜 단위로 걸리는 동작과 페이지 전체 덤프가 한도를 급격히 깎는 함정, `#이슈번호/` 브랜치 컨벤션에서 `#`이 쉘 주석으로 잘리는 문제 — 는 wiki/프론트엔드/·wiki/도구/에 각각 일반 지식 페이지로 분리했다. index에 "프로젝트/ToyVillage-Admin-FE" 섹션을 신설했다. 커밋 메시지 형식을 어떻게 다듬을지 주고받은 대화, 개별 커밋 diff 설명, 게이트 재승인 명령을 실제로 실행한 로그, PR·원격 브랜치 정리 같은 이 세션에만 유효한 진행 상태는 재사용 가치가 낮아 제외했다.

## 2026-09-07 21:55 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
- 생성: [[Claude-Code-느낌표-bash-접두사-채팅전용]]
- 갱신: [[ToyVillage-Admin-FE/프로젝트-현황]] — 업무관리 목록 퍼블리싱 완료·커밋(`160faab`)·push 반영, `task-create`/`task-edit`/`task-detail` spec 전면 재작성과 게이트 4종 승인 완료(다음 단계는 ③ 퍼블리싱, 착수 전 세션 종료), `approve.mjs`가 `approvals/`의 파일 내용을 검증 없이 승인해 cp를 빼먹으면 "유효하지만 의미 없는" 통과가 되는 함정, 이미 push된 커밋을 amend하면 force push가 필요해지는 사례 추가
- 비고: 앞선 세션(19:37 ingest)에 이어지는 같은 저장소의 후속 세션. `업무관리 · 목록` 슬라이스를 실제로 커밋·push까지 마쳤고, 이어서 `생성`·`수정`·`상세` 화면이 yot 기준으로 구조 자체가 바뀐다는 걸 확인해 spec 3종을 새로 쓰고 게이트 승인까지 받았다(실제 코드 작업은 다음 세션으로 미룸). 이 저장소 고유 사실은 기존 프로젝트 페이지에 병합했다. 새로 페이지화한 건 이 세션에서 세 번 반복된 사고 패턴 하나 — `!` bash 즉시실행 접두사를 채팅이 아니라 실제 터미널에 붙여넣어 `command not found: !node`로 실패한 것 — 로, 다른 프로젝트 세션에서도 재발할 수 있는 Claude Code 자체의 동작이라 wiki/도구/에 일반 지식으로 분리했다. 팀·직원 mock 명단을 어떻게 채웠는지, 승인 명령을 몇 번 다시 안내했는지 같은 이 세션 한정 진행 로그는 제외했다.

## 2026-09-08 00:01 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 갱신: [[Claude-Code-MCP-서버-등록]] — 세션 부팅 후 등록한 MCP 서버를 못 쓰는 걸 우회하려고 `~/.claude/.credentials.json`을 직접 읽어 HTTP로 호출하는 스크립트를 시도하면(값이 아니라 길이만 확인하는 용도라도) auto mode 권한 분류기가 차단한다는 사실 추가
- 비고: 앞선 2026-09-07 세션에 이어지는 재밋(Zaemit) 공모전 MCP 연동 세션. 엔드포인트 함정·게이트웨이 3종 구조·Free 플랜 한도·OAuth 세션 부팅/Desktop 제약·토큰 교환 실패 패턴은 전날 이미 페이지화된 내용과 대부분 중복이라 새로 페이지화하지 않았다. 이번 세션에서만 나온 새 사실 하나 — 토큰 파일을 직접 읽는 우회 시도가 auto mode에 막히고 결국 새 세션(네이티브 툴 연결)으로 전환해야 했던 것 — 만 기존 페이지에 보강했다. Desktop vs 터미널 세션 혼동, `/mcp` 메뉴 커서 위치 안내, 3단계 기능 테스트를 터미널 세션으로 인계하는 과정 등은 이 세션에만 유효한 진행 상태라 제외했다.

## 2026-09-08 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
- 갱신: [[JOBIS-FE-V2/프로젝트-현황]] — 이슈/브랜치/PR "각각 1개가 기본값"이라는 작업 관리 컨벤션 추가(9이슈/8PR로 쪼갰다가 정정받아 통합한 사례), `createIdMutationHook` 더블슬래시 결함을 수정 완료로 갱신, `updateParams`(kebab-case 변환)와 `getParam`/`loader`(원래 키로 읽음) 표기 불일치로 상태·기술스택 필터 둘 다 죽어있던 버그 클래스를 신규 섹션으로 추가, 백엔드 문답으로 확정된 도메인 지식 8건(모집의뢰서 status enum, acceptances DELETE=취소 의미, 근로계약 변경 API, 조회수 POST /views 필요, 공지 수정 첨부파일 미지원, 취업관리 전체조회 API 부재, banner_url 폐기와 배너 하드코딩 유지 결정, student_gcn 필드) 추가, DS 컴포넌트 로컬 vitest 불가(Playwright chromium 미설치) 인프라 이슈 추가, FileUpload 컴포넌트 신설 사실 반영
- 비고: 어드민/스튜던트 미연동 화면 9개를 몰아서 처리한 세션. 화면별 담당자 배정, 각 이슈·PR 번호, Figma 노드 링크 탐색 과정, 디자이너·백엔드에게 보낼 문의 메시지 초안, "co-authored-by 제거" 같은 세션 한정 대화는 재사용 가치가 낮아 제외했고, 그중 실제로 다시 쓸 만한 결정(작업 관리 방식 정정)과 코드/도메인 사실(버그 원인, API 의미 확정)만 추출해 기존 프로젝트 페이지에 병합했다. 새 페이지는 만들지 않음.

## 2026-09-08 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/Commonly-fe)
- 생성: [[Commonly-FE/프로젝트-현황]], [[Vite-빌드타임-환경변수-인라인]], [[도로명주소-검색-API-신청]]
- 비고: Commonly-fe가 이 vault에 처음 등장하는 프로젝트라 wiki/프로젝트/Commonly-FE/ 폴더와 index의 해당 섹션을 신설했다. FE 소스와 방금 fetch한 BE(`cb594fd`) 코드를 직접 대조해 얻은 FE↔BE 갭 목록(개별등록/민원인 발급이 막힌 이유, 대량등록 데이터가 `humanId=null`로 저장되어 영구히 조회·발급 불가능한 백엔드 버그, JWT에 이름/역할 클레임이 없는 문제)과 이번에 처리한 이슈·PR(#68/#70 발급결과 새로고침 복구, #69/#71 대상자 삭제, #72/#73 juso 키 문서화)을 프로젝트 페이지로 정리했다. 다른 프로젝트에도 재사용 가능한 두 개념 — Vite `VITE_*`가 빌드타임에 번들로 치환되는 메커니즘, business.juso.go.kr 도로명주소 API 신청 절차(개편된 SPA 경로 포함) — 는 wiki/프론트엔드/ 하위 별도 페이지로 분리했다. 이슈 우선순위 논의(어떤 항목부터 파고들지 고른 과정), AskUserQuestion 선택지 문구, curl 검증 원문 로그, 커밋 트레일러를 뺄지 말지 같은 세션 한정 대화는 제외했다.

## 2026-09-08 08:30 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 갱신: [[Zaemit-공모전/Zaemit-MCP-연동]] — OAuth 인증 최종 성공 확인(accessToken 753자, scope `mcp pii newsite`)과 이에 따라 공모전 절대 제약 "연동 이력 1회 이상" 충족 사실 추가
- 갱신: [[Claude-Code-MCP-서버-등록]] — `/mcp` 메뉴에서 커서가 새로 등록한 서버가 아닌 다른 항목에 남아 있어 그대로 Enter를 누르면 엉뚱한 서버 메뉴로 들어가는 함정 추가
- 비고: 이전(2026-09-07, 2026-09-08 00:01) 캡처와 거의 같은 재밋 MCP 연동 세션의 재캡처. 엔드포인트 함정·게이트웨이 구조·Free 플랜 한도·세션 부팅/Desktop 제약·토큰 교환 실패 패턴은 이미 페이지화되어 중복이라 새로 반영하지 않았고, 이번 세션에서만 확인된 두 가지(OAuth 최종 성공과 그 scope, `/mcp` 커서 위치 함정)만 기존 페이지에 보강했다. 3단계 기능 테스트(멀티 페이지·이미지·CSS·폼/게시판·반응형·발행 URL)는 터미널 세션으로 인계된 채 아직 결과가 없어 반영하지 않음. Desktop vs 터미널 세션 혼동 설명, 토큰 파일 위치를 다시 훑어본 과정 등 세션 한정 진행 상태는 제외.

## 2026-09-08 15:39 — ingest (Claude Code 세션)
- 원본: Claude Code 세션 (Windows PC + Raspberry Pi 4). 수업 자료 `heartcom/Linux-Program` 2번 PPT(Rpi_GPIO_DHT11_PyQt) 실습.
- 생성: [[Qt-Designer-PyQt5-연결]], [[라즈베리파이-PyQt5-설치-ARM64]], [[라즈베리파이-GUI-SSH-VNC-실행]], [[gpiozero-GPIO-배선-확인]]
- 비고: Qt Designer로 `.ui`를 그려 `pyuic5`로 변환하고 라즈베리파이 GPIO에 붙여 LED를 제어하는 실습 세션. 재사용 가치가 있는 네 덩어리를 분리해 페이지화했다 — (1) Designer↔코드 연결에서 `objectName`이 유일한 연결고리이고 폼 objectName을 잘못 바꾸면 `Ui_` 클래스명이 통째로 바뀌는 함정(실제로 `<class>LED1_Button</class>`이 되어 있었다), (2) PyQt5의 ARM64 pip 휠 부재로 apt + `venv --system-site-packages`를 써야 하는 점, (3) GUI가 SSH에서 못 뜨는 이유와 코드가 import 시점에 `QT_QPA_PLATFORM`을 덮어써 셸 환경변수 우회가 통하지 않던 함정, (4) gpiozero가 LED 미연결 시에도 에러를 내지 않아 핀 스윕으로 실제 배선(자료는 GPIO13/19/26, 실제는 17/27/22)을 찾아야 했던 과정. 수업 자료가 Pi 5 기준이라 Pi 4에서 달라지는 지점(RPi.GPIO 가용 여부, LGPIOFactory 불필요, 팬 제어 오버레이 무관)도 관련 페이지에 반영했다. 파일 전송 절차, VNC 뷰어 설치 안내, PowerShell `&&` 미지원 같은 세션 한정 진행 상태와 일반 상식은 제외했다.
- 비고: 이 세션 시작 시 로컬이 origin/main보다 21커밋 뒤처져 있었고 `.obsidian/workspace.json` 로컬 수정 때문에 pull이 막혀 있었다. 백업 후 되돌려 pull 완료. 해당 파일은 이번에 받은 커밋의 `.gitignore`에 이미 등록돼 있으나 아직 추적 중이라 재발 가능 (아래 참고).

## 2026-09-08 16:10 — ingest (Claude Code 세션)
- 원본: Claude Code 세션 (Windows PC + Raspberry Pi 4). 앞선 15:39 항목에 이어지는 같은 실습 세션.
- 생성: [[라즈베리파이-GUI-개발-워크플로]], [[gpiozero-PWMLED-밝기-제어]]
- 갱신: [[gpiozero-GPIO-배선-확인]] — `lgpio.error: 'GPIO busy'`가 같은 핀을 쓰는 프로그램 중복 실행 때문이라는 점과 `ps`/`pkill`로 잔여 프로세스를 정리하는 방법 추가
- 갱신: [[라즈베리파이-GUI-SSH-VNC-실행]] — 디스플레이 유무와 별개로 PC 터미널과 파이 터미널을 프롬프트로 구분해야 한다는 섹션 추가(`scp`를 파이 창에서 실행해 `$env:USERPROFILE`가 문자로 처리된 사례, PowerShell 5.1의 `&&` 미지원)
- 갱신: [[Qt-Designer-PyQt5-연결]] — 반복문에서 시그널 연결 시 람다 늦은 바인딩을 기본 인자로 캡처해야 하는 점, 값 표시(valueChanged)와 하드웨어 적용(Send)을 분리하는 패턴 추가
- 비고: 세로 슬라이더 3개(R/G/B) + SEND 버튼으로 RGB 밝기를 조절하는 조명 제어 GUI를 Designer로 만들어 파이에서 동작 확인한 세션. 새로 나온 두 개념 — PWMLED 조광과 전체 개발 워크플로 — 를 페이지로 분리하고, 실제로 부딪힌 세 가지 실패(GPIO busy, PC/파이 터미널 혼동, 람다 늦은 바인딩)는 기존 페이지에 보강했다. 워크플로 페이지는 "어떻게 만들었는지"를 한눈에 보도록 기존 페이지들을 잇는 허브 역할을 겸한다. Flask로 LED를 제어하는 웹 서버(수업 자료 3번 PPT)도 작성했으나 파이에서 실행하지 않아 검증되지 않았으므로 페이지화하지 않았다. 파일 전송 명령 오타, 비밀번호 입력 실패(한글 IME 추정) 같은 세션 한정 사건은 제외했다.

## 2026-09-08 16:45 — ingest (Claude Code 세션)
- 원본: Claude Code 세션 (Windows PC + Raspberry Pi 4). 앞선 15:39 / 16:10 항목과 같은 실습 세션의 개념 설명 구간.
- 생성: [[GPIO-기초]], [[가짜-모듈로-하드웨어-없이-테스트]]
- 갱신: [[Qt-Designer-PyQt5-연결]] — pyuic5가 .ui(XML)를 파이썬 코드로 번역하는 실제 모습(objectName이 그대로 속성명이 됨), `-x` 옵션의 역할, 생성된 .py를 직접 고치면 재변환 때 덮어써진다는 주의 추가
- 갱신: [[라즈베리파이-GUI-SSH-VNC-실행]] — SSH·SCP·VNC가 각각 무엇인지 구분하는 표와, VNC는 저장 공간이 아니라 화면이며 파일은 파이 디스크에만 있다는 점, VNC 창을 닫아도 프로그램은 계속 돌아 `GPIO busy`의 원인이 된다는 점 추가
- 갱신: [[gpiozero-GPIO-배선-확인]] — 프로그램 동시 실행 자체는 가능하고 배타적인 것은 핀·포트 단위라는 점, 같은 핀을 공유하려면 한 프로그램이 소유하고 나머지가 요청하는 구조로 가야 한다는 점 추가
- 갱신: [[라즈베리파이-GUI-개발-워크플로]] — 가짜 모듈 예제 코드를 전용 페이지 [[가짜-모듈로-하드웨어-없이-테스트]] 링크로 정리(중복 제거)
- 비고: 실습 자체보다 "그게 무엇이고 왜 그렇게 했는지"를 되짚은 구간에서 나온 내용. GPIO의 정의·PWM 원리·핀 번호 두 체계, sys.modules 주입으로 하드웨어 없이 검증하는 기법, SSH/SCP/VNC의 역할 구분 등 기존 페이지에 사실만 있고 개념 설명이 없던 부분을 채웠다. DHT11 온습도 센서를 조명 GUI에 추가하는 작업도 같은 세션에서 진행했으나, 라즈베리파이에서 아직 실행·검증하지 않아 사용자 요청에 따라 이번 ingest에서 제외했다. 파일 전송 명령을 잘못된 터미널에서 실행한 사건 등은 이미 15:39·16:10 항목에서 페이지화되어 중복이라 다시 다루지 않았다.

## 2026-09-08 17:20 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
- 생성: [[CSS-fieldset-legend-flex-패딩-무시]]
- 갱신: [[ToyVillage-Admin-FE/프로젝트-현황]] — task-create/task-edit/task-detail 퍼블리싱·게이트 승인·커밋 완료로 갱신(담당자 모델 전환 커밋 3개가 개별로는 안 빌드된다는 트레이드오프 포함), 퍼블리싱 후 이어진 UX 결정 4건(우선순위 pill accent 색, 수정 화면 담당자 트리 초기 전체 접힘, 제목/상세 필드 포커스 링 제거, 업무 보고 정렬 유지)과 업무 상태 모델 변경(반려 제거 → 완료기한 기준 파생 `지연` 상태, 목록 10행·지연 탭 추가) 반영, task-report spec S16 재승인 미결 사항 기록
- 비고: `/publishing` 스킬로 업무지시 생성·수정·상세 세 feature를 퍼블리싱하고 이어서 사용자 피드백(레이아웃 깨짐 스크린샷, 선택색 변경, Figma 재수정 반영, 담당자 트리 접힘, 포커스 제거, 리스트 10행, 반려→지연 상태, 지연 탭)에 따라 반복 수정한 긴 작업 세션. 대부분은 이 프로젝트 한정 일회성 구현 지시라 제외했고, 재사용 가치가 있는 CSS 버그(fieldset/legend, 반응형 그리드, 한글 줄바꿈)만 새 페이지로 분리했다. 나머지는 프로젝트 현재 상태와 결정 근거로서 의미가 있는 부분만 프로젝트 페이지에 요약 반영했다. 게이트 재승인 절차 세부(scenarioHash/e2eHash 메커니즘)는 기존 페이지에 이미 있어 중복 반영하지 않았다.

## 2026-09-08 19:10 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/Commonly-fe)
- 갱신: [[Commonly-FE/프로젝트-현황]] — CodeRabbit 리뷰 코멘트 4건 대응 결과 추가: #71 삭제 버튼 `disabled` 조건이 락 변수(`deletingApplicantId !== ""`) 대신 행 id와 비교하고 있어 동시 삭제 진행 중 다른 행 클릭이 조용히 씹히던 버그 수정(`eb44f8c`), #70 `normalizeCertificateDetail`이 빈 `documentNo`를 그대로 통과시키던 버그를 `.trim() === ""` 거부 조건으로 수정(`55f3f8b`), #73 juso 오탐(dev 키 본인인증 불필요 주장에 대한 CodeRabbit 웹검색 지적)을 실제 발급 근거로 반박해 CodeRabbit이 검토의견 철회
- 비고: 3개 PR(#70/#71/#73)의 CodeRabbit 리뷰 코멘트를 확인·대응·재검증한 세션. 두 코드 버그(동시성 disabled 스코프, 빈 문자열 미검증)는 다른 프로젝트에서도 재발 가능한 패턴이라 교훈 형태로 프로젝트 페이지에 남겼다. PR 자체는 세션 종료 시점까지 머지하지 않고 OPEN으로 남음 — 리뷰 코멘트 답변 문구, 스레드 resolve를 몇 번 확인했는지 같은 진행 로그, "닫았냐"는 질문에 리뷰/머지 중 무엇을 뜻하는지 되짚은 대화는 재사용 가치가 없어 제외했다. 새 페이지는 만들지 않음.

## 2026-09-08 19:12 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/ToyVillage-Admin-FE)
- 갱신: [[ToyVillage-Admin-FE/프로젝트-현황]] — 업무 목록(`TaskListPage.tsx`) 필터·페이지네이션 구현 패턴 섹션 신설: 탭 문자열→상태 룩업 테이블, `useMemo` 3단 파이프라인(allTasks→items→filtered→tasks), `today`를 한 번만 잡아 필터·표에 공유, 탭 전환 시 `useEffect` 대신 렌더 중 상태 보정으로 페이지를 1로 리셋하는 패턴, `YYYY-MM-DD` 문자열 비교
- 비고: 필터 로직 설명 질문과, Figma 담당자 필드 위치 변경(`/publishing` 스킬, 커밋 `242faab`) 두 건이 있던 세션. 후자는 이미 프로젝트 페이지에 기록된 "task-create/edit 퍼블리싱 완료" 상태에 속하는 일회성 구현 지시라 새로 반영하지 않았다(검증 순서를 시각적 순서에 맞추는 결정도 기존 spec 결정 사항의 재적용일 뿐 새 지식 아님). 전자는 아직 문서화되지 않았던 상태 파생·필터·페이지네이션 아키텍처 패턴이라 프로젝트 페이지에 추가했다. 새 페이지는 만들지 않음.

## 2026-09-08 19:35 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/data/project/Commonly-fe)
- 생성: [[401을-세션만료로-오인한-로그인-루프]], [[Vite-모노레포-워크스페이스-패키지-dev서버-캐시]]
- 갱신: [[Commonly-FE/프로젝트-현황]] — 로그인 무한 루프 버그(이슈 #74/PR #75) 섹션 추가: 민원인 계정으로 `/api/certificates/self` 등을 호출하면 백엔드가 권한 부족에도 403 대신 401(빈 본문)을 준다는 사실을 실계정으로 재확인했고, FE가 401을 무조건 세션 만료로 처리해 로그인 직후 화면이 스스로 로그아웃을 유발하던 것을 "토큰 유효성 우선 확인"으로 수정. 로컬 재현 안 됨 제보는 dev 서버가 `packages/utils` 옛 캐시를 서빙한 것으로 판명.
- 비고: 사용자가 실제로 겪은 로그인 루프 버그를 진단·수정·이슈화(#74)·PR화(#75)까지 마친 세션. 두 가지 재사용 가치가 높은 일반 개념 — (1) 백엔드가 인가 실패를 401로 잘못 내려줄 때 프론트가 이를 세션 만료와 혼동해 로그인 루프에 빠지는 패턴과 해결 원칙, (2) Vite 모노레포에서 앱 밖 워크스페이스 패키지 수정이 HMR을 안 타 dev 서버가 옛 변환 캐시를 계속 서빙하는 문제 — 는 wiki/프론트엔드/에 별도 페이지로 분리했다. 이 프로젝트 고유의 근본 원인 세부(어떤 엔드포인트가 왜 막혀 있는지, 이슈/PR 번호)는 기존 프로젝트 페이지에 병합했다. 커밋 트레일러를 넣을지 뺄지 논의, "백엔드 이슈도 올려줄까요" 같은 세션 중 제안·확인 대화는 제외했다.

## 2026-09-08 19:47 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[GNOME-오버뷰-창-미리보기-사라짐]]
- 비고: 3손가락 스와이프로 GNOME 오버뷰를 열면 가끔 창 미리보기 영역만 비는 증상을 조사한 세션. 처음 세운 가설(`_gestureEnd` 예외로 `gestureInProgress`가 고착된다)은 익스텐션에 넣은 상태값 진단 로그로 명확히 반증됐다 — 증상 재현 시점에도 상태 기계는 매번 정상 종료하고 있었다. 원인은 세션 종료 시점까지 미확정. 재현 빈도가 낮아 수동 덤프 대신 익스텐션이 스스로 이상을 감지해 액터 트리를 자동 기록하고 강제 relayout으로 복구를 시도하는 진단 도구를 설치해 다음 재현을 기다리는 상태로 페이지화했다 — 다음 조사가 반증된 가설을 반복하지 않고 실제 덤프부터 보게 하기 위함. 별도로 이 과정에서 반복 확인된 GNOME 일반 지식(확장 프로그램은 코드 변경을 핫로드하지 않아 로그아웃/로그인이 필요하고 `ReloadExtension` D-Bus는 deprecated로 막혀 있다는 것)도 같은 페이지에 남겼다. apt hold된 gnome-shell/mutter 패키지, 별개인 workspaceAnimation.js 크래시는 손대지 않고 참고용으로만 기록. 스크린샷 두 장을 주고받으며 타임스탬프를 맞추던 과정, 최초 가설이 스크린샷과 안 맞아 재검토한 시행착오 자체는 결론(반증)만 남기고 제외했다.

## 2026-09-08 22:18 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 생성: [[Zaemit-공모전/Zaemit-사이트-제작-트러블슈팅]]
- 갱신: [[Zaemit-공모전/Zaemit-MCP-연동]] — 인증 이후 실제 사이트 제작 단계로 넘어갔음을 반영하고 새 트러블슈팅 페이지로 링크
- 비고: 재밋 MCP로 공모전 출품 사이트(yunho)의 히어로·CASE 사진 교체, CTA 장식 제거, 게시판 "최근 기록" 노출 개수 수정, 가로 스크롤 수정 등 겉모습 마무리 작업을 이어간 세션. 개별 교체 내용(어떤 사진을 어떤 사진으로 바꿨는지, CASE 패널 문구, 페이지 시퀀스 번호 등)은 이 사이트에 한정된 일회성 편집이라 제외했다. 대신 다른 재밋 작업에도 재현 가능한 기술적 원인·해결 6가지 — board_embed의 data-limit이 루트 엘리먼트에서만 읽히는 버그, 브라우저 렌더 기반 검수가 필요한 이유, 툴 인자 한글 리터럴 처리, overflow:hidden+position:relative 클리핑 조합, image_import로 외부 이미지 만료 방지(및 워터마크 육안 검수 필요성), 반투명 카드/밝은 배경 가독성 결함 패턴 — 만 추출해 새 페이지로 남겼다. "이쁘게 개선할 부분 없나" 같은 반복 질문, 8장의 이미지 후보를 비교한 과정 자체, 인수인계 메모리 갱신 등은 세션 한정이라 제외.

## 2026-09-08 22:40 — 재편 (폴더 구조 세분화)
- 갱신: `wiki/index.md` — 섹션을 실제 폴더 경로와 일치하도록 재구성. 페이지 요약 문구는 그대로 유지(38개 전부 보존, 유실 0).
- 갱신: `CLAUDE.md` — "wiki/ 하위 분류" 표 추가. 새 페이지는 반드시 하위 폴더에 넣고, 맞는 칸이 없으면 새 폴더를 만들며 index 섹션도 함께 갱신하도록 규칙화.
- 이동: `환경/`에 라즈베리파이·GNOME·데스크톱 앱·전원이 뒤섞여 있어 분리.
  - `라즈베리파이/GPIO/` ← GPIO-기초, gpiozero-GPIO-배선-확인, gpiozero-PWMLED-밝기-제어
  - `라즈베리파이/PyQt5-GUI/` ← 라즈베리파이-PyQt5-설치-ARM64, 라즈베리파이-GUI-SSH-VNC-실행, 라즈베리파이-GUI-개발-워크플로, Qt-Designer-PyQt5-연결(도구/에서), 가짜-모듈로-하드웨어-없이-테스트(도구/에서)
  - `환경/GNOME/` ← GNOME-오버뷰-창-미리보기-사라짐, GNOME-Wayland-wl-clipboard-포커스-토스트
  - `환경/리눅스-데스크톱/` ← 절전-복귀-지연-원인과-zram-도입, Ptyxis-터치패드-스크롤-속도-패치, Orca-IDE-리눅스-설치
  - `메타/` ← 위키-자동-캡처
  - `프론트엔드/CSS/`, `프론트엔드/빌드도구/`, `프론트엔드/디자인-연동/`, `프론트엔드/API-인증/` — 평평하던 11개 분할
- 비고: 라즈베리파이는 "내 PC 환경"과 성격이 달라 `환경/` 하위가 아닌 최상위로 승격했다(사용자 결정). 모든 이동은 `git mv`로 이력 보존. 위키링크는 경로 없는 `[[페이지명]]` 형태라 Obsidian이 파일명으로 해석하므로 이동해도 깨지지 않는다 — 경로가 붙은 링크는 `프로젝트/` 계열뿐이고 그쪽은 건드리지 않았다. `프로젝트/`, `언어/`, `도구/`는 이미 분류가 서 있어 유지.
- 비고: 이 작업 전 vault가 미해결 머지 충돌 상태(`UU wiki/index.md`, `UU wiki/log.md`)였다. 양쪽 모두 추가만 있었으므로 둘 다 살려 해소했고, log.md는 시각 순서(origin의 15:39~16:45 → 로컬의 17:20~22:18)로 정렬했다.

## 2026-09-08 22:44 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho/.local/share/gnome-shell/extensions/Vitals@CoreCoding.com)
- 생성: [[Vitals-확장-상단바-시스템-모니터]], [[리눅스-메모리-점유-앱별-진단-PSS]], [[Figma-데스크톱-앱-메모리-중복]]
- 갱신: [[절전-복귀-지연-원인과-zram-도입]] — Figma가 메모리를 많이 먹는 구체적 원인을 다룬 새 페이지로 링크 추가
- 비고: 상단바에 CPU/메모리를 띄우고 싶다는 요청으로 시작해 Vitals 확장을 활성화·설정하고, 이어서 "왜 메모리를 이렇게 많이 먹지"를 진단한 세션. 확장 자체 스키마는 `gsettings --schemadir`로 설치 경로의 schemas를 직접 지정해야 한다는 점과 GNOME 50의 D-Bus 스크린샷 차단(`AccessDenied`)은 다른 확장 작업에도 재현될 일반 사실이라 남겼다. 메모리 진단은 RSS 대신 PSS로 합산해야 하는 이유와 zram/Shmem/slab을 더해야 총량이 맞는다는 방법론을 별도 페이지로, Figma 데스크톱 앱(`figma-linux-next`)이 비공식 Electron 래퍼라 Chromium 런타임을 중복으로 띄우는 원인과 탭을 자동 해제하지 않는 특성, `settings.json`을 열린 탭 목록으로 오인하면 안 되는 함정은 별도 페이지로 남겼다. 세션이 진행 중이던 시점의 구체적인 수치(그 순간 몇 GB를 먹고 있었는지, 어떤 특정 파일 탭 7개가 열려 있었는지)와 "어느 걸 지금 정리할지" 확인을 구하며 끝난 미결 대화는 스냅샷이라 제외했다.

## 2026-09-08 22:48 — ingest (Claude Code 세션 자동 캡처)
- 원본: Claude Code 세션 자동 캡처 (/home/yunho)
- 갱신: [[절전-복귀-지연-원인과-zram-도입]] — zram 도입 후에도 절전 복귀 지연이 재발해 재조사한 내용 추가. `resume-latency` 측정 스크립트로 5일 23회 표본 확보(느림 4/23), 스왑 고갈은 이미 해소돼 있었고 남은 원인은 ① 복귀 직후 `Persistent=true` 타이머 폭주로 인한 CPU 경합(libinput 랙 로그로 확인), ② gnome-shell 자신이 zram으로 내보낸 페이지(최대 89MB)를 `page-cluster=0`(readahead 없음) 상태로 재적재하는 것. `CPUWeight`가 같은 부모 슬라이스의 형제끼리만 경쟁한다는 사실을 놓쳐 1차 드롭인(배치 서비스에 `Nice=19`+`CPUWeight=1`, `system.slice` 소속)이 `user.slice`의 gnome-shell엔 무효했던 정정 사항 포함. 최종 6가지 조치(MemoryMin 체인, 최상위 `background.slice` 신설, NVMe `none`→`mq-deadline`, 타이머 9개 `Persistent=false`, fstrim 스케줄 이동, gnome-shell `CPUWeight=1000`)와 되돌리기 스크립트(`resume-fix-revert`)를 반영. 세션 종료 시점까지 미검증 상태임을 명시.
- 비고: 사용자가 "다른거 할 수 잇는거 다 해봐"로 포괄적 조치를 요청한 세션. 앞선(2026-09-06) 진단이 절반만 맞았고(스왑 조치는 유효했으나 원인이 더 있었음) `CPUWeight` cgroup 계층 함정처럼 자기 정정이 있었던 세션이라, 최종 결론뿐 아니라 정정 과정 자체도 재사용 가치가 있어 함께 남겼다. 세부 로그 수치(각 복귀 시각별 CPU 점유 초 단위 등)와 pkexec 인증창 진행 과정은 제외.
