---
tags: [지문인식, fprintd, libfprint, Ubuntu, Galaxy-Book5, PAM]
updated: 2026-11-02
---

# Galaxy Book5 지문 인식 (Egis 05a5, libfprint SDCP 포크)

Galaxy Book5 960XHA의 지문 센서(Egis ETU906Axx-E, `1c7a:05a5`)는 Ubuntu 26.04 기본 libfprint(1.95.1)가 지원하지 않아 `fprintd-list`가 "No devices available"을 낸다. 보안 연결(SDCP)을 처리하는 커뮤니티 포크를 직접 빌드해 `libfprint-2.so.2`를 교체하면 등록·로그인·잠금 화면·sudo/관리자 인증까지 동작한다.

## 원인과 해법
- `fprintd`는 이미 설치돼 있어도 기본 드라이버가 이 센서를 미지원 장치로 분류한다.
- 센서가 SDCP(보안 연결 절차)를 요구한다. 포크 [TenSeventy7/libfprint-egismoc-sdcp](https://github.com/TenSeventy7/libfprint-egismoc-sdcp)가 `05a5`를 지원한다.
- 같은 기종(Kubuntu 26.04) 성공 보고: 포크 issue #6, 절차 정리 [gkrost/galaxybook5-fingerprint-linux](https://github.com/gkrost/galaxybook5-fingerprint-linux).

## 절차
1. 포크 빌드 (관리자 권한 불필요):
   `meson setup builddir --prefix=/usr --libdir=lib/x86_64-linux-gnu --buildtype=plain -Ddoc=false -Dgtk-examples=false -Dintrospection=false -Ddrivers=all && ninja -C builddir`
2. 시스템 `libfprint-2.so.2`를 빌드 결과로 교체 (원본은 `/root/libfprint-backups/`에 백업). `pkexec`는 작업 디렉터리를 바꾸므로 **절대 경로**로 실행해야 한다.
3. `fprintd-list`로 "Egis Technology (LighTuning) Match-on-Chip" 인식 확인.
4. `fprintd-enroll`로 등록 — 15단계(`enroll-stage-passed` ×15 → `enroll-completed`), 기본은 오른손 검지.
5. `fprintd-verify`로 `verify-match` 확인.
6. PAM에서 지문을 켠다(로그인·잠금 화면·sudo·polkit). 지문 → 실패/10초 후 비밀번호 순.

## 함정과 대처
- **Windows Hello 지문이 칩에 남아 있으면** 등록이 `enroll-duplicate`로 거부된다 → 포크 예제의 `clear-storage`로 칩을 비워야 한다(되돌릴 수 없고 Windows 지문도 삭제됨). Windows에서 재등록할 땐 다른 손가락을 쓸 것.
- **단계당 10초 제한**: 손가락 감지 시간 초과 시 `enroll-unknown-error`. 메시지를 기다리지 말고 1초 대고 떼기를 끊지 않고 반복, 위치를 조금씩 바꾼다.
- **시간 초과 뒤 다음 시도에서 `fprintd` 데몬이 죽는 버그**(`failed to claim device ... NoReply`, assertion 종료): 포크 PR #5의 수정 부분만 가져와 재빌드·재설치로 해결, 등록을 끊고 바로 재오픈해도 죽지 않음을 확인.
- **apt로 libfprint가 업데이트되면** 교체한 파일이 덮인다 → 업데이트 후 자동 재교체 apt 훅과 캐시된 새 파일을 둔다.
- `!` 모드(터미널 없음)에서는 `sudo -v`가 "A terminal is required"로 실패 — 지문 문제가 아님. 실제 터미널이나 polkit 창으로 확인한다.
- 부팅 직후 지문 로그인 시 로그인 키링은 비밀번호로만 풀려 비밀번호를 따로 물을 수 있다.
- 공식 드라이버가 아니라 커뮤니티 포크라 버그 보고(데몬 종료, SDCP 캐시 꼬임)가 열려 있다. 문제 시 백업 원본으로 되돌리면 된다.

## 지문 재시도 횟수 늘리기 (PAM `max-tries`)
- 기본 `pam_fprintd` 프로필은 sudo·polkit에서 **한 번 틀리면 바로 비밀번호**로 넘어간다(잠금 화면은 원래 3회).
- 패키지 파일(`/usr/share/pam-configs/fprintd`)은 건드리지 않고, 같은 내용에 `max-tries=3`만 바꾼 `/usr/share/pam-configs/fprintd-retry`를 만들어 `pam-auth-update`로 전환했다. 패키지 `Conflicts`는 쉼표+공백 구분. 이렇게 하면 `fprintd` 업데이트가 설정을 되돌리지 않는다.
- 손가락을 안 대면 10초 뒤 비밀번호로 넘어간다.

## 검증 상태
- 확인됨: 등록, `verify-match`, 잠금 화면 지문 해제, 관리자 인증 창 통과.
- 미확인: 재시도(3회)가 실제로 동작하는지는 로그로 구분 불가(사용자 관찰 의존). apt 훅의 실제 업데이트 복구는 미실행.

## 관련
- [[절전-복귀-지연-원인과-zram-도입]]

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho), 2026-11-02 — 위 포크·가이드 링크 참고
