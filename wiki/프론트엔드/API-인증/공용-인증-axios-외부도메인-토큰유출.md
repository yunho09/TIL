---
tags: [frontend, auth, axios, cors, security]
updated: 2026-09-16
---

# 공용 인증 axios 인스턴스로 외부 도메인을 부르면 토큰이 샌다

공용 axios 인스턴스에 인증 인터셉터를 붙여 모든 요청에 Bearer 토큰을 자동으로 실어 보내는 구조에서는, 그 인스턴스로 **우리 API 서버가 아닌 외부 도메인**(CDN, 서드파티 등)을 호출하면 안 된다. 토큰이 그 서버로 새어 나가고, 그 서버가 준 403 응답이 "세션 만료"로 오인되어 로그아웃까지 유발할 수 있다.

## 원인

요청 인터셉터는 보통 "로그인/재발급 경로인지"만 보고 그 외 모든 요청에 토큰을 붙인다 — **호출하는 URL이 자사 서버인지 외부 서버인지는 판단하지 않는다.**

```ts
api.interceptors.request.use((config) => {
  if (isPublicAuthPath(config.url)) return config   // 예외는 이것뿐
  const accessToken = readAccessToken()
  if (accessToken) config.headers.set('Authorization', `Bearer ${accessToken}`)
  return config
})
```

여기에 axios의 성질이 하나 겹친다. `baseURL`이 설정돼 있어도 **절대 URL을 주면 axios는 baseURL을 무시하고 그 주소로 그대로 요청을 보낸다.** 하지만 인터셉터는 그대로 실행된다. 그래서 `api.get('https://cdn.example.com/file.pdf')`처럼 절대 URL로 외부 서버를 부르면, 우리 서버로 가는 요청과 똑같이 `Authorization: Bearer <토큰>` 헤더가 붙은 채 그 외부 서버로 나간다.

## 왜 문제인가

- **토큰이 남의 인프라 로그에 남는다.** bearer 토큰은 "가진 사람이 곧 권한"이라 비밀번호와 실질적 무게가 비슷하다. CDN 등은 애초에 이 값이 필요 없는데(공개 파일이면 인증 없이도 200이 온다), 요청마다 반복해서 새어 나간다.
- **응답 인터셉터가 있으면 로그아웃까지 번진다.** "403 = 세션 만료"로 일괄 처리하는 앱이라면, 외부 서버가 주는 403(파일 없음, 키 오류, 버킷 권한 등 전혀 다른 의미)도 세션 무효로 해석돼 `endSession()`이 불려 로그인 화면으로 튕긴다. 파일 하나 못 받은 게 입력 중이던 폼을 통째로 날리는 일로 번질 수 있다.

## 해결

외부 도메인 호출은 인증 인터셉터가 걸리지 않는 별도 경로로 우회한다 — 가장 간단한 방법은 브라우저 내장 `fetch`를 직접 쓰는 것.

```ts
export async function fetchStoredFile(fileKey: string): Promise<Blob> {
  const response = await fetch(storedFileUrl(fileKey))
  if (!response.ok) throw new Error(`파일을 받지 못했습니다(HTTP ${response.status}).`)
  return response.blob()
}
```

`fetch`는 인터셉터와 무관하므로 우리가 명시한 헤더만 나가고(토큰 없음), 403이 와도 `response.ok === false`일 뿐 세션에는 손대지 않는다. 실패는 예외로 던져 화면이 토스트 등으로 처리한다.

## 검토했지만 기각한 대안

- **인증 없는 axios 인스턴스를 새로 만들기** — 기술적으로 같은 효과지만, "공용 인스턴스 외 새 axios 인스턴스 금지" 같은 팀 규칙이 있다면 규칙을 하나 더 어기는 셈이라 예외가 더 커진다.
- **인터셉터에 "외부 도메인이면 토큰 생략" 조건 추가** — 인증 로직 자체에 예외 분기가 늘어난다. 나중에 누가 그 조건을 잘못 건드리면 토큰이 조용히 다시 새어 나갈 수 있어, 경계를 아예 다른 함수(`fetch`)로 분리하는 쪽이 더 안전하다.

## 회귀를 e2e로 막기

말로만 정해두면 "다들 공용 인스턴스 쓰는데 여기만 왜 `fetch`지?" 하고 나중에 누가 되돌리기 쉽다. 아래 두 가지를 테스트로 못박아 둔다.

- 외부 서버로 간 요청의 `authorization` 헤더가 `undefined`인지
- 외부 서버가 403을 줬을 때 URL이 로그인 페이지로 안 튕기고 그대로인지(세션 유지 확인)

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho/orca/workspaces/ToyVillage-Admin-FE/develop) — [[ToyVillage-Admin-FE/프로젝트-현황]] (#71 첨부 다운로드, CDN 파일 서버 호출에 `fetch` 사용 결정)
