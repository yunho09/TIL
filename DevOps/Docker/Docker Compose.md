## 1. Docker Compose란?

Docker Compose는 **여러 Docker 컨테이너의 실행 설정을 하나의 YAML 파일에 작성하고 한 번에 실행하고 관리하는 도구**다.

예를 들어 백엔드와 DB를 각각 `docker run`으로 실행하면 명령어가 길어진다.

```bash
docker run ...
docker run ...
```

Compose를 사용하면 `compose.yaml`에 설정을 작성하고 다음 명령어 하나로 모두 실행할 수 있다.

```bash
docker compose up
```

Compose 파일에서는 서비스, Network, Volume 등의 설정을 함께 관리할 수 있다.

---

## 2. Dockerfile과 Docker Compose의 차이

```text
Dockerfile
→ 하나의 Docker Image를 만드는 방법을 작성

compose.yaml
→ 하나 이상의 Container를 어떻게 실행하고 연결할지 작성
```

예를 들어 백엔드 Image는 Dockerfile로 만들고 백엔드와 DB를 함께 실행하는 설정은 Compose에 작성한다.

---

## 3. 기본 파일 구조

프로젝트 최상위 폴더에 `compose.yaml` 파일을 만든다.

```text
docker-backend-practice/
├── Dockerfile
├── compose.yaml
├── package.json
└── index.js
```

기본 예시는 다음과 같다.

```yaml
services:
  backend:
    build: .
    container_name: backend
    ports:
      - "8080:8080"
```

### 주요 설정

```yaml
services:
```

Compose로 실행할 서비스 목록이다. 서비스 하나가 보통 Container 하나의 역할을 한다.

```yaml
backend:
```

서비스 이름이다.

```yaml
build: .
```

현재 폴더의 Dockerfile을 사용해서 Image를 만든다.

```yaml
container_name: backend
```

Container 이름을 `backend`로 지정한다.

```yaml
ports:
  - "8080:8080"
```

호스트의 `8080`번 포트와 Container의 `8080`번 포트를 연결한다.

```text
브라우저
localhost:8080
      ↓
호스트 8080
      ↓
Container 8080
      ↓
백엔드 서버
```

---

## 4. 백엔드와 DB 함께 실행하기

```yaml
services:
  backend:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_PORT: 3306
    depends_on:
      - db

  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: practice
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

### `build`와 `image`

```yaml
build: .
```

내 프로젝트의 Dockerfile로 Image를 만든다.

```yaml
image: mysql:8.4
```

이미 만들어진 MySQL Image를 사용한다.

```text
build
→ Dockerfile로 직접 Image 생성

image
→ 기존에 만들어진 Image 사용
```

---

## 5. Container 사이의 통신

Compose는 기본적으로 서비스들을 같은 Docker Network에 연결한다.

같은 Compose Network에 있는 Container들은 **서비스 이름을 주소처럼 사용**할 수 있다.

```yaml
services:
  backend:
  db:
```

백엔드에서 DB에 접근할 때는 다음 주소를 사용한다.

```text
db:3306
```

여기서 `db`는 DB 서비스 이름이다.

```text
backend Container에서 localhost
→ backend Container 자기 자신

db
→ db Container
```

따라서 백엔드 Container에서 DB 주소를 `localhost`로 설정하면 안 되고 서비스 이름인 `db`를 사용해야 한다.

---

## 6. Volume

Container를 삭제하면 Container 내부에 저장된 데이터도 사라질 수 있다.

Volume은 데이터를 Container 바깥에 따로 저장하여 Container를 삭제하거나 다시 만들어도 데이터를 유지하게 한다.

```yaml
volumes:
  - db-data:/var/lib/mysql
```

```text
db-data
→ Docker가 관리하는 데이터 저장 공간

/var/lib/mysql
→ MySQL이 Container 내부에서 데이터를 저장하는 위치
```

주로 DB 데이터 유지에 사용한다.

---

## 7. 환경변수

Compose 파일에서 환경변수를 전달할 수 있다.

```yaml
environment:
  DB_HOST: db
  DB_PORT: 3306
```

또는 `.env` 파일을 사용할 수 있다.

```yaml
env_file:
  - .env
```

```
env
DB_HOST=db
DB_PORT=3306
```

---

## 8. `depends_on`

```yaml
depends_on:
  - db
```

백엔드보다 DB Container를 먼저 시작하도록 의존 관계를 지정한다. 다만 DB Container가 시작됐다는 것이 DB 프로그램의 준비까지 끝났다는 뜻은 아니므로 실제 프로젝트에서는 재연결 처리가 필요할 수 있다.

---

## 9. 핵심 명령어

```bash
# 실행
docker compose up -d

# Image를 다시 빌드하고 실행
docker compose up -d --build

# 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f

# 특정 서비스 로그
docker compose logs -f backend

# 중지
docker compose stop

# 다시 시작
docker compose start

# 재시작
docker compose restart

# Container와 Network 삭제
docker compose down
```

Volume까지 삭제하려면 다음 명령어를 사용한다.

```bash
docker compose down -v
```

DB 데이터까지 사라질 수 있으므로 주의해야 한다.

## 느낀점
---
