# Docker란

- 애플리케이션과 실행 환경을 **컨테이너**라는 단위로 묶어서 실행할 수 있게 해주는 기술
- 프로그램 코드만 관리하는 것이 아니라 프로그램이 실행되는 데 필요한 환경까지 함께 묶는다.
- Docker를 사용하면 내 컴퓨터, 다른 사람의 컴퓨터, 서버 환경이 달라도 비슷한 실행 환경을 만들 수 있다.

한마디로
애플리케이션과 실행 환경을 컨테이너로 묶어서 어디서든 동일하게 실행할 수 있게 해주는 도구이다.

## Docker가 필요한 이유

개발을 하다보면 동료 컴퓨터에서는 되는데 내 컴퓨터에서는 안되거나 필요한 프로그램이 없거나 버전이 달라서 오류가 생기는 등등 여러 문제들이 발생합니다. 
Docker는 이런 문제를 줄이기 위해 사용합니다.

Docker를 사용하면 실행 환경을 이미지로 만들어 둘 수 있기 때문에 어디서 실행하든 같은 환경을 유지할 수 있습니다.

```
Docker 사용 전:서버마다 직접 Node.js 설치, 패키지 설치, 환경변수 설정 필요
Docker 사용 후:이미지를 만들고 서버에서는 그 이미지를 컨테이너로 실행
```

## Dockerfile, Image, Container

### Dockerfile

Dockerfile은 Docker 이미지를 만들기 위한 설명서이다.

Dockerfile에는 어떤 환경에서 실행할지, 어떤 파일을 복사할지, 어떤 명령어를 실행할지 등을 적는다.

#### 예시

```
FROM node:20

WORKDIR /app

COPY package.json .
RUN npm install

COPY . .

CMD ["npm", "run", "dev"]
```

이 Dockerfile의 의미는 다음과 같다.

```
FROM node:20
→ Node.js 20 환경을 사용한다.

WORKDIR /app
→ 컨테이너 안에서 /app 폴더를 작업 공간으로 사용한다.

COPY package.json .
→ package.json 파일을 복사한다.

RUN npm install
→ 필요한 패키지를 설치한다.

COPY . .
→ 현재 프로젝트 파일들을 컨테이너 안으로 복사한다.

CMD ["npm", "run", "dev"]
→ 컨테이너가 실행될 때 npm run dev 명령어를 실행한다.
```
### Image

Image는 컨테이너를 만들기 위한 실행 환경 템플릿이다.

이미지 안에는 애플리케이션 실행에 필요한 파일과 설정이 들어 있다.

```
- OS 기반 환경
- 런타임
- 라이브러리
- 프로젝트 코드
- 실행 명령어
```

쉽게 말하면

```
Image = 실행 준비가 끝난 파일 묶음
```
### Container

Container는 Image를 실제로 실행한 것이다.

쉽게 말하면

```
Image = 설계도
Container = 실제 실행 중인 프로그램
```

하나의 이미지로 여러 개의 컨테이너를 만들 수 있다.

```
my-app 이미지
├─ container 1
├─ container 2
└─ container 3
```

## 기본 동작 흐름

Docker 기본 흐름

```
Dockerfile 작성
→ Image 생성
→ Container 실행
```

명령어

```
docker build -t my-app .
docker run -p 3000:3000 my-app
```

의미는 다음과 같다.

```
docker build -t my-app .
→ 현재 폴더의 Dockerfile을 이용해서 my-app이라는 이미지를 만든다.

docker run -p 3000:3000 my-app
→ my-app 이미지를 컨테이너로 실행한다.
```

### 느낀점
---
도커를 듣기만 하고 잘 몰랐는데 이제 도커가 뭔지 알았으니까 직접 도커 파일 만들고 이미지도 만들고 컨테이너도 실행해봐야겠다.


