# 1. Raspberry Pi 4 측 구성

## 사용 부품

### Waveshare Isolated CAN HAT
(MCP2515 + SN65HVD230 기반)

공식:
- https://www.waveshare.com/2-ch-can-hat.htm

---

# 왜 이걸 사용하는가?

## 1) Raspberry Pi 4에는 CAN Controller가 없음

Raspberry Pi 4는:
- SPI
- UART
- I2C

는 내장되어 있지만,
CAN Controller는 내장되어 있지 않다.

따라서 외부 CAN Controller가 필요하다.

---

## 2) MCP2515가 CAN Controller 역할 수행

HAT 내부의:
- MCP2515

가:
- CAN 프레임 생성
- Arbitration
- CAN 송수신 제어

를 담당한다.

그리고 Raspberry Pi와는:
- SPI

로 연결된다.

---

## 3) SN65HVD230이 CAN 물리 신호 처리

CAN Bus는:
- CAN_H
- CAN_L

차동 신호를 사용한다.

SN65HVD230은:
- CAN Transceiver 역할
- 실제 CAN 전압 신호 처리

를 담당한다.

---

## 4) SN65HVD230을 선택한 이유

이유:
- 3.3V 기반
- Raspberry Pi GPIO와 전압 호환 좋음
- STM32와도 전압 호환 좋음
- 저전력
- 자료 풍부

반면:
- TJA1050
- MCP2551

같은 칩은 대부분 5V 기반이라 현재 구성에는 덜 적합하다.

---

## 5) Isolated CAN HAT를 사용하는 이유

일반 저가 MCP2515 모듈보다 좋은 이유:

### 포함 기능
- 절연 회로(Isolation)
- ESD 보호
- TVS 보호
- 안정적인 전원 설계
- 종단저항

즉:
- 노이즈 내성 증가
- 안정성 증가
- 차량 환경과 유사한 구조 구성 가능

---

## 6) Linux SocketCAN 사용 가능

Linux에서:

```bash
candump can0
cansend can0 123#11223344
```

같은 명령 사용 가능.

즉:
- Linux 네트워크 관점에서 CAN 학습 가능
- Wireshark CAN 분석 가능
- ROS CAN 브릿지 가능

---

# 2. STM32F103RB 측 구성

## 사용 부품

### STM32F103RB
+
### SN65HVD230 CAN Transceiver Module

---

# 왜 이렇게 구성하는가?

## 1) STM32F103RB는 CAN Controller 내장

STM32F103RB 내부에는:
- bxCAN Controller

가 이미 들어있다.

따라서:
- MCP2515 같은 외부 CAN Controller 불필요

이다.

---

## 2) 하지만 CAN Transceiver는 필요

STM32 내부 CAN은:
- 디지털 CAN 논리 처리

만 수행한다.

실제:
- CAN_H
- CAN_L

전압 신호 처리는 외부 Transceiver가 담당해야 한다.

그래서:
- SN65HVD230

을 사용한다.

---

## 3) SN65HVD230을 사용하는 이유

이유:
- STM32와 같은 3.3V 계열
- 전압 호환 매우 좋음
- 배선 단순
- 저전력
- CAN 실습 자료 풍부

---

# 3. 케이블 구성

# 사용 케이블

## 추천:
### CAT5e 랜선 일부 사용

이유:
- 내부에 트위스티드 페어(Twisted Pair) 존재
- CAN 차동 신호에 적합
- 노이즈 내성 우수
- 저렴
- 구하기 쉬움

---

# 실제 연결 방식

| 랜선 선 | 연결 |
|---|---|
| 주황 | CAN_H |
| 주황흰색 | CAN_L |
| 갈색 | GND |

예시일 뿐이고:
- 서로 꼬여 있는 한 쌍
을 CAN_H/CAN_L로 사용하는 게 핵심이다.

---

# 왜 트위스티드 페어를 사용하는가?

CAN은:
- Differential Signal(차동 신호)

를 사용한다.

그래서:
- 외부 노이즈 제거
- 신호 안정성 향상

을 위해 꼬인 선이 매우 중요하다.

---

# 초기 테스트는?

초기에는:
- 듀퐁 점퍼선

으로도 가능하다.

하지만:
- 노이즈 내성
- 안정성

면에서 CAT5e 사용이 더 좋다.

---

# 4. 종단저항(Termination Resistor)

CAN Bus 양 끝에는:
- 120Ω 저항

이 필요하다.

구조:

```text
120Ω                      120Ω
 |                          |
[Pi]====================[STM32]
```

---

# 현재 구성에서는?

노드가:
- Raspberry Pi
- STM32

2개뿐이므로:

## 양 끝 각각 120Ω
→ 총 2개 필요

이다.

---

# 다행히 대부분 이미 포함됨

## Waveshare CAN HAT
- 종단저항 ON/OFF 스위치 존재

## SN65HVD230 모듈
- 일부 제품은 120Ω 내장

따라서:
- 구매 후 활성 여부만 확인하면 된다.

---

# 5. 전체 최종 구조

```text
[Raspberry Pi 4]
 Linux + SocketCAN
        │
    MCP2515
        │
   SN65HVD230
        │
==================== CAN BUS ====================
        │
   SN65HVD230
        │
 STM32F103RB
 (내장 bxCAN)
```

---

# 6. 최종 부품 리스트

| 용도 | 부품 |
|---|---|
| Linux CAN 인터페이스 | Waveshare Isolated CAN HAT |
| STM32 CAN 물리 계층 | SN65HVD230 Module |
| CAN 배선 | CAT5e 랜선 |
| CAN 종단 | 120Ω × 2 |
