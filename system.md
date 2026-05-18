# 시스템 역할 분리 및 RTOS/CAN 구조 설계

## 1. 전체 시스템 구조

본 프로젝트는 총 3개의 ECU 역할로 시스템을 분리하였다.

```text
[ Raspberry Pi : AI / Supervisor ECU ]
    - 카메라 입력
    - 자세 / 졸음 인식
    - 사용자 상태 분석
    - 상위 정책 판단
    - CAN 송신

                CAN BUS

[ STM_1 : Sensor Safety ECU ]
    - G-Sensor 입력
    - 압력센서 입력
    - ΔV / SPOM 기반 충돌 판단
    - Emergency Trigger 생성
    - CAN 송신

                CAN BUS

[ STM_2 : Headrest Control ECU ]
    - Mode Request 수신
    - Emergency Trigger 수신
    - 상태머신 실행
    - Servo / Aircell / Haptic / Lock 제어
```

---

# 2. 역할 분리 이유

초기에는 모든 판단을 Raspberry Pi에서 수행하는 구조를 고려하였다.

하지만 사고 상황에서는 다음 문제가 발생할 수 있다.

```text
STM_1 → Raspberry Pi → STM_2
```

즉:

- Raspberry Pi를 반드시 거쳐야 함
- Linux scheduling delay 존재 가능
- AI inference latency 존재 가능
- Camera processing load 존재 가능

따라서 실시간성이 중요한 충돌 감지는 Raspberry Pi를 거치지 않고 STM 간 직접 CAN 통신으로 처리하도록 구성하였다.

---

# 3. Raspberry Pi 역할

Raspberry Pi는 고수준 판단(High-Level Decision)을 담당한다.

예시:

- 졸음 여부 판단
- 사용자가 앞으로 숙였는지 판단
- 자세 분석
- 상위 mode command 생성
- UI / 로그 관리

예:

```text
"사용자가 앞으로 숙였다"
→ target_mode = SOFT_CONTACT
```
---

# 4. STM_1 역할

STM_1은 Safety Sensor ECU 역할을 수행한다.

담당 기능:

- LIS3DH G-Sensor 처리
- 압력센서 처리
- ΔV 계산
- SPOM 계산
- 충돌 판단
- Emergency CAN 송신

---

# 5. STM_2 역할

STM_2는 실제 Headrest 제어를 담당한다.

담당 기능:

- 상태머신(State Machine)
- Servo 제어
- Aircell 제어
- Haptic Motor 제어
- Locking 제어
- Emergency 최우선 처리

---

# 6. 상태머신(State Machine) 구조

STM_2 내부에는 상태머신을 구성한다.

```text
NORMAL / HOVERING
        ↓
SOFT_CONTACT
        ↓
HOLD
        ↓
EMERGENCY
```

각 상태는:

- Raspberry Pi의 Mode Request
- STM_1의 Emergency Trigger
- 내부 timeout 조건

등에 의해 전이된다.

---

# 7. 상태 전환 우선순위

Emergency 상황은 최우선으로 처리한다.

우선순위는 다음과 같이 구성하였다.

| 우선순위 | 이벤트 |
|---|---|
| 1 | STM_1 Emergency CAN |
| 2 | STM_2 자체 Fault / Fail-safe |
| 3 | Raspberry Pi Mode Request |
| 4 | 기본 Hovering 유지 |

즉 충돌 발생 시 Raspberry Pi 판단과 관계없이 즉시 Emergency 상태로 진입한다.

---

# 8. CAN 구조 설계 이유

본 프로젝트는 각 ECU를 CAN BUS로 연결하였다.

CAN을 선택한 이유:

- 차량 환경에서 널리 사용됨
- 실시간성 우수
- Priority Arbitration 지원
- Multi-ECU 구조에 적합
- Fault Isolation 용이

특히 Emergency Trigger는 low latency가 중요하므로 CAN 기반 direct ECU communication 구조를 채택하였다.

---

# 9. Raspberry Pi에서 필요한 하드웨어

Raspberry Pi는 기본적으로 CAN Controller가 내장되어 있지 않다.

따라서 다음과 같은 외부 CAN 인터페이스가 필요하다.

## 권장 구성

```text
Raspberry Pi
+
MCP2515 기반 CAN HAT
+
CAN Transceiver
```

예:

- MCP2515 CAN HAT
- Waveshare CAN HAT
- USB-CAN Adapter

---

# 10. STM32F103RB에서 필요한 하드웨어

STM32F103RB는 내부 CAN Controller를 지원한다.

따라서:

```text
별도 CAN Controller 불필요
```

하지만:

```text
CAN Transceiver는 반드시 필요
```

하다.

권장 CAN Transceiver:

- SN65HVD230
- TJA1050
- MCP2551

추천:

```text
SN65HVD230
```

이유:

- 3.3V 계열
- STM32와 연결 편리

---

# 11. 센서 추가 시 구조

센서는 직접 CAN ID를 가지는 것이 아니다.

구조는 다음과 같다.

```text
센서
↓
STM_1이 센서 읽기
↓
STM_1이 판단 수행
↓
CAN 메시지 생성
```

즉 CAN ID는:

```text
센서 종류가 아니라
메시지 종류
```

를 의미한다.

예:

| CAN ID | 의미 |
|---|---|
| 0x080 | Crash Emergency |
| 0x100 | Pressure Event |
| 0x200 | Raspberry Pi Mode Request |
| 0x300 | Headrest Status |

따라서 새로운 센서를 추가하더라도:

```text
STM 내부 처리 로직만 추가하면 됨
```

구조 확장이 용이하다.
