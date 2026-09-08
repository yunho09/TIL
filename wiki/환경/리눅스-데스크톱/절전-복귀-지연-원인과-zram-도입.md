---
tags: [리눅스, 절전, 스왑, zram, systemd]
updated: 2026-09-06
---

노트북이 절전(suspend)에서 깨어난 뒤 화면이 바로 켜지지 않는 문제의 원인을 로그로 추적한 결과, 커널 자체의 복귀는 항상 빠르고(0.2~0.8초) **디스크 스왑 고갈 + 복귀 직후 몰려 실행되는 systemd 타이머**가 겹칠 때만 9초 이상 지연됨을 확인했다. zram(압축 스왑) 도입과 타이머 조정으로 조치했다.

## 증상과 진단 방법

`journalctl`에서 "Lid opened"부터 잠금화면(fprintd 호출)까지 걸린 시간을 재면 복귀 지연을 정량화할 수 있다:

```bash
journalctl -b --no-pager -o short-precise | grep -E "Lid opened|unit='fprintd" | tail -4
```

- 정상: 0.2~0.8초
- 문제 상황: 9.8초 (커널의 `PM: suspend exit`은 236ms로 정상이었는데, 잠금화면이 뜨기까지 9.8초 걸림)

## 원인

증상이 있던 날은 다음 조건이 동시에 겹쳤다:

1. **스왑이 완전히 포화** — `swap: 8.0Gi 중 7.9Gi 사용, 여분 53Mi`. 부팅 이후 스왑 인 124만 페이지, major fault 87만 회.
2. **복귀 직후 타이머 폭주** — 절전 중 놓친 실행이 `Persistent=true`인 타이머(`devlog-sync.service`, `snap.firmware-updater.firmware-notifier` 등)가 복귀 즉시 한꺼번에 발동해 CPU와 메모리를 요구함. 특히 `snap.firmware-updater.firmware-notifier`가 CPU 19.8초를 먹었다.
3. Wi-Fi 재연결이 늦어지면(8초 지연) 네트워크를 기다리는 서비스들이 그 시간만큼 메모리를 더 붙잡고 있었다.

결과적으로: 스왑 꽉 참 → 복귀 직후 무거운 서비스들이 새 메모리 요구 → 커널이 회수하려고 gnome-shell 페이지를 (포화 상태인 디스크) 스왑에서 도로 읽어옴 → 화면이 안 켜짐. 같은 서비스라도 스왑에 여유가 있고 타이머가 안 겹친 날은 0.5~0.8초로 정상 복귀했다.

## 해결

1. **zram 도입 (핵심 조치)** — `systemd-zram-generator` 설치, `/etc/systemd/zram-generator.conf`에 8192MB / zstd / priority 100 설정. 이후 새로 스왑되는 페이지는 디스크(`/swap.img`, prio -1) 대신 압축된 RAM(`/dev/zram0`, prio 100)으로 우선 들어가 수십 배 빠르다.
   - **함정**: 패키지 설치 시점에 이미 기본값(4G/lzo-rle)으로 zram 장치가 생성되어 있어서, 설정 파일을 고친 뒤 `systemctl start`만 해서는 반영되지 않았다. 스왑 유닛을 완전히 내렸다 올려야 새 설정(8G/zstd)이 적용됨.
2. **zram에 맞는 커널 파라미터** — `/etc/sysctl.d/99-zram-tuning.conf`
   ```
   vm.page-cluster = 0    # 기본 3 → 0, zram은 readahead가 오히려 낭비
   vm.swappiness   = 100  # 기본 60 → 100, 캐시를 버리기보다 압축 스왑 우선
   ```
3. **복귀 직후 타이머 폭주 완화**
   - `snap.firmware-updater.firmware-notifier.timer`를 `systemctl --user mask` (되돌리려면 `unmask`)
   - `devlog-sync.timer`에 `RandomizedDelaySec=20min` 추가해 정각 실행이 몰리지 않게 분산 (원본은 `devlog-sync.timer.bak`으로 백업)

## 한계 / 남은 것

기존에 이미 꽉 차 있던 `/swap.img`(디스크 스왑)는 조치 직후에는 여전히 대부분 사용 중이라, **재부팅 전까지는 복귀가 완전히 빨라지지 않는다.** 재부팅하면 디스크 스왑이 비워지고 zram이 처음부터 주력으로 잡히면서 완전히 해결된다.

근본적으로는 RAM 30GB 중 25GB를 상시 사용하는 상태(Figma 다중 인스턴스, Docker Desktop VM(qemu), IntelliJ 등)라, 안 쓰는 앱을 정리하는 것도 체감 효과가 크다.

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-06.
