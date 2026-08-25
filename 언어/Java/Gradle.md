**빌드 도구**

- 프로젝트를 만들고 실행하는 데 필요한 잡일들을 자동화해주는 프로그램

구체적으로 세 가지를 한다.

1. **의존성 관리** — 필요한 라이브러리를 인터넷에서 받아옴
2. **빌드** — 소스코드를 컴파일해서 실행 가능한 결과물 생성
3. **태스크 실행** — 테스트 돌리기, 서버 켜기, 배포 파일 만들기

### npm과 비교

|       | npm                      | Gradle                       |
| ----- | ------------------------ | ---------------------------- |
| 설정 파일 | `package.json`           | `build.gradle.kts`           |
| 저장소   | npm registry             | **Maven Central**            |
| 설치 위치 | `node_modules/` (프로젝트 안) | `~/.gradle/caches/` (**전역**) |
| 실행    | `npm run dev`            | `./gradlew bootRun`          |
| 결과물   | 없거나 `dist/`              | `build/libs/*.jar`           |

### 중요한 차이: 설치 위치

npm은 프로젝트마다 `node_modules`를 새로 만든다. 
프로젝트 10개면 같은 라이브러리가 10벌 복사된다.

Gradle은 **홈 폴더에 한 번만** 받아두고 모든 프로젝트가 공유한다.

```
~/.gradle/caches/modules-2/files-2.1/
└── org.springframework.boot/
    └── spring-boot-starter-web/
        └── 3.4.0/...
```

그래서 첫 프로젝트는 로딩이 오래 걸리지만 두 번째부터는 훨씬 빠르다. 이미 받아놨으니까.

### Gradle은 라이브러리를 어디에 두는가

#### 구조

```
~/.gradle/caches/          ← 실제 jar 파일 저장소 (전역, 원본 1개)
    ├── spring-web-6.2.0.jar
    ├── jackson-2.17.0.jar
    └── h2-2.2.jar
              ▲
              │ 참조만 함 (복사 안 함)
              │
프로젝트A/                  프로젝트B/
└── build.gradle.kts       └── build.gradle.kts
    "spring-web 쓸래"           "spring-web, h2 쓸래
```

- 프로젝트 폴더에 `node_modules` 같은 게 **없음**
- 첫 프로젝트는 로딩 오래 걸리고, **2회차부터 빠름**
- 캐시에 라이브러리가 아무리 많아도 **프로젝트는 `build.gradle.kts`에 적힌 것만 사용**