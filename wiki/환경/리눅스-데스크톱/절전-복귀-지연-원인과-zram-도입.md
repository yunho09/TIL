---
tags: [리눅스, 절전, 스왑, zram, systemd, cgroup, logind, inhibitor]
updated: 2026-09-09
---

노트북이 절전(suspend)에서 깨어난 뒤 화면이 바로 켜지지 않는 문제의 원인을 로그로 추적한 결과, 커널 자체의 복귀는 항상 빠르고(0.2~0.8초) **디스크 스왑 고갈 + 복귀 직후 몰려 실행되는 systemd 타이머**가 겹칠 때만 9초 이상 지연됨을 확인했다. zram(압축 스왑) 도입과 타이머 조정으로 조치했다.

**2026-09-08 갱신**: zram 도입 후에도 증상이 재발해 재조사한 결과, 스왑 고갈은 이미 해소돼 있었고(디스크 스왑 0B, zram이 5.3GB 흡수) 남은 원인은 두 겹이었다 — ① 복귀 직후 `Persistent=true` 타이머 폭주로 인한 CPU 경합(느린 복귀 4/23회 전부에서 확인), ② **gnome-shell 자신이 zram으로 내보낸 페이지를 복귀 시 재적재**하는 것. 아래 "2차 조사" 절 참고.

**2026-09-09 갱신 — 진짜 원인은 "복귀"가 아니라 "진입"이었다**: 1·2차 조사는 실제로 유효한 개선이었지만(같은 조건 복귀가 4.02초→0.81초로 단축), 사용자가 체감하는 "검은화면 5초"는 다른 경로였다. 사용자는 뚜껑이 아니라 **전원 버튼**으로 절전에 들어가는데, 측정 도구가 `Lid opened`만 짝지어서 그 경로를 통째로 못 보고 있었다. 실제 원인은 gnome-shell이 sleep 억제자를 놓지 않아 절전 **진입 자체**가 5초 지연되는 것이었다. 아래 "3차 조사" 절 참고.

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

근본적으로는 RAM 30GB 중 25GB를 상시 사용하는 상태(Figma 다중 인스턴스, Docker Desktop VM(qemu), IntelliJ 등)라, 안 쓰는 앱을 정리하는 것도 체감 효과가 크다. Figma가 왜 그렇게 많이 먹는지는 [[Figma-데스크톱-앱-메모리-중복]] 참고.

## 2차 조사 (2026-09-08) — zram 도입 후에도 재발

### 측정 방법: `resume-latency` 스크립트

`journalctl`에서 `Lid opened`부터 잠금화면이 그려질 때까지의 지연과 그 사이 뜬 서비스를 표로 뽑는 명령을 만들어 둠. 5일간 23회 복귀를 표본으로 재현율(정상 19회 vs 느림 4회, 17%)을 정량화했다.

```
resume-latency "3 days ago"
```

### 원인 재확인: 스왑 고갈이 아니라 타이머 폭주였다

1차 조치(zram) 이후 디스크 스왑은 0B, zram이 5.3GB를 흡수하며 메모리 압력 자체는 낮은 상태였다. 그런데도 느린 복귀가 남아 있었던 건 다른 메커니즘 때문이었다 — **느린 4회 전부**, 복귀 직후 200ms 안에 `Persistent=true` 타이머 여러 개(`fstrim`, `dpkg-db-backup`, `fwupd-refresh`, `sysstat` 등)가 한꺼번에 밀린 실행을 발동했고, 두 번은 `gnome-shell: libinput error: event processing lagging behind by 867ms/982ms, your system is too slow` 로그로 CPU 경합이 직접 확인됐다. 커널 복귀 자체(`PM: suspend exit`)는 매번 0.2초로 정상.

`fwupd-refresh.timer`에 이미 `RandomizedDelaySec=1h`가 있었지만, 복귀 즉시 실행되는 데는 무의미했다 — **놓친 실행 시점 자체가 이미 과거**라 랜덤 지연이 과거 시점에 떨어져 소용이 없다. `Persistent=true` 타이머의 랜덤 지연은 정시 실행에는 효과가 있어도 "부팅/복귀 때 밀린 걸 몰아서 실행" 상황에는 안 먹는다.

### 진짜 급소: gnome-shell 자신이 zram에 내보낸 페이지

gnome-shell 프로세스 자체가 최대 89MB를 zram으로 내보낸 상태였다. 복귀해서 화면을 그리려면 이 페이지를 4KB 단위로 수만 번 압축해제하며 되읽어야 하는데, `vm.page-cluster=0`(1차 조치에서 zram 최적화용으로 설정)이라 readahead가 없어 폴트 하나하나가 개별 압축해제로 처리된다. 이게 검은 화면의 실체였고, 타이머 부하는 그 위에 겹친 부차적 요인이었다.

### 조치 1차 시도의 함정 — 형제 유닛끼리만 경쟁

처음엔 배치 서비스 16개에 `Nice=19` + `CPUWeight=1` 드롭인(`/etc/systemd/system/<unit>.service.d/50-resume-lowprio.conf`)만 적용했다. 그런데 systemd cgroup 계층에서 `CPUWeight`는 **같은 부모 슬라이스 안의 형제끼리만** 상대 비교된다 — 배치 서비스는 `system.slice` 소속, gnome-shell은 `user.slice` 소속이라 서로 다른 트리라 전혀 경쟁 관계가 아니었다. 우선순위를 gnome-shell에게 실제로 유리하게 걸려면 **최상위 슬라이스 단위**로 분리해야 한다.

### 최종 적용한 6가지 조치

| # | 조치 | 효과 |
|---|---|---|
| 1 | **MemoryMin 체인** 1G→1G→1G→768M→**640M**(gnome-shell까지) | gnome-shell을 reclaim 대상에서 제외 → 애초에 zram/스왑으로 안 밀려나감 |
| 2 | **`background.slice`**(CPUWeight=1) 최상위에 신설, 배치 서비스 16개를 여기로 이동 | 최상위 슬라이스 단위로 밀어야 `user.slice`의 gnome-shell에게 실제로 효과가 있음 |
| 3 | NVMe I/O 스케줄러 `none` → **`mq-deadline`** | `none`에서는 `IOSchedulingClass=idle`(ionice)이 아예 동작하지 않음. 스케줄러를 바꿔야 ionice가 먹힘. 트레이드오프: 최대 IOPS가 소폭 낮아짐(유일하게 대가가 있는 변경) |
| 4 | 타이머 9개에 `Persistent=false` | 절전 중 놓친 실행을 복귀 즉시 몰아서 재생하는 동작 자체를 제거 |
| 5 | `fstrim` 스케줄을 월요일 00:00 → **수요일 14:00**로 이동 | discard는 NVMe 장치 레벨 명령이라 nice/ionice로 못 미룸 — 유일한 대응은 절전 복귀와 안 겹치는 시간으로 옮기는 것 |
| 6 | gnome-shell 자체에 **`CPUWeight=1000`** | 남은 경합에서도 gnome-shell이 우선권을 갖도록 |

핵심은 1번(MemoryMin)과 2번(최상위 슬라이스 분리)이다 — 앞서 놓쳤던 "형제끼리만 경쟁" 함정을 바로잡은 것이 2번.

### 한계

- `MemoryMin`은 **앞으로의** reclaim만 막는다. 조치 시점에 이미 zram으로 나가 있던 89MB는 강제로 당겨올 방법이 없다(자연히 다시 쓰일 때 돌아옴). `swapoff`로 강제 회수하면 zram에 있던 6.6GB 전체가 한꺼번에 RAM으로 몰려 세션 OOM 위험이 있어(전례 있음) 시도하지 않았다.
- `fstrim`은 시간을 옮겼을 뿐 완전히 막지는 못한다. 옮긴 시간대에 우연히 절전 복귀가 겹치면 여전히 느릴 수 있다.
- `fwupd-refresh` 하나만 뜬 복귀 중에도 빠른 사례(0.59초)가 있어, 타이머 부하가 **유일한** 원인이라고 단정할 근거는 아니다. 다만 느린 4회 전부에 이 부하가 걸려 있었다.
- 세션 종료 시점까지 조치 후 배치 작업이 실제로 뜬 복귀 샘플이 없어 **미검증** 상태. `resume-latency "3 days ago"`로 며칠 뒤 재확인 필요. 서비스가 여러 개 뜬 줄이 1초 이하면 성공, 여전히 느리면 그 줄에 찍힌 서비스명이 다음 단서(특히 `fstrim`이면 예상된 한계).
- 되돌리려면 `resume-fix-revert` 명령(또는 드롭인 파일 삭제 + `systemctl daemon-reload`).

## 3차 조사 (2026-09-09) — 진짜 원인: 절전 "진입" 지연

### 측정 도구의 사각지대

`resume-latency`는 `Lid opened` 이벤트를 기준으로 복귀 지연만 쟀다. 그런데 사용자는 뚜껑이 아니라 **전원 버튼**으로 절전에 들어가고 있었다(`systemd-logind: Power key pressed short.`) — 뚜껑을 열고 닫는 게 아니라 버튼을 눌러 재우고, 다시 버튼을 눌러 깨우는 사용 패턴. 이 경로는 애초에 측정 대상이 아니었다. **사용자의 실제 조작 방식을 먼저 확인하지 않고 로그에 잡히는 이벤트만으로 가설을 세운 것이 1·2차 조사의 맹점이었다.**

### 결정적 증거

```
09:50:30.231  systemd-logind: Delay lock is active (PID 5345/gnome-shell)
              but inhibitor timeout is reached.
```

전원 버튼을 누르면 화면은 **즉시** 까매지지만, gnome-shell이 sleep 억제자(inhibitor)를 놓지 않는 한 logind는 실제 절전 진입을 미룬다. 억제자를 쥔 프로세스가 제때 놓지 않으면 logind는 `InhibitDelayMaxSec`(기본 5초) 타임아웃을 다 채운 뒤에야 강제로 절전에 들어간다. 측정 결과 전원키 입력 → 실제 절전 진입까지 **5.24초 / 5.11초**가 걸렸다.

**"검은화면 5초"의 정체**: 전원 버튼을 눌러 화면이 까매진 뒤(그러나 아직 진짜로 자고 있지 않은 5초 동안), 사용자가 "꺼졌네" 하고 다시 전원 버튼을 누르면 — 기계는 절전에 들어가는 중이라 그 사이클을 완료한 뒤에야 복귀 사이클이 시작된다. 즉 절전 진입 지연 + 복귀 한 사이클을 통째로 다시 기다리게 된다.

### 범인 특정

`systemd-inhibit --list`로 gnome-shell이 쥔 억제자 2개를 확인하고, 억제자 설명 문구와 GNOME 설정을 대조해 범인을 좁혔다: **GNOME 50의 "화면 시간"(Screen Time) 기록 기능**. `org.gnome.desktop.screen-time-limits history-enabled true`가 켜져 있으면 절전 진입 시 사용 기록을 저장하기 위해 억제자를 잡는데, 이 문구가 로그의 억제자 사유와 정확히 일치했다. 꺼보니 억제자가 즉시 사라짐(gnome-shell 억제자 2개 → 1개, 남은 하나는 화면 잠금용으로 약 0.94초에 정상 해제).

### 적용한 조치

- `gsettings set org.gnome.desktop.screen-time-limits history-enabled false` — 근본 원인 제거.
- `InhibitDelayMaxSec` 5초 → **2초**로 하향 (`/etc/systemd/logind.conf` 계열, 재부팅 후 적용). 앞으로 어떤 앱이 억제자를 오래 붙잡아도 상한을 2초로 제한하는 안전망.
  - **주의**: 라이브 Wayland 세션에서 `systemctl restart systemd-logind`로 즉시 반영을 시도하면 다음 절전 후 영구 검은 화면이 되는 전례가 있어, 재부팅 전까지는 의도적으로 미반영 상태로 둔다.
- `resume-latency`를 **[진입]/[복귀] 두 구간**으로 나누고 억제자 PID를 표시하도록 재작성. `Lid opened` 대신 `Power key pressed` 등 실제 절전 트리거를 함께 잡아 사용자의 실제 조작 경로를 놓치지 않게 함.

### 1·2차 조치와의 관계

1·2차에서 잡은 배치 타이머 경합/zram 재적재 문제는 이 조사와 별개로 실재했고 실제로 개선됐다(같은 조건 복귀 지연이 4.02초→0.81초, 이후 느린 복귀 0회). 다만 사용자가 체감한 "5초 검은화면"의 주된 원인은 그게 아니라 절전 **진입** 지연이었다 — 두 문제가 겹쳐 있었을 뿐 서로 원인은 다르다.

### 남은 한계

- `InhibitDelayMaxSec=2` 반영에는 재부팅이 필요. 재부팅 전에도 근본 원인(억제자)은 이미 제거됐으므로 효과는 있어야 하지만, 상한 자체는 재부팅 전까지 5초로 남아 있다.
- 화면 잠금용 억제자(~0.94초)는 의도적으로 유지 — 보안과 맞바꾸는 문제라 사용자 확인 후 유지 결정.
- 향후 다른 억제자가 잡힐 경우의 후보로 Electron 앱들(claude-desktop, clawd-on-desk, code, orca-ide)을 지목해뒀으나, 조사 시점엔 모두 0.16초 안에 정상적으로 억제자를 놓고 있어 혐의가 없었다.

## 출처

원본 파일 없음 — Claude Code 세션 자동 캡처 (/home/yunho), 2026-09-06, 2026-09-08, 2026-09-09.
