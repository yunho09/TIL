---
tags: [frontend, cors, file-download, s3]
updated: 2026-11-19
---

브라우저가 `<a href download>`로 파일을 받을 때, 파일 주소가 페이지와 다른 도메인(S3/CloudFront 등)이면 `download` 속성이 무시되어 새 탭에서 파일이 열리기만 하고, 파일명도 원래 이름이 아니라 저장소 key 그대로 나올 수 있다. 원래 파일명으로 받으려면 프론트가 파일을 직접 fetch해 Blob으로 받아야 하는데, 그러려면 저장소 CORS 설정이 필요하다.

## 증상과 원인

- 같은 도메인 파일은 `<a download="이름.pdf" href="...">`가 그대로 동작해 원래 이름으로 저장된다.
- 파일 도메인이 API/페이지 도메인과 다르면 브라우저가 `download` 속성을 무시한다. 링크를 클릭해도 새 탭에서 파일이 열리거나, 저장은 되어도 파일명이 주소의 key 그대로 나온다.

## 해결 경로 두 가지

1. **저장소가 CORS GET을 허용**하면: 프론트가 `fetch(fileUrl)`로 Blob을 받아 `URL.createObjectURL` + 임시 `<a>`로 원하는 파일명으로 저장할 수 있다.
2. **서버/스토리지가 응답에 `Content-Disposition: attachment; filename=...` 헤더**를 붙이면 브라우저가 그 헤더의 파일명으로 바로 다운로드한다. 이 경우 프론트는 fetch 없이 링크만 걸어도 된다.

둘 중 하나만 있으면 되고, 둘 다 없으면 "새 탭으로 열기"까지만 가능하다(원래 파일명 유지 불가).

**1번을 구현할 때 주의할 점**: 이때 쓰는 `fetch`는 공용 인증 axios 인스턴스가 아니라 브라우저 내장 `fetch`를 그대로 써야 한다. 인증 인터셉터가 붙은 인스턴스로 외부 도메인(CDN 등)을 부르면 토큰이 새고, 그 서버의 403이 세션 만료로 오인될 수 있다 — 자세한 이유는 [[공용-인증-axios-외부도메인-토큰유출]] 참고.

**Blob URL 해제 시점**: `link.click()` 직후 바로 `URL.revokeObjectURL(url)`을 부르면 일부 브라우저(특히 Safari/WebKit)에서 다운로드가 시작되기 전에 blob 참조를 놓쳐 `WebKitBlobResource error 1`로 실패할 수 있다. `setTimeout(() => URL.revokeObjectURL(url), 1_000)`처럼 해제를 늦추면 안전하다. Chromium은 클릭 시점에 blob 참조를 잡아두므로 즉시 해제해도 괜찮지만, 대상 브라우저에 Safari가 있으면 지연 해제가 필요하다.

## 사례: JOBIS-FE-V2 첨부 "다운로드가 웹뷰로만 열림" (2026-11-19)

- **증상**: 다운로드 버튼을 눌러도 저장 대신 PDF 뷰어로 열림. 어드민은 새 탭, 학생 지원하기의 파일 칩은 **같은 탭**으로 열려 작성 중이던 화면을 벗어나는 부작용까지 있었다.
- **원인**: S3 주소에 `<a download>`를 걸었는데 출처가 달라 `download`가 무시되고, S3 응답이 `Content-Disposition` 없이 `application/pdf`라 브라우저 내장 뷰어가 열렸다. 같은 패턴이 여러 곳(공지 상세, `FileDownload` 컴포넌트 등)에 흩어져 있었다.
- **해결**: 디자인 시스템(공용 패키지)에 `downloadFile` 헬퍼를 두고 전부 재사용 — `fetch`로 Blob을 받아 원래 파일명으로 저장하고, **받기 실패 시 새 탭으로 열기**로 폴백. 전제는 S3 CORS가 모든 출처의 GET을 허용(해결 경로 1). 한 곳씩 고치지 않고 헬퍼로 묶은 것이 포인트.
- **테스트**: Blob URL 해제 지연을 0으로 바꾸면 새 테스트가 실패하는지 확인해 테스트가 회귀를 실제로 잡는지 검증했고, 실제 S3 파일을 Chromium으로 받아 원래 파일명 저장·화면 이동 없음까지 확인했다.
- **한계(프론트 우회의 비용)**: ① 파일을 전부 받은 뒤 저장되므로 큰 파일은 클릭 후 잠깐 무반응, ② 폴백 새 탭은 받는 데 오래 걸렸으면 팝업 차단될 수 있고 사용자 안내가 없다. 둘 다 **백엔드가 업로드 시 `Content-Disposition: attachment`를 붙이면(해결 경로 2) 우회 없이 해결**된다 — 백엔드에 전달할 항목.
- 같은 PR에서 함께 발견: 기업 상세 첨부가 파일 주소 앞의 S3 base URL(`FILE_URL`)이 빠졌고 표시 파일명에 uuid만 남고 확장자가 없었다 — 아래 확인 순서 1번(전체 주소인지 key인지)에 해당. 프로젝트 맥락은 [[JOBIS-FE-V2/프로젝트-현황]].

## 비공개(private) 버킷인 경우

`baseurl + key`로 주소를 조합해도 인증 없이는 접근이 막힌다(`AccessDenied` XML 응답). 이때는 백엔드가 만료시간이 있는 **서명된 다운로드 URL(presigned URL)**을 내려주는 API를 만들어야 하며, 프론트만으로는 해결할 수 없다. 업로드 쪽의 비슷한 패턴은 [[S3-Presigned-URL-업로드]] 참고.

## 확인 순서

1. 업로드/조회 응답에 파일의 전체 주소(`fileUrl`)가 오는지, key만 오는지 확인 — 주소가 없으면 baseurl을 백엔드에 물어야 한다.
2. 그 주소를 로그인 없이 새 탭에 붙여 넣어 열리는지 확인 — 안 열리면 비공개 버킷, presigned URL이 필요하다.
3. 콘솔에서 `fetch(fileUrl).then(r => console.log(r.status, r.headers.get('content-disposition')))`을 실행해 CORS 통과 여부와 `content-disposition` 헤더를 확인.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop) — 2026-09-14(원인 진단), 2026-09-16(실제 구현·스테이징에서 pptx/pdf/png 바이트 일치 검증 완료, Blob 해제 시점·인증 인스턴스 재사용 문제를 CodeRabbit 리뷰로 발견·수정)
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2) — 2026-11-19(JOBIS 사례: `downloadFile` 헬퍼·새 탭 폴백·한계)
