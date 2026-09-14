---
tags: [frontend, cors, file-download, s3]
updated: 2026-09-14
---

브라우저가 `<a href download>`로 파일을 받을 때, 파일 주소가 페이지와 다른 도메인(S3/CloudFront 등)이면 `download` 속성이 무시되어 새 탭에서 파일이 열리기만 하고, 파일명도 원래 이름이 아니라 저장소 key 그대로 나올 수 있다. 원래 파일명으로 받으려면 프론트가 파일을 직접 fetch해 Blob으로 받아야 하는데, 그러려면 저장소 CORS 설정이 필요하다.

## 증상과 원인

- 같은 도메인 파일은 `<a download="이름.pdf" href="...">`가 그대로 동작해 원래 이름으로 저장된다.
- 파일 도메인이 API/페이지 도메인과 다르면 브라우저가 `download` 속성을 무시한다. 링크를 클릭해도 새 탭에서 파일이 열리거나, 저장은 되어도 파일명이 주소의 key 그대로 나온다.

## 해결 경로 두 가지

1. **저장소가 CORS GET을 허용**하면: 프론트가 `fetch(fileUrl)`로 Blob을 받아 `URL.createObjectURL` + 임시 `<a>`로 원하는 파일명으로 저장할 수 있다.
2. **서버/스토리지가 응답에 `Content-Disposition: attachment; filename=...` 헤더**를 붙이면 브라우저가 그 헤더의 파일명으로 바로 다운로드한다. 이 경우 프론트는 fetch 없이 링크만 걸어도 된다.

둘 중 하나만 있으면 되고, 둘 다 없으면 "새 탭으로 열기"까지만 가능하다(원래 파일명 유지 불가).

## 비공개(private) 버킷인 경우

`baseurl + key`로 주소를 조합해도 인증 없이는 접근이 막힌다(`AccessDenied` XML 응답). 이때는 백엔드가 만료시간이 있는 **서명된 다운로드 URL(presigned URL)**을 내려주는 API를 만들어야 하며, 프론트만으로는 해결할 수 없다. 업로드 쪽의 비슷한 패턴은 [[S3-Presigned-URL-업로드]] 참고.

## 확인 순서

1. 업로드/조회 응답에 파일의 전체 주소(`fileUrl`)가 오는지, key만 오는지 확인 — 주소가 없으면 baseurl을 백엔드에 물어야 한다.
2. 그 주소를 로그인 없이 새 탭에 붙여 넣어 열리는지 확인 — 안 열리면 비공개 버킷, presigned URL이 필요하다.
3. 콘솔에서 `fetch(fileUrl).then(r => console.log(r.status, r.headers.get('content-disposition')))`을 실행해 CORS 통과 여부와 `content-disposition` 헤더를 확인.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop)
