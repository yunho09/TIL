---
tags: [s3, unicode, nfc, nfd, file-upload, jobis]
updated: 2026-11-19
---

# 한글 파일명 NFD/NFC 불일치로 S3 NoSuchKey

맥에서 찍은 스크린샷처럼 **자모가 분리된(NFD) 한글 파일명**을 그대로 업로드하면, S3 키도 NFD로 저장된다. 이를 **완성형(NFC)** 으로 요청하는 클라이언트는 화면상 같은 이름이어도 바이트가 달라 `NoSuchKey`(404)를 받는다. 업로드 직전에 `file.name.normalize("NFC")`로 맞추면 새 파일부터 해결된다.

## 증상과 확인법
- 어드민이 맥 스크린샷을 반려 첨부로 올림 → 앱에서 열면 S3 `NoSuchKey`. 파일은 버킷에 실제로 있음(UUID로 조회하면 존재).
- "스" 하나가 `ㅅ`+`ㅡ` 둘로 저장(NFD, 퍼센트 인코딩 `%E1%84%89%E1%85%B3`) vs 앱 요청 `%EC%8A%A4`(NFC). 같은 경로로 요청하면 **NFD 200, NFC 404**.
- curl 두 번으로 "파일이 존재하는지 / 어느 정규화 형태로 저장됐는지"를 가를 수 있다.
- 화면에 `AccessDenied` 대신 `NoSuchKey`가 뜨는 건 버킷 목록 조회가 열려 있어서(아래 보안 이슈).

## 원인 체인
1. 맥 브라우저가 파일명을 NFD로 넘김.
2. 프론트가 정규화 없이 presign 요청(`packages/api/src/files`), 백엔드도 받은 이름 그대로 S3 키·DB에 저장하고 반환(정규화 코드·필터 없음).
3. 웹은 값을 그대로 URL에 붙여 열리고, 앱 경로 어딘가에서 NFC가 되어 404. iOS 코드는 `appendingPathComponent` 후 `UIApplication.open`(Safari)으로 넘길 뿐 문자열을 바꾸지 않아, 변환 지점은 Safari 이후로 **추정**(소스로 확정 못 함, 실기기 확인 필요).

## 해결과 범위
- **FE 수정(채택, 이슈 #204 → PR #205):** 공용 업로드 훅에서 요청 전 파일명을 `normalize("NFC")`. 반려 첨부·공지·버그 제보·지원서 첨부·기업 로고가 한 곳에서 해결.
- NFC는 다시 NFC로 바꿔도 그대로(멱등)라서 앱·Safari가 바꾸든 안 바꾸든 키와 요청이 일치한다. 그래서 **웹에서 올리는 파일은 앱 코드 수정이 필요 없다**.
- **이미 NFD로 올라간 파일은 FE 수정으로 안 고쳐진다.** 다시 올리거나, 앱이 Safari 대신 `URLSession`+`QLPreviewController`로 직접 받아 열어야 함. 이 앱은 업로드를 하지 않아 앱 수정은 불필요, 깨진 파일만 재업로드.
- 앱이 업로드하는 구조라면 iOS `.precomposedStringWithCanonicalMapping`, Android `Normalizer.normalize(it, Normalizer.Form.NFC)`가 필요.
- 책임 판별: 앱이 받은 API 응답 URL 문자열을 로그로 찍어 이미 NFC면 백엔드가, NFD면 앱/Safari가 바꾼 것.

## 같이 알아둘 것
- 이 파일 버킷은 **인증 없이 목록 조회(ListBucket)가 가능**해 반려·지원서 첨부 파일명이 노출된다. 저장소가 공개라 이슈에는 버킷 주소·목록 공개 문제를 쓰지 않고 백엔드·인프라에 따로 전달.
- 배포 확인: develop 머지 후 stag 빌드가 끝나면 배포된 번들에 `normalize("NFC")`가 있는지 받아서 확인([[배포-반영-확인-번들-해시-대조]]). 운영은 develop→main 머지 뒤에야 반영. 실제 맥 스크린샷 재업로드→앱 열림 확인은 미완.
- 진행 중인 #202 체크아웃을 건드리지 않도록 별도 git worktree에서 작업(머지 후 `git worktree remove`).

## 관련
- [[S3-Presigned-URL-업로드]] · [[JOBIS-FE-V2/프로젝트-현황]] · [[JOBIS-FE-V2/iOS-앱-점검-2026-11-17]]

## 출처
- Claude Code 세션 자동 캡처 (/data/project/JOBIS-FE-V2)
