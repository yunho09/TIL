## 1. Nginx란?

Nginx는 사용자의 요청을 가장 먼저 받는 **웹 서버**이다.

사용자가 브라우저에서 서비스 주소로 접속하면 요청은 먼저 Nginx로 들어온다.  
Nginx는 요청의 종류에 따라 프론트엔드 파일을 보여주거나, 백엔드 서버로 요청을 전달한다.

```
사용자 브라우저
      ↓
Nginx
      ↓
프론트엔드 파일 또는 백엔드 서버
```

즉, Nginx는 서비스의 **입구 역할**을 한다.

## 2. Nginx의 주요 역할

## 2-1. 정적 파일 서빙

프론트엔드 프로젝트를 build하면 HTML, CSS, JavaScript 파일이 만들어진다.

React/Vite 기준으로는 보통 다음 명령어를 사용한다.

```
npm run build
```

그러면 `dist` 폴더가 생성된다.

```
dist/
 ├─ index.html
 └─ assets/
    ├─ index.js
    └─ index.css
```

Nginx는 이 build된 정적 파일들을 사용자에게 전달한다.

```
사용자 요청
   ↓
Nginx
   ↓
index.html, JS, CSS 파일 전달
```

이렇게 하면 React 개발 서버를 실행하지 않아도 사용자가 웹 화면을 볼 수 있다.

---

## 2-2. Reverse Proxy

백엔드 서버는 보통 8080번 같은 포트에서 실행된다.

```
백엔드 서버: localhost:8080
```

하지만 사용자가 직접 8080번 포트로 요청을 보내는 것은 좋지 않다.

그래서 Nginx가 사용자의 요청을 먼저 받고, `/api`로 시작하는 요청만 백엔드 서버로 전달한다.

```
사용자 요청: http://localhost/api/users
      ↓
Nginx
      ↓
백엔드 서버: http://localhost:8080/users
```

이처럼 Nginx가 앞에서 요청을 받고 뒤에 있는 백엔드 서버로 대신 전달하는 구조를 **Reverse Proxy**라고 한다.

쉽게 말하면:

```
사용자는 Nginx만 보고,
Nginx가 뒤에 있는 백엔드 서버로 요청을 전달한다.
```

---

## 3. 우리가 만들 구조

이번 실습에서 만들 구조는 다음과 같다.

```
사용자 브라우저
      ↓
http://localhost
      ↓
Nginx : 80번 포트
   ├─ /        → 프론트엔드 정적 파일
   └─ /api     → 백엔드 서버 : 8080번 포트
```

`/` 요청은 프론트엔드 화면을 보여준다.

```
http://localhost/
```

`/api` 요청은 백엔드 서버로 전달한다.

```
http://localhost/api/...
```

---

## 4. Nginx 설치 및 상태 확인

Nginx 버전 확인:

```
nginx -v
```

Nginx 실행 상태 확인:

```
sudo systemctl status nginx
```

정상 실행 중이면 다음과 같이 나온다.

```
Active: active (running)
```

이 뜻은 Nginx가 현재 정상적으로 실행 중이라는 의미이다.

Nginx 시작:

```
sudo systemctl start nginx
```

Nginx 재시작:

```
sudo systemctl restart nginx
```

Nginx 설정 다시 적용:

```
sudo systemctl reload nginx
```

설정 파일 문법 검사:

```
sudo nginx -t
```

---

## 5. Nginx 설정 파일

Nginx 설정 파일은 보통 다음 위치에 있다.

```
/etc/nginx/sites-available/
```

실제로 적용되는 설정은 다음 위치에 연결된다.

```
/etc/nginx/sites-enabled/
```

예시 설정:

```
server {
    listen 80;
    server_name localhost;

    root /var/www/my-app;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 6. 설정 코드 설명

```
listen 80;
```

80번 포트로 들어오는 요청을 받겠다는 뜻이다.  
브라우저에서 `http://localhost`로 접속하면 기본적으로 80번 포트를 사용한다.

```
root /var/www/my-app;
```

프론트엔드 build 파일이 있는 위치를 지정한다.

```
index index.html;
```

기본으로 보여줄 파일을 `index.html`로 지정한다.

```
location / {
    try_files $uri $uri/ /index.html;
}
```

`/`로 들어오는 요청은 프론트엔드 파일을 보여준다.  
React Router를 사용할 때 새로고침하면 404가 날 수 있는데, 이 설정을 통해 `index.html`로 다시 보내준다.

```
location /api/ {
    proxy_pass http://localhost:8080/;
}
```

`/api`로 시작하는 요청은 백엔드 서버인 `localhost:8080`으로 전달한다.  
이 부분이 Reverse Proxy 설정이다.

---

## 7. Nginx 로그 확인

Nginx는 요청 기록과 에러 기록을 로그로 남긴다.

접속 로그 확인:

```
sudo tail -f /var/log/nginx/access.log
```

에러 로그 확인:

```
sudo tail -f /var/log/nginx/error.log
```

`access.log`는 사용자가 어떤 요청을 보냈는지 확인할 때 사용한다.

`error.log`는 404, 403, 502 같은 문제가 발생했을 때 원인을 찾기 위해 사용한다.

---

## 8. 자주 발생하는 오류

## 8-1. 404 Not Found

404는 요청한 파일이나 경로를 찾지 못했을 때 발생한다.

예시:

```
http://localhost/login
```

React Router를 사용하는 페이지에서 새로고침했을 때 404가 날 수 있다.

해결 방법:

```
location / {
    try_files $uri $uri/ /index.html;
}
```

---

## 8-2. 403 Forbidden

403은 권한이 없을 때 발생한다.

예시:

```
Nginx가 /var/www/my-app/index.html 파일을 읽을 권한이 없음
```

확인 명령어:

```
ls -al /var/www/my-app
```

파일이나 폴더 권한이 잘못되어 있으면 Nginx가 파일을 읽지 못해서 403이 발생할 수 있다.

---

## 8-3. 502 Bad Gateway

502는 Nginx가 백엔드 서버로 요청을 전달하려고 했지만, 백엔드 서버에 연결하지 못했을 때 발생한다.

예시:

```
/api 요청
   ↓
Nginx
   ↓
localhost:8080 백엔드 서버
```

그런데 백엔드 서버가 꺼져 있으면 502가 발생한다.

확인 명령어:

```
curl localhost:8080
```

또는 포트 확인:

```
sudo lsof -i :8080
```

---

## 9. 발표용 요약

Nginx는 사용자의 요청을 가장 먼저 받는 웹 서버이다.

프론트엔드 화면 요청이 들어오면 build된 HTML, CSS, JavaScript 파일을 사용자에게 전달한다.

그리고 `/api`로 시작하는 요청은 백엔드 서버로 전달한다.

이렇게 하면 사용자는 하나의 주소로 프론트엔드와 백엔드를 모두 사용할 수 있다.

또한 Nginx는 access.log와 error.log를 남기기 때문에, 문제가 발생했을 때 로그를 확인해서 원인을 찾을 수 있다.

---

## 10. 핵심 한 줄 정리

```
Nginx는 서비스의 입구 역할을 하며, 프론트엔드 정적 파일을 보여주고 /api 요청을 백엔드 서버로 전달한다.
```