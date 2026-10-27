---
tags: [curl, 쉘, API문서, bash]
updated: 2026-10-21
---

# curl API 호출 예시: 쉘 인용 규칙

API 문서의 "토큰 발급 → 토큰으로 호출" curl 예시를 쓸 때 인용 방식이 왜 그렇게 정해지는지 정리한다. 로컬에 요청을 그대로 출력하는 임시 서버를 띄워 실제로 보내 확인한 결과이며, 문법만 검증한 것이고 서버 동작은 별개다.

## 예시와 각 부분
```bash
curl -X POST {BASE_URL}/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "Jun227", "password": "Password1!"}'

curl "{BASE_URL}/students?grade=2&classNum=3" \
  -H "Authorization: Bearer {accessToken}"
```
- `-H "Content-Type: application/json"`은 본문이 JSON이라고 서버에 알리는 헤더, `-d`가 요청 본문.
- 두 번째는 `-X`가 없어 기본 **GET**. `Authorization: Bearer <토큰>`이 빠지면 401.
- 줄 끝 `\`는 명령이 다음 줄로 이어진다는 표시.

## 인용이 맞는 이유
- **URL은 큰따옴표로 감싼다**: 쿼리의 `&`를 따옴표 없이 쓰면 쉘이 "백그라운드 실행"으로 읽어 뒤 파라미터(`classNum=3`)가 잘려 나간다.
- **JSON 본문은 작은따옴표로 감싼다**: `Password1!`의 `!`는 큰따옴표 안에서 bash가 히스토리 확장으로 해석해 오류가 날 수 있고, 작은따옴표 안에서는 안전하다.
- JS 소스에 이 문자열을 넣을 땐 `\`를 `\\`로 적어야 한다(문서 사이트 데이터 파일 등).

## 사용자 입력으로 cURL을 생성할 때
- 화면에서 URL·헤더·본문을 조합해 "cURL 복사" 문자열을 만들면, 큰따옴표에 넣은 값에 `$(…)`·`"`·백틱·`\`가 있을 때 붙여넣은 셸에서 깨지거나 치환이 실행된다. 모든 보간 값을 하나의 헬퍼로 작은따옴표 인용한다: `'` + `s.replace(/'/g, "'\''")` + `'` (xquare `/api-test`, [[xquare-openapi-frontend/프로젝트-현황]]).

## 알아둘 점
- `-d`를 쓰면 curl이 자동으로 POST로 보내므로 `-X POST`는 생략 가능. 문서에서는 메서드가 바로 보이게 남겨 두는 경우가 많다.
- 줄 끝 `\` 연결은 macOS/Linux(bash·zsh) 기준. Windows cmd·PowerShell에는 그대로 붙여넣을 수 없어, 필요하면 "macOS/Linux 기준" 문구를 단다.
- 이런 검증은 문법·전달 내용(메서드·경로·헤더·본문)만 보장한다. 서버가 그 필드명·파라미터 값을 쓰는지는 백엔드 명세로 따로 확인해야 한다(실제로 이 예시는 명세와 달라 이월됨 — [[xquare-openapi-frontend/프로젝트-현황]]).

## 출처
- Claude Code 세션 자동 캡처 (/data/project/xquare-openapi-frontend, 2026-10-21)
