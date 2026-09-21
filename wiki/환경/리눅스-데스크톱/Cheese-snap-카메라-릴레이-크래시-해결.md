---
tags: [cheese, snap, pipewire, v4l2loopback, libcamera]
updated: 2026-09-22
---

# Cheese(snap) 카메라 앱 크래시와 Camera Relay 인식 문제

IPU7 카메라 노트북에서 snap으로 설치한 Cheese가 안 켜지던 문제는 **원인이 두 겹**이었다. (1) 세션 전역 `SPA_PLUGIN_DIR` 환경변수가 snap 안으로 새어 들어가 **SIGSEGV**, (2) 크래시를 고친 뒤에도 v4l2loopback 릴레이 장치가 `exclusive_caps=0`이라 snap 안의 GStreamer가 장치를 **M2M으로 오인해 목록에서 제외**. 각각 환경변수 제거와 `exclusive_caps=1`로 해결했고 사용자가 정상 동작을 확인했다.

## 배경 구조
- 이 PC는 IPU7 원시 ISP 노드를 그대로는 앱이 못 쓰기 때문에, libcamera → PipeWire → **Camera Relay**(v4l2loopback 가상 `/dev/video0`) 서비스를 별도로 구성해 뒀다 (온디맨드 시작, 첫 프레임은 검은색일 수 있음).
- Cheese가 `camera='ipu7'`(원시 ISP 노드)를 잡으면 프리뷰가 정상일 수 없다 → 릴레이 장치(`Camera Relay`)를 선택해야 한다.

## 문제 1 — `SPA_PLUGIN_DIR` 누수로 SIGSEGV
- 증상: Cheese 실행 즉시 세그폴트.
- 원인: `/etc/profile.d/libcamera-ipa.sh`, `/etc/environment.d/libcamera-ipa.conf`가 세션 전역에 `SPA_PLUGIN_DIR=/usr/lib/x86_64-linux-gnu/spa-0.2`를 export → 환경변수가 **snap 내부로 그대로 상속**되어 snap 자체의 SPA 플러그인 경로를 덮어씀 → snap 마운트 네임스페이스엔 그 경로가 없음 → `pw_loop_new()` 실패 → NULL 참조 → SIGSEGV.
- 이 값은 libpipewire의 **컴파일 기본값과 동일**해서 호스트 앱에는 원래 불필요했다(변수 없이도 libcamera 노드 정상 열거). 그래서 해당 export 줄만 제거하면 된다.
- 일반 교훈: **전역(`/etc/environment.d`, `profile.d`)에 라이브러리 경로류 변수를 export하면 snap/flatpak 앱이 깨질 수 있다.** 필요한 서비스 유닛에만 국한해서 설정한다.
- 확인 방법: 셸이 아니라 GNOME이 앱을 띄우는 방식(**systemd scope**)으로 실행해 테스트해야 실제 환경과 같다.

## 문제 2 — snap 안에서 Camera Relay가 안 보임
- 증상: 캡처는 잘 되는데 snap 내부 장치 목록에 `/dev/video0`이 없음.
- 원인: **GStreamer(v4l2 device provider)는 capture와 output 캡을 동시에 보고하는 장치를 M2M(메모리-투-메모리)으로 간주해 무시**한다. v4l2loopback을 `exclusive_caps=0`으로 만들면 정확히 그 상태다. (릴레이 스크립트 주석에 Chromium이 같은 이유로 못 보던 문제를 피하려는 의도가 적혀 있었음 — GStreamer도 같은 규칙.)
- 호스트에서는 GStreamer가 **PipeWire 경로**로 Camera Relay를 정상 인식한다. 하지만 **snap은 PipeWire 소켓에 접근할 수 없어** 그 경로를 못 쓴다 → snap Cheese를 살리는 방법은 v4l2 경로뿐이고, 그러려면 `exclusive_caps=1`이 필요.
- 해결: v4l2loopback을 `exclusive_caps=1`로 재로드 → Device Caps가 **capture 전용**(`0x05200001`)으로 바뀌고 snap이 `Camera Relay`를 인식, Cheese에서 스트리밍 시작(릴레이 상태가 `STREAMING`으로 전환되는 것으로 확인).
- 트레이드오프: `exclusive_caps` 값은 다른 소비자(Chromium 등)의 인식에도 영향을 주므로 바꾼 뒤 **모든 소비자를 다시 검증**해야 한다 (Claude 보충: 세션에서 Chromium 재검증 결과는 기록되지 않음).

## 진단 요령
- 크래시는 PipeWire 쪽 SIGSEGV → 환경변수 유입 의심 → 변수 유무를 바꿔가며 실행해 확정.
- 장치 목록은 호스트와 snap 내부(`snap run --shell`)에서 각각 비교해 격리 문제인지 탐색 단계 문제인지 가른다. 호스트 GStreamer 열거 테스트를 Python으로 할 땐 `gi` 모듈 유무부터 확인(없으면 "안 보임"으로 오판).
- Cheese가 `camera` 설정값(gsettings)을 `ipu7`로 되돌려 쓰면 선택 실패 후 폴백한 정황. 디버그 로그(`GST_DEBUG`)로 흐름을 본다.
- 실행 중인 셸이 끝나면 자식 Cheese도 같이 죽으니 "죽었다"만으로 실패 판정하지 말고 지속 스트리밍 여부로 판단.

## 미해결 — 좌우 반전
- 사용자가 프리뷰의 좌우 반전을 원치 않았다. 반전은 Cheese 자체 효과가 아니라 릴레이/libcamera 경로에서 오는 것으로 봤다(Cheese 기본 효과는 `identity`).
- Cheese에는 **"뒤집기(Flip)"** 효과가 내장(`videoflip video-direction=horiz`)돼 있어 Cheese에서만 되돌릴 수 있다. 릴레이 단에서 고치면 모든 앱에 적용되는 차이가 있어 범위를 고르는 단계에서 세션이 중단됨 → **적용 여부 미확정**.

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-22 (raw/ 파일 없음)
- 관련: [[PC-중복-설치-정리-Homebrew-도입]] (snap 앱 관련 정리), [[GNOME-Wayland-wl-clipboard-포커스-토스트]]
