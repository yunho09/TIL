---
tags: [라즈베리파이, gpiozero, pwm, gpio, led]
updated: 2026-09-08
---

# gpiozero PWMLED로 밝기 제어

`LED`는 켜고 끄는 것만 가능하다. **밝기를 단계적으로 조절하려면 `PWMLED`를 쓴다.** 값은 `0.0`~`1.0` 실수이고, 하드웨어 PWM 핀이 아니어도 소프트웨어 PWM으로 동작하므로 아무 GPIO에나 붙일 수 있다.

## LED와 PWMLED

| | `LED` | `PWMLED` |
|---|---|---|
| 상태 | `on()` / `off()` / `toggle()` | 위에 더해 `value = 0.0~1.0` |
| 읽기 | `is_lit` (bool) | `value` (float), `is_lit`은 0 초과면 True |
| 용도 | 스위치 | 조광(dimming), 색 혼합 |

```python
from gpiozero import PWMLED

led = PWMLED(17)
led.value = 0.5        # 50% 밝기
led.value = 1.0        # 최대
led.off()              # 0.0 과 같음
```

## 슬라이더 값(0~255) 매핑

RGB 관례대로 UI를 0~255로 두고, 적용할 때만 실수로 바꾸는 게 편하다.

```python
STEPS = 255
led.value = slider.value() / STEPS      # 255 -> 1.0, 128 -> 0.502, 64 -> 0.251
```

UI 범위를 0~100으로 바꾸면 `STEPS`도 같이 100으로 맞춰야 한다. 둘이 어긋나면 슬라이더를 끝까지 올려도 절반 밝기에서 멈추는 식으로 조용히 틀어진다.

## PWM은 핀을 계속 점유한다

`PWMLED`는 소프트웨어 PWM을 돌리기 위해 프로그램이 살아 있는 동안 핀을 붙잡는다. 같은 핀을 쓰는 다른 프로그램을 동시에 띄우면 `GPIO busy`가 난다 → [[gpiozero-GPIO-배선-확인]]

## 출처

Claude Code 세션 (2026-09-08), Raspberry Pi 4 Model B Rev 1.5 / gpiozero 2.0.1. PyQt5 세로 슬라이더 3개로 RGB 밝기를 조절하는 조명 제어 GUI를 만들며 확인.

관련: [[Qt-Designer-PyQt5-연결]], [[라즈베리파이-PyQt5-설치-ARM64]]
