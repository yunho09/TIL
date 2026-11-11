---
tags: [vite, nginx, spa, deploy, react-router]
updated: 2026-11-11
---

# Vite SPA를 nginx로 정적 서빙하기

Vite로 빌드한 SPA는 서버에서 Node를 돌릴 필요 없이 `dist`를 올리고 nginx가 파일만 내주면 된다. nginx 설정에서 `try_files` 폴백과 `/assets/` 장기 캐시가 핵심이다.

## 구조
- `bun run build` 결과물은 `dist/index.html` + `assets/index-<해시>.js|css`. React는 브라우저에서 실행되므로 서버는 **요청받은 파일을 찾아 보내는 일**만 한다.
- 흐름: 브라우저 → 앞단 프록시(openresty 등, HTTPS·도메인 라우팅) → http :80 nginx → `root` 폴더 응답. API 호출은 브라우저가 백엔드로 직접 보낸다.
- `VITE_*`는 빌드 시점에 번들에 박히므로 환경(prod/stag)마다 **빌드를 따로** 해야 한다 → [[Vite-빌드타임-환경변수-인라인]]. 모드 분리는 `vite build --mode staging` + `.env.staging`.

## nginx 설정 핵심
```nginx
server {
    listen 80;
    root /var/www/xquare;
    index index.html;
    location / { try_files $uri $uri/ /index.html; }
    location /assets/ { expires 1y; add_header Cache-Control "public, immutable"; }
}
```
- **`try_files ... /index.html`**: `BrowserRouter`를 쓰면 `/dashboard` 새로고침 시 nginx가 그런 파일을 찾다 404를 낸다. 파일이 없으면 `index.html`을 주고 라우팅은 React Router가 하게 한다. (Cloudflare 쪽 같은 증상: [[Cloudflare-Workers-SPA-fallback-404]])
- **1년 캐시가 안전한 이유**: Vite는 내용이 바뀌면 파일명 해시가 바뀌고, 새 `index.html`이 새 파일을 가리킨다.
- 서브경로(`/admin/`)에 올리려면 `vite.config.ts`에 `base: "/admin/"`; 서브도메인+server 블록이 더 간단.
- 적용: `nginx -t && systemctl reload nginx`, 확인 `curl -I localhost:포트` (파일 없으면 403/404가 정상).
- 한 서버에 prod/stag를 두려면 `listen` 포트와 `root` 폴더만 다른 설정 2개.

## `vite preview`를 운영에 안 쓰는 이유
- Vite 문서상 빌드 결과 미리보기용이지 운영 서버용이 아니다.
- 서버에 bun/node 설치와 pm2 같은 프로세스 관리가 따로 필요한 반면, nginx는 systemd가 상시 기동·재부팅 복구를 해준다.

## 터미널 붙여넣기 함정
- 서버 셸에 붙여넣은 첫 명령이 `^[[200~`가 앞에 붙어 깨지면 bracketed paste 탓. 셸에서 직접 `bind 'set enable-bracketed-paste off'`를 타이핑한다.

## 출처
- Claude Code 세션 자동 캡처 (xquare-admin-application, 2026-11-11) — raw 파일 없음
