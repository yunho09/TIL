---
tags: [NPU, intel_vpu, Ubuntu, OpenVINO, 갤럭시북5]
updated: 2026-11-02
---

# NPU가 안 쓰이는 이유 (Core Ultra 7 258V, Ubuntu 26.04)

이 노트북의 NPU는 하드웨어·커널은 정상인데, NPU를 호출할 **유저 공간 소프트웨어가 없어서** 사용 시간 카운터가 0이고 절전 상태로 놀고 있다. 손해는 작고, 쓸 앱이 생기기 전까지는 그대로 둬도 된다.

## 점검 결과 (직접 확인)
- **정상:** NPU 칩 인식, 커널 드라이버 `intel_vpu` 로드, 펌웨어 파일 존재, `/dev/accel/accel0` 접근 권한(직접 열어 봄).
- **없음:** 유저 드라이버·컴파일러(`intel-npu-driver`), Level Zero 로더(`libze1`), OpenVINO 같은 추론 프레임워크, NPU를 쓰는 앱(Ollama·Whisper 등).

## 원인
- **Ubuntu 26.04 apt 저장소에 NPU 유저 드라이버가 없다.** 따로 설치해야 한다.
- NPU는 기기 안에서 AI 모델을 돌릴 때만 쓰인다. 브라우저·Docker·빌드, 그리고 서버에서 모델이 도는 Claude Code는 해당 없음.
- 윈도우 Copilot+처럼 NPU를 자동으로 쓰는 데스크톱 기능이 리눅스엔 사실상 없다. 드라이버를 깔아도 **앱이 NPU를 직접 지정해야** 쓰인다.

## 손해 판단
- NPU가 놀아도 CPU·GPU 성능은 그대로. 이 칩 라인은 NPU가 기본 탑재라 옵션으로 돈을 더 낸 것도 아니다.
- 못 누리는 건 Copilot+, 카메라 배경 흐림 같은 윈도우용 NPU 기능뿐이며 리눅스에선 드라이버로도 해결 안 된다.

## 나중에 쓰게 된다면
- **용도:** 음성 받아쓰기(Whisper), 작은 로컬 모델. 저전력으로 오래 돌리는 작업에 맞다. 속도·한국어 품질은 이 노트북에서 **실측한 적 없음**.
- **설치:** Canonical snap `intel-npu-driver`(1.38.0) 또는 GitHub의 26.04용 패키지로 드라이버 설치 → OpenVINO 추가.
- **메모리:** NPU는 시스템 RAM을 공유한다. 당시 30GB 중 25GB 사용·가용 5GB라 큰 모델은 어렵고, 로컬 LLM은 Claude 대체로 부족하다. 메모리 상황은 [[리눅스-메모리-점유-앱별-진단-PSS]] 참고.

## 출처
- Claude Code 세션 자동 캡처 (2026-11-02, /home/yunho/.claude/projects/-home-yunho/memory)
