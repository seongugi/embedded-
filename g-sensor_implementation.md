# 충돌 감지 알고리즘 구현 구조

## 1. 전체 구조

본 프로젝트의 충돌 감지 알고리즘은 실제 airbag deployment 특허의 signal-processing 구조를 참고하여 다음과 같이 구성하였다.

```text
LIS3DH Sampling
↓
Software LPF
↓
Dominant Axis Selection
↓
ΔV Calculation
↓
SPOM Calculation
↓
Threshold Comparison
↓
Emergency CAN TX
```

## 2. Dominant Axis를 사용하는 이유

실제 충돌 상황에서는 특정 방향 축에서 가장 큰 acceleration이 발생하는 경우가 많다.

따라서 본 프로젝트에서는 매 샘플마다 `|ax|`, `|ay|`, `|az|` 중 가장 큰 값을 선택하여 충돌 판단에 사용하였다.

이를 통해 충돌 방향에 대한 민감도를 높이고, 작은 축 noise의 영향을 줄이며, 단일 축 기반으로 threshold tuning을 단순화할 수 있다.

## 3. 상태 구성

| State | 설명 |
|---|---|
| NORMAL | 평상시 충돌 감지 수행 |
| CRASH_DETECTED | 충돌 확정 상태 |
| LOCKOUT | 중복 감지 방지 상태 |

## 4. 상태 전이

```text
NORMAL
  │
  │ ΔV >= DELTA_V_TH
  │ && SPOM >= SPOM_TH
  ▼
CRASH_DETECTED
  │
  │ Emergency CAN TX
  ▼
LOCKOUT
  │
  │ lockout time elapsed
  ▼
NORMAL
```

## 5. NORMAL 상태

평상시 충돌 감지를 수행하는 상태이다.

매 1ms마다 다음 작업을 수행한다.

```text
LIS3DH Read
↓
LPF
↓
Dominant Axis Selection
↓
ΔV Calculation
↓
SPOM Calculation
↓
Threshold Comparison
```

다음 조건을 만족할 경우 충돌로 판단한다.

```text
ΔV >= DELTA_V_TH
AND
SPOM >= SPOM_TH
```

## 6. CRASH_DETECTED 상태

충돌이 확정된 상태이다.

이 상태에서는 `Emergency CAN Message`를 전송한다.

CAN 송신 후 즉시 `LOCKOUT` 상태로 전이한다.

## 7. LOCKOUT 상태

충돌 직후에는 acceleration 값이 계속 크게 변화할 수 있다.

따라서 일정 시간 동안 추가 충돌 판단을 막기 위해 `LOCKOUT` 상태를 사용한다.

LOCKOUT 시간이 종료되면 다시 `NORMAL` 상태로 복귀한다.

## 8. Velocity Signal, ΔV

특허에서는 accelerometer data를 시간 적분하여 velocity signal을 계산한다고 설명한다.

본 프로젝트에서는 다음 수식을 사용하였다.

$\Delta V_i = \sum_{k=0}^{i} a_k \cdot \Delta t$

- $a_k$ : k번째 샘플의 acceleration
- $\Delta t$ : 샘플링 시간 간격
- 의미: 충돌 동안의 누적 속도 변화량

## 9. SPOM, High Order Signal Energy

특허에서는 jerk 기반 high-order signal energy를 사용한다.

본 프로젝트에서는 다음 수식을 사용하였다.

$SPOM_i = \sum_{k=2}^{i} \left( \frac{|a_k-a_{k-2}|}{2\Delta t} \right)^S$

- $a_k-a_{k-2}$ : acceleration 변화량
- $\frac{|a_k-a_{k-2}|}{2\Delta t}$ : jerk, 즉 가속도 변화율
- $S$ : high-order power, 본 프로젝트에서는 $S=4$ 적용
- 의미: 충돌의 급격성과 intensity를 강조하는 metric

## 10. Threshold 기반 충돌 판단

본 프로젝트는 다음 두 조건을 동시에 만족할 경우 충돌로 판단한다.

```text
1. ΔV >= DELTA_V_TH
2. SPOM >= SPOM_TH
```

구현 형태는 다음과 같다.

```c
if (delta_v >= DELTA_V_TH &&
    spom >= SPOM_TH)
{
    crash_detected = true;
}
```

## 11. LPF 적용 이유

SPOM은 jerk 기반 metric이므로 noise에 민감할 수 있다.

따라서 본 프로젝트에서는 STM32 내부에서 Software LPF를 적용한 뒤 ΔV 및 SPOM 계산을 수행하였다.

본 프로젝트에서는 1차 IIR LPF를 사용하였다.

```c
filtered = alpha * prev + (1.0f - alpha) * raw;
```

## 12. Threshold 선정 방식

특허에서는 threshold가 `determined empirically`, 즉 실험 기반으로 결정된다고 설명한다.

따라서 본 프로젝트 또한 정상 상황 데이터, 비충돌 상황 데이터, 충돌 상황 데이터를 반복 측정하여 threshold를 결정하였다.

## 13. Sliding Window 적용 이유

기존의 단순 누적 방식에서는 다음과 같이 ΔV와 SPOM이 계속 증가하는 문제가 있다.

```c
delta_v += delta_v_sample;
spom += spom_sample;
```

이 경우 실제 충돌이 발생하지 않아도 작은 acceleration noise나 진동이 장시간 누적되면 threshold를 초과할 수 있다.

따라서 본 프로젝트에서는 최근 일정 시간 구간의 crash pulse만 판단에 사용하기 위해 Sliding Window 방식을 적용하였다.

본 프로젝트에서는 1kHz 샘플링 기준으로 `WINDOW_SIZE = 100`을 사용하여 최근 100ms 구간의 ΔV와 SPOM만 유지한다.

```c
#define WINDOW_SIZE 100   // 1kHz 기준 100ms
```

Sliding Window는 새 샘플을 추가하기 전에 가장 오래된 샘플을 제거하는 방식으로 동작한다.

```c
delta_v -= delta_v_window[window_idx];
spom    -= spom_window[window_idx];

delta_v_window[window_idx] = delta_v_sample;
spom_window[window_idx] = spom_sample;

delta_v += delta_v_sample;
spom    += spom_sample;
```

이를 통해 ΔV와 SPOM은 시스템 시작 이후 전체 누적값이 아니라, 최근 100ms 동안의 충돌 특성만 반영하게 된다.

즉, 본 알고리즘은 다음과 같은 기준으로 충돌을 판단한다.

```text
최근 100ms 동안의 ΔV >= DELTA_V_TH
AND
최근 100ms 동안의 SPOM >= SPOM_TH
```

Sliding Window 적용을 통해 다음 효과를 얻을 수 있다.

- ΔV와 SPOM의 무한 누적 방지
- 장시간 noise 누적에 의한 오탐 감소
- 실제 crash pulse와 같이 짧은 시간 구간에서 발생하는 충격 특성 반영
- threshold tuning 기준 명확화

## 14. 최종 구현 코드

### 14.1 상수 및 상태 정의

```c
#define DT_SEC          0.001f
#define G_TO_MS2        9.81f
#define LPF_ALPHA       0.8f

#define WINDOW_SIZE     100     // 1kHz 기준 100ms

#define DELTA_V_TH      0.8f
#define SPOM_TH         500000.0f

#define LOCKOUT_MS      1000

typedef enum {
    CRASH_NORMAL = 0,
    CRASH_DETECTED,
    CRASH_LOCKOUT
} CrashState_t;

typedef struct {
    float ax;
    float ay;
    float az;
} Accel_t;

static CrashState_t crash_state = CRASH_NORMAL;

static float f_ax = 0.0f;
static float f_ay = 0.0f;
static float f_az = 0.0f;

static float a_hist[3] = {0.0f, 0.0f, 0.0f};

static float delta_v_window[WINDOW_SIZE] = {0.0f};
static float spom_window[WINDOW_SIZE] = {0.0f};

static float delta_v = 0.0f;
static float spom = 0.0f;

static uint16_t window_idx = 0;
static uint16_t lockout_ms = 0;
```

### 14.2 LPF 함수

```c
static float LPF_Update(float prev, float raw)
{
    return LPF_ALPHA * prev + (1.0f - LPF_ALPHA) * raw;
}
```

### 14.3 Dominant Axis 선택 함수

```c
static float Get_Dominant_Accel(float ax, float ay, float az)
{
    float abs_x = fabsf(ax);
    float abs_y = fabsf(ay);
    float abs_z = fabsf(az);

    if (abs_x >= abs_y && abs_x >= abs_z) return abs_x;
    if (abs_y >= abs_x && abs_y >= abs_z) return abs_y;
    return abs_z;
}
```

### 14.4 Crash Metric Reset 함수

```c
static void CrashMetric_Reset(void)
{
    delta_v = 0.0f;
    spom = 0.0f;

    window_idx = 0;

    for (int i = 0; i < WINDOW_SIZE; i++) {
        delta_v_window[i] = 0.0f;
        spom_window[i] = 0.0f;
    }

    a_hist[0] = 0.0f;
    a_hist[1] = 0.0f;
    a_hist[2] = 0.0f;
}
```

### 14.5 충돌 감지 업데이트 함수

```c
void CrashDetection_Update_1ms(Accel_t raw)
{
    float a_dom_g;
    float a_dom_ms2;

    float jerk;
    float j2;
    float j4;

    float delta_v_sample;
    float spom_sample;

    switch (crash_state)
    {
    case CRASH_NORMAL:

        // 1. LPF 적용
        f_ax = LPF_Update(f_ax, raw.ax);
        f_ay = LPF_Update(f_ay, raw.ay);
        f_az = LPF_Update(f_az, raw.az);

        // 2. Dominant Axis 선택
        a_dom_g = Get_Dominant_Accel(f_ax, f_ay, f_az);

        // 3. g → m/s² 변환
        a_dom_ms2 = a_dom_g * G_TO_MS2;

        // 4. ΔV sample 계산
        delta_v_sample = a_dom_ms2 * DT_SEC;

        // 5. SPOM sample 계산
        jerk = fabsf(a_dom_ms2 - a_hist[0]) / (2.0f * DT_SEC);

        j2 = jerk * jerk;
        j4 = j2 * j2;

        spom_sample = j4;

        // 6. Sliding Window Update
        delta_v -= delta_v_window[window_idx];
        spom    -= spom_window[window_idx];

        delta_v_window[window_idx] = delta_v_sample;
        spom_window[window_idx] = spom_sample;

        delta_v += delta_v_sample;
        spom    += spom_sample;

        // 7. Window Index Update
        window_idx++;

        if (window_idx >= WINDOW_SIZE) {
            window_idx = 0;
        }

        // 8. History Update
        a_hist[0] = a_hist[1];
        a_hist[1] = a_hist[2];
        a_hist[2] = a_dom_ms2;

        // 9. Threshold 판단
        if (delta_v >= DELTA_V_TH &&
            spom >= SPOM_TH)
        {
            crash_state = CRASH_DETECTED;
        }

        break;

    case CRASH_DETECTED:

        // Emergency CAN TX
        Send_Crash_Emergency_CAN();

        // Metric Reset
        CrashMetric_Reset();

        // LOCKOUT 진입
        lockout_ms = 0;
        crash_state = CRASH_LOCKOUT;

        break;

    case CRASH_LOCKOUT:

        lockout_ms++;

        if (lockout_ms >= LOCKOUT_MS) {
            crash_state = CRASH_NORMAL;
        }

        break;
    }
}
```

### 14.6 CAN Emergency 송신 함수

```c
void Send_Crash_Emergency_CAN(void)
{
    CAN_TxHeaderTypeDef txHeader;
    uint8_t txData[1];
    uint32_t txMailbox;

    txHeader.StdId = 0x080;
    txHeader.IDE = CAN_ID_STD;
    txHeader.RTR = CAN_RTR_DATA;
    txHeader.DLC = 1;

    txData[0] = 0x01; // Crash Emergency Flag

    for (int i = 0; i < 3; i++) {

        HAL_CAN_AddTxMessage(
            &hcan,
            &txHeader,
            txData,
            &txMailbox
        );

        HAL_Delay(10);
    }
}
```

## 15. CAN Emergency 메시지

충돌 판단 시 다음 CAN 메시지를 송신한다.

| 항목 | 값 |
|---|---|
| CAN ID | 0x080 |
| DLC | 1 |
| Byte0 | Crash Emergency Flag |

`Byte0 = 0x01`이면 충돌 Emergency 발생을 의미한다.

이를 통해 STM_2, Headrest ECU가 즉시 Emergency Mode를 수행하도록 구성하였다.
