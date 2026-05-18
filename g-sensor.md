# 1. 알고리즘 선정 배경

기존의 단순 Peak Acceleration 기반 충돌 감지 방식은  
짧은 순간의 impulse noise 또는 비충돌성 충격에도 오탐 가능성이 존재한다.

따라서 본 프로젝트에서는 실제 차량 에어백 시스템 특허에서 사용되는  
velocity signal(ΔV) 및 SPOM 기반 충돌 판단 구조를 참고하여  
프로토타입용 충돌 감지 알고리즘을 설계하였다.

참고 특허:

- US5394326A
- *Air Bag Deployment Control System and Method*

https://patents.google.com/patent/US5394326A/en

---

# 2. Velocity Signal(ΔV) 적용 이유

특허에서는 accelerometer data를 시간 적분하여  
velocity signal을 계산한다고 설명한다.

특허 원문:

> "velocity curve 34 is calculated by taking the integral of accelerometer data 30"

또한 다음과 같은 수식을 제시한다.

\[
\Delta V_i = \sum a_i \cdot \Delta t
\]

즉, 가속도 데이터를 시간 적분하여  
충돌 동안의 속도 변화량(ΔV)을 계산한다.

본 프로젝트에서는 단순 peak acceleration보다  
충돌 severity를 더 잘 반영할 수 있다고 판단하여  
ΔV 기반 판단 구조를 적용하였다.

---

# 3. SPOM 적용 이유

특허에서는 velocity signal뿐 아니라  
High Order Signal Energy(SPOM)도 함께 계산한다.

특허 수식:

\[
SPOM_i =
\sum
\left(
\frac{|a_i-a_{i-2}|}{2\Delta t}
\right)^S
\]

해당 수식은 jerk(가속도 변화율)를 기반으로 하는 metric으로,  
충돌처럼 매우 급격한 acceleration 변화를 강조하기 위한 목적을 가진다.

이를 통해:

- 일반 진동
- 짧은 impulse noise
- 실제 crash pulse

를 구분할 수 있도록 설계하였다.

---

# 4. S = 4 적용 이유

특허에서는 다음과 같이 설명한다.

> "The power S used in this embodiment ... is four (4)"

따라서 본 프로젝트에서는  
특허 실시예를 참고하여 S=4를 적용하였다.

---

# 5. 최종 충돌 판단 로직

본 프로젝트의 충돌 판단은 다음 두 조건을 동시에 만족할 경우 수행한다.

1. ΔV ≥ ΔV Threshold
2. SPOM ≥ SPOM Threshold

구현 예시는 다음과 같다.

```c
if(delta_v >= DELTA_V_TH &&
   spom >= SPOM_TH)
{
    crash_detected = true;
}
```
# 6. Threshold 선정 방법

특허에서는 threshold가 다음과 같이:

> "determined empirically"

즉 실험 기반(empirical)으로 결정된다고 설명한다.

따라서 본 프로젝트 또한:

- 정상 상황 데이터
- 비충돌 상황 데이터
- 실제 충돌 데이터

를 반복 측정하여 threshold를 결정하였다.

---

# 7. Threshold 튜닝 절차

## (1) Normal Data 측정

비충돌 상황 데이터 수집:

- 손 흔들림
- 일반 이동
- 약한 충격
- 책상 진동

---

## (2) Crash Data 측정

충돌 상황 데이터 수집:

- 모형 급정지
- 벽 충돌
- 강한 충격

---

## (3) 데이터 로깅

각 실험마다:

- Max ΔV
- Max SPOM

을 기록하였다.

---

## (4) Threshold 결정

정상 상황 최대값보다 약간 높은 값을  
threshold로 설정하였다.

---

# 8. 센서 선정 이유

사용 센서:

- LIS3DH (3축 MEMS Accelerometer)

선정 이유:

- STM32 연동 용이
- 충분한 샘플링 속도
- 충돌 pulse 측정 가능
- 프로토타입 환경에 적합

---

# 9. 시스템 구조

## STM_1 (Sensor ECU)

```text
LIS3DH Sampling
↓
LPF
↓
ΔV Calculation
↓
SPOM Calculation
↓
Threshold Comparison
↓
Emergency CAN TX
```
# 10. 설계 의의

본 프로젝트는 실제 차량 에어백 ECU 전체를 구현한 것이 아니라,  
실제 airbag deployment 특허의 signal-processing 구조를 참고하여  
프로토타입 환경에 맞게 단순화 구현한 시스템이다.

특히:

- ΔV 기반 velocity signal
- SPOM 기반 jerk-energy metric
- dual threshold discrimination

구조를 적용하여 단순 peak acceleration 방식보다  
실차 crash discrimination 구조에 가까운 알고리즘을 구현하였다.

---

# 11. 예상 질문 및 취약점 대응

# Q1. 실제 차량은 센서를 여러 개 사용하는데 왜 하나만 사용했나요?

실제 양산 차량은:

- 중앙 crash sensor
- satellite sensor
- occupancy sensor
- yaw/roll sensor

등을 함께 사용한다.

본 프로젝트는 경진대회용 프로토타입 환경을 고려하여  
단일 3축 가속도 센서를 기반으로 단순화 구현하였다.

다만 충돌 판단 구조 자체는  
실제 airbag deployment 특허의 signal-processing 구조를 참고하였다.

---

# Q2. 왜 ΔV를 사용했나요?

단순 peak acceleration 방식은  
짧은 impulse noise에도 반응 가능성이 존재한다.

반면 ΔV는:

- 충돌 동안의 누적 속도 변화량
- crash severity

를 더 잘 반영할 수 있다.

또한 특허에서도 accelerometer data를 적분하여  
velocity signal을 생성한다고 설명한다.

---

# Q3. 왜 duration 대신 SPOM을 사용했나요?

초기에는 duration 기반 filtering도 고려하였다.

그러나 실제 특허에서는  
jerk 기반 high-order signal energy(SPOM)를 사용하고 있었다.

따라서 실차 알고리즘 구조에 더 가깝게 구현하기 위해  
SPOM 기반 구조를 채택하였다.

---

# Q4. SPOM은 noise에 민감하지 않나요?

맞다.

Jerk 기반 metric은 미분 성분이 포함되므로  
noise에 민감할 수 있다.

따라서 본 프로젝트에서는:

- LPF(Low Pass Filter)
- empirical threshold tuning

을 적용하여 false trigger를 줄이도록 설계하였다.

---

# Q5. 왜 S=4를 사용했나요?

특허에서는:

> "The power S used in this embodiment ... is four (4)"

라고 설명한다.

따라서 본 프로젝트에서도  
특허 실시예를 참고하여 S=4를 적용하였다.

---

# Q6. Threshold는 어떻게 정했나요?

특허에서도 threshold는:

> "determined empirically"

즉 실험 기반으로 결정한다고 설명한다.

따라서 본 프로젝트 또한:

- 정상 상황
- 비충돌 상황
- 충돌 상황

데이터를 반복 측정하여  
ΔV 및 SPOM 분포를 기반으로 threshold를 설정하였다.

---

# Q7. 왜 CAN을 사용했나요?

충돌 감지는 low latency와 deterministic communication이 중요하다.

CAN은 차량 환경에서 널리 사용되는 실시간 네트워크이며,  
priority arbitration 구조를 통해 emergency message를 빠르게 전달할 수 있다.

따라서 충돌 감지 ECU와 Headrest ECU 간 통신에 CAN을 적용하였다.

---

# Q8. 왜 Raspberry Pi를 거치지 않았나요?

충돌 감지는 수 ms 수준의 빠른 대응이 중요하다.

따라서 emergency path는:

STM_1 → CAN → STM_2

형태로 직접 연결하였다.

Raspberry Pi는:

- AI 자세 인식
- 졸음 감지
- 상위 모드 제어

중심으로 역할을 분리하였다.

---

# Q9. 실제 차량 수준의 crash test를 수행했나요?

실제 차량 수준의 crash test는 수행하지 못하였다.

다만 프로토타입 환경에서 반복 가능한 충격 조건을 구성하여  
충돌 pulse 특성을 측정하였다.

본 프로젝트의 목적은:

실차 deployment 검증이 아니라,

실제 airbag deployment 특허의 crash discrimination 구조를  
임베디드 프로토타입 환경에서 구현하는 것이다.

---

# Q10. 현재 구조의 한계점은 무엇인가요?

현재 구조는:

- 단일 가속도 센서 기반
- 소형 프로토타입 환경

이라는 한계를 가진다.

따라서 실제 차량의:

- chassis deformation
- crumple zone
- multi-point crash sensing

등은 반영하지 못한다.

다만 본 프로젝트는  
실차 signal-processing 철학을 프로토타입 환경에 적용하는 데 목적을 두었다.
