# Nginx 정리

## 1. Nginx란?

Nginx는 사용자의 요청을 가장 먼저 받는 웹 서버이다.

사용자가 브라우저에서 서비스 주소로 접속하면 요청은 먼저 Nginx로 들어온다.  
Nginx는 설정에 따라 정적 파일을 사용자에게 응답할 수 있다.

```txt
사용자 브라우저
      ↓
Nginx
      ↓
정적 파일 응답
```

즉, Nginx는 서비스의 입구 역할을 한다.

---

## 2. 정적 파일 배포

프론트엔드 프로젝트를 build하면 HTML, CSS, JavaScript 파일이 생성된다.

```txt
index.html
assets/
  ├─ JavaScript 파일
  └─ CSS 파일
```

이런 파일들은 서버에서 따로 계산하지 않고 그대로 사용자에게 전달할 수 있기 때문에 정적 파일이라고 한다.

Nginx는 지정된 폴더에서 정적 파일을 찾아 사용자에게 응답한다.

```nginx
root /var/www/nginx-demo;
index index.html;
```

위 설정은 사용자가 접속했을 때 `/var/www/nginx-demo` 폴더 안의 `index.html`을 기본 파일로 보여주겠다는 의미이다.

```txt
http://localhost
      ↓
Nginx
      ↓
/var/www/nginx-demo/index.html
```

---

## 3. Nginx 설정 예시

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/nginx-demo;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 4. 설정 코드 설명

```nginx
listen 80;
```

80번 포트로 들어오는 HTTP 요청을 받겠다는 의미이다.  
브라우저에서 `http://localhost`로 접속하면 기본적으로 80번 포트를 사용한다.

```nginx
server_name localhost;
```

`localhost` 주소로 들어온 요청을 이 설정에서 처리하겠다는 의미이다.

```nginx
root /var/www/nginx-demo;
```

Nginx가 사용자에게 보여줄 정적 파일의 위치를 지정한다.

```nginx
index index.html;
```

기본으로 보여줄 파일을 `index.html`로 지정한다.

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

`/`로 시작하는 요청을 처리하는 설정이다.

요청한 파일이 실제로 있으면 그 파일을 보여주고, 없으면 `index.html`을 보여준다.

---

## 5. Nginx 설정 파일 위치

Nginx 설정 파일은 보통 다음 위치에 작성한다.

```bash
/etc/nginx/sites-available/
```

실제로 활성화된 설정은 다음 위치에 연결된다.

```bash
/etc/nginx/sites-enabled/
```

설정 파일을 수정한 뒤에는 문법 검사를 해야 한다.

```bash
sudo nginx -t
```

정상이라면 다음과 같이 나온다.

```txt
syntax is ok
test is successful
```

설정을 적용할 때는 다음 명령어를 사용한다.

```bash
sudo systemctl reload nginx
```

---

## 6. 로그 확인

Nginx는 요청 기록과 에러 기록을 로그로 남긴다.

접속 로그 확인:

```bash
sudo tail -f /var/log/nginx/access.log
```

에러 로그 확인:

```bash
sudo tail -f /var/log/nginx/error.log
```

`access.log`에서는 어떤 요청이 들어왔고 어떤 상태 코드로 응답했는지 확인할 수 있다.

`error.log`에서는 요청 처리 중 발생한 오류 원인을 확인할 수 있다.

---

## 7. 자주 발생하는 오류

### 404 Not Found

요청한 파일이나 경로를 찾지 못했을 때 발생한다.

예를 들어 실제 서버 폴더에 없는 파일을 요청하면 404가 발생할 수 있다.

### 403 Forbidden

권한 문제로 Nginx가 파일을 읽을 수 없을 때 발생한다.

### 502 Bad Gateway

Nginx가 뒤에 있는 서버로 요청을 전달하려고 했지만, 해당 서버에 연결하지 못했을 때 발생한다.

예를 들어 백엔드 서버가 꺼져 있는데 Nginx가 그 서버로 요청을 넘기려고 하면 502가 발생할 수 있다.

