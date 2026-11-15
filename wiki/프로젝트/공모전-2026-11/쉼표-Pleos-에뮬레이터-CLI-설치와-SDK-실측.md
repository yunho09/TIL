---
tags: [Pleos, Android, 에뮬레이터, AAOS, 공모전]
updated: 2026-11-16
---

# 쉼표 — Pleos 에뮬레이터 CLI 설치와 SDK 실측

현대 차량용 앱 공모전(쉼표)을 리눅스 PC에서 Android Studio 없이 빌드·에뮬레이터 실행까지 한 기록이다.
루트 디스크가 꽉 차 snap 설치가 안 돼 `/data`에 명령줄 도구로 깔았고, 짐작으로 쓴 SDK 시그니처를 에뮬레이터에서 실측해 고친 내용을 남긴다.

## 설치 (Android Studio 없이, sudo 없이)
- **이유**: 루트 `/`가 99% 차서(1.1G 남음) snap 설치 불가 → [[루트-공간-부족-Docker-디스크-data-이전과-Discord-로그-스팸]]과 같은 상황. 빌드·에뮬레이터·Virtual Console에는 Studio가 필요 없다.
- SDK는 `/data/android/sdk`에 command-line tools + platform-tools + emulator + platform 35. AVD는 `/data/android/avd`.
- KVM은 `kvm` 그룹에 이미 있으면 sudo 불필요 (`/dev/kvm` 권한·그룹 확인).
- **Pleos 시스템 이미지**: 공식 문서의 `crp-sys-img.xml` 사이트를 `sdkmanager`가 읽지 못했다 → XML에 적힌 zip(약 4GB)을 직접 받아 **체크섬 확인 후 SDK의 `system-images/` 아래에 풀었다**. 패키지 ID는 `system-images;android-34;pleos_car_emulator;x86_64`, emulator 24 이상 필요. 압축 해제 후 약 19GB.
- AVD: 기기 프로필 Pleos Connect v2.1.0, 2560×1440, RAM 4096MB, Android 14.
- 시간대가 서울이 아니면 화면 시각이 틀리므로 에뮬레이터 시간대를 서울로 바꿨다.

## 실행 함정
| 증상 | 원인 | 해결 |
|---|---|---|
| 창이 모니터보다 커서 오른쪽(점검 탭)이 잘림 | HiDPI 200% 배율에서 2560×1440을 "자동 배율"로 띄움 | AVD `config.ini`의 `window.scale = 0.55` 고정 (창 약 1400×800). 실행 스크립트에서 `PLEOS_WINDOW_SCALE=0.7`처럼 덮어쓰기 |
| 스냅샷으로 빨리 켜면 에뮬레이터가 자꾸 죽음 | GPU 오류 | 항상 **콜드 부팅**(`-no-snapshot-load`류)으로 |
| 앱 화면 초당 5장, TTS 소리 지직거림, 녹화 끊김 | 에뮬레이터가 노트북 Intel GPU를 신뢰하지 못해 CPU(SwiftShader)로 렌더링 | **`-gpu host` 강제** → 앱 렌더링 초당 약 5장 → 약 40장. 녹화 fps 4 → 11 → 19로 개선 |
| 소리가 끊김 | PC 부하(kind·Gradle 데몬·Claude 앱 VM) + 위 GPU 문제 | Gradle 데몬 종료, 오디오 지연 늘림, PT 때는 kind도 끔 |
- 실행 스크립트는 `/data/android/start-pleos.sh`로 묶어 두면 재현이 쉽다 (배율·콜드부팅·`-gpu host` 고정).

## 에뮬레이터 조작
- **user 빌드라서 `adb`로 기어를 못 바꾼다.** → Virtual Console(Motion → Gear)을 쓴다.
- `adb forward tcp:8088 tcp:8088` 후 브라우저 `http://127.0.0.1:8088`. 앱 안 브라우저로 열어도 된다.
- Virtual Console 창 크기가 바뀌면 기어 버튼 위치가 달라진다 (자동화할 때 주의).
- 화면 녹화: 안드로이드 기본 `screenrecord`는 소리가 없다. 부하가 크면 4fps까지 떨어졌고, 에뮬레이터 자체 녹화 + GPU 모드에서 19fps.

## SDK 실측 결과 (짐작 → 확인)
- **리스너 인터페이스에 `onFailed`가 필요**했다 (컴파일 에러 2곳). 짐작했던 나머지 시그니처(`DoorArea`·`SeatArea` 생성자, TTS `EventListener`, STT `ResultListener`, `setLocale`, `unregisterDoorPosition`)는 실제 클래스와 맞았다.
- **문 잠금**: 에뮬레이터는 운전석 단독(0x1)이 아니라 **4개 문 묶음 영역**만 받는다. → 영역을 하드코딩하지 말고 **SDK에 지원 영역을 먼저 물어본 뒤** 그 영역으로 호출.
- **공조 전원·내기 순환**: 좌석 5개를 묶은 영역(0x75)만 받는다. 같은 방식으로 지원 영역 조회.
- **`remainTime`은 초 단위**: 내비 화면 15분 → 앱에서 900.
- **목적지가 없으면 내비가 빈 값**을 줘서 경유지 추가가 실패 → 이때는 휴게소로 경로 요청.
- TTS는 `onDone`까지 재생되고 소리도 호스트 스피커로 나간다. TTS·STT가 **ON_DEVICE 모드면 Pleos Client ID/Secret이 필요 없다**. HYBRID로 바꿀 때만 `local.properties`/환경변수에 넣고 커밋 금지.
- 미확인(시연 전 점검 필요): STT 말소리 인식(호스트 마이크 입력 필요), 속도에 따른 주행 제한 화면, `heading`(정지 상태라 0.0만 확인).
- 기어는 AAOS `CarPropertyManager`(GEAR_SELECTION)로 읽는다. 앱은 에뮬레이터의 기어 D를 받아 주행 화면으로 바뀌었다.

## 릴리스 APK
- 서명 키는 **저장소 밖**(`/data/android/keys`, 권한 600)에 만들고 `build.gradle.kts`는 `local.properties`에서 키 위치만 읽는다. 키 분실 시 같은 서명으로 재배포 불가 → 제출 후 백업.
- release APK 43MB (debug는 51MB로 GitHub 경고 기준 50MB 초과).

## 출처
- [[raw/Claude Code 세션 자동 캡처 (/data/downloads/shimpyo)]]
- 관련: [[공모전-제출-운영-복수출품-메일-규정과-커밋-해시-관리]]
