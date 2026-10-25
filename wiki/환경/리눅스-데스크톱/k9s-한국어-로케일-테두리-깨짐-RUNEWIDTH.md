---
tags: [k9s, lazydocker, 로케일, 터미널, zsh, kubectl]
updated: 2026-10-25
---

한국어 로케일에서 **k9s** 화면이 깨지는 원인은 TUI 라이브러리가 `─ │` 같은 테두리 문자를 **두 칸**으로 계산하는데 터미널은 **한 칸**으로 그리기 때문이다. `~/.zshrc`에 `export RUNEWIDTH_EASTASIAN=0`을 넣어 한 칸으로 세게 하면 해결된다. 같은 방식으로 그리는 lazydocker에도 적용된다.

## 증상

- 테두리가 `─ ─ ─`처럼 띄엄띄엄 나온다.
- 제목이 `o s d f u t`처럼 글자 사이가 벌어져 끊긴다.
- 선택한 줄이 윗줄에 겹쳐 보인다.

## 원인

- 한국어(East Asian) 로케일에서는 문자 폭 계산 라이브러리(go-runewidth)가 **모호한 폭(ambiguous width)** 문자를 2칸으로 취급한다.
- 박스 드로잉 문자(`─ │` 등)가 여기에 속한다. 터미널은 1칸으로 그리므로 계산과 실제 그림이 어긋난다.

## 해결

- `~/.zshrc`에 추가:
  ```
  export RUNEWIDTH_EASTASIAN=0
  ```
- 새 셸에서 k9s를 띄워 테두리가 정상으로 이어지는 것을 확인했다.
- **이미 떠 있는 k9s는 예전 환경변수로 실행된 상태**라 다시 띄워야 한다. `:q`로 나가면 k9s 안에서 건 포트포워딩(PF 칸의 ●)도 같이 끊긴다.

## 같이 알아둘 kubectl port-forward 규칙

- `port-forward pod/nginx 80` → 1024 미만 포트는 root만 열 수 있어 권한 오류.
- `port-forward pod/nginx 8080` → 포트 하나만 쓰면 **로컬:파드 같은 번호**로 해석된다. nginx가 80에서 듣고 있으면 연결이 안 된다. `로컬:파드` 형태로 써야 한다.
- 로컬 8080이 이미 쓰이고 있으면(예: 다른 Spring 서버) 다른 로컬 포트로 연결: `kubectl port-forward pod/nginx 8081:80` → `http://localhost:8081`.

## Vim 이름 없는 버퍼에서 못 나갈 때

- 저장할 이름이 없는 버퍼에 입력이 있으면 `:q`가 막힌다.
- 버리고 종료: `:qa!`. 남기려면 `:w ~/memo.txt` 후 `:qa`.

## 출처

- Claude Code 세션 자동 캡처 (2026-10-25)
