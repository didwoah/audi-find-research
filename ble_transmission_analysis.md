# Audi-Find: Mel-Spectrogram 변환 · 암호화 · BLE 전송 분석

> 10초 오디오 스냅샷을 Galaxy Buds에서 스마트폰으로 전송하는 전 과정 분석.
> **하드웨어 기준**: Galaxy Buds2 Pro (BES2700YP SoC, teardown 확인) / Buds3 Pro (SoC 미공개).
> 연산 시간 수치는 공개 스펙 기반 추정치이며, 실측값이 아님을 명시한다.
> 상세 하드웨어 스펙은 buds_hardware.md 참조.

---

## 1. 원본 오디오 용량

Qwen AuT 입력 스펙(16kHz 리샘플링, 128ch Mel-Spectrogram)에 맞춰 처리한다.

| 항목 | 수치 | 근거 |
|---|---|---|
| 샘플링 레이트 | 16,000 Hz | Qwen3.5-Omni Technical Report §2.3: "resample the waveform to 16 kHz" |
| 비트 깊이 | 16-bit PCM | PCM 오디오 표준 (CD 품질 기준) |
| 채널 | 모노 | ANC 외부 마이크 단일 채널 수집 |
| 수집 구간 | 10초 | audio_window.md 확정 사항 |
| **Raw PCM 용량** | **320 KB** | 아래 계산 참조 |

```
[Raw PCM 용량 계산]
샘플 수  = 16,000 Hz × 10 s = 160,000 샘플
바이트   = 160,000 샘플 × 2 B/샘플 (16-bit) = 320,000 B = 312.5 KB ≈ 320 KB
```

---

## 2. Mel-Spectrogram 변환

### 파라미터 (Qwen AuT 스펙)

| 항목 | 수치 | 근거 |
|---|---|---|
| 윈도우 크기 | 25ms → **400샘플** | Qwen3.5-Omni Technical Report §2.3: "25 ms window" / 16,000 Hz × 0.025 s = 400 |
| 홉 크기 | 10ms → **160샘플** | Qwen3.5-Omni Technical Report §2.3: "10 ms hop size" / 16,000 Hz × 0.010 s = 160 |
| Mel 필터 채널 수 | **128** | Qwen3.5-Omni Technical Report §2.3: "128-channel mel-spectrogram" |
| 총 프레임 수 (10초) | ≈ **1,000 프레임** | 아래 계산 참조 |

```
[프레임 수 계산]
총 샘플 수 = 160,000
프레임 수  = ⌈(160,000 − 400) / 160⌉ + 1
           = ⌈159,600 / 160⌉ + 1
           = 998 + 1
           = 999 ≈ 1,000 프레임
```

### 출력 행렬 및 용량

```
[출력 행렬 크기]
128 채널 × 1,000 프레임 = 128,000 값
```

| 데이터 타입 | 바이트/값 | 총 용량 | 계산 | Raw PCM(320 KB) 대비 |
|---|---|---|---|---|
| float32 | 4 B | **500 KB** | 128,000 × 4 = 512,000 B ≈ 500 KB | **+56% (더 큼)** |
| float16 | 2 B | **250 KB** | 128,000 × 2 = 256,000 B = 250 KB | **−22% 감소** |
| int8 (선형 양자화) | 1 B | **125 KB** | 128,000 × 1 = 128,000 B = 125 KB | **−61% 감소** |

> **결론**: Mel-Spectrogram을 float32로 그대로 전송하면 Raw PCM보다 오히려 크다. Mel-Spectrogram 전송의 목적은 용량 압축이 아니라 (1) **Raw 오디오 비전송(프라이버시)** 과 (2) **클라우드에서 AuT 인코더가 기대하는 포맷을 그대로 전달** 이다. 용량 절감이 필요하다면 float16이 정밀도와 크기의 균형점이다. Audi-Find에서는 **float16 채택 권장**.

### 변환 소요 시간 (BES2700YP 기준)

BES2700YP의 연산 유닛 구성:
- **CPU**: ARM Cortex-M55 — Helium(MVE) SIMD 지원. 범용 연산 담당.
- **DSP**: Tensilica HiFi 4 — 오디오 DSP 전용 코어. FFT, FIR/IIR 필터, 행렬 연산에 특화.

Mel-Spectrogram 파이프라인은 FFT와 필터뱅크 행렬 곱으로 구성되어 HiFi 4 DSP에서 처리하기 적합하다.

| 연산 단계 | 담당 유닛 | 추정 소요 시간 | 추정 근거 |
|---|---|---|---|
| STFT (400pt FFT × 1,000회) | HiFi 4 DSP | **~5 ms** | Tensilica HiFi4: 400pt 복소 FFT 기준 단일 코어 ~0.003–0.005 ms/회 × 1,000회 ≈ 3–5 ms. [Cadence HiFi4 벤치마크] |
| Mel 필터뱅크 적용 (128 × 201 행렬 × 1,000회) | HiFi 4 DSP | **~3 ms** | 128×201 행렬-벡터 곱 = 25,728 MAC/프레임. HiFi4 단일 사이클 2 MAC 처리, ~200 MHz 가정 → ~0.064 ms/프레임 × 1,000 ≈ 64 ms. 단, HiFi4 SIMD 병렬화(VFPU) 적용 시 10–20× 가속 → ~3–6 ms로 추정. |
| float16 변환 및 버퍼 기록 | Cortex-M55 | **~2 ms** | 128,000회 float32→float16 변환. Cortex-M55 Helium VCVT 명령어: 8개/사이클. 128,000 / 8 = 16,000 사이클 / 200 MHz ≈ 0.08 ms. 버퍼 DMA 기록 포함 여유분 ~2 ms. |
| **합계** | | **~10 ms** | |

> **주의**: BES2700YP의 실제 클럭 속도와 HiFi4 설정 파라미터(SIMD 폭, 캐시 구성)는 공식 비공개다. 위 수치는 Cadence HiFi4 공개 벤치마크 및 동급 SoC(BES2500 계열, Qualcomm QCC 계열) 구현 사례를 기반으로 한 추정치다. **실측 검증 필요.**

---

## 3. 암호화

### 암호화 알고리즘 선택

TLS 1.3 핸드셰이크 완료 후 레코드 계층에서 AES-256-GCM으로 Mel-Spectrogram 페이로드를 암호화한다.

> TLS 1.3(RFC 8446)은 레코드 계층 암호화에 AEAD(Authenticated Encryption with Associated Data) 알고리즘만 허용하며, AES-256-GCM이 권장 선택지다.

### BES2700YP에서의 AES 처리

BES2700YP의 하드웨어 AES 가속기 탑재 여부는 공식 데이터시트에 명시되지 않았다. 세 가지 경우를 구분한다.

| 조건 | 추정 처리 속도 | 250 KB 암호화 시간 | 계산 | 근거 |
|---|---|---|---|---|
| 하드웨어 AES 가속기 탑재 시 | ~100 MB/s | **~2.5 ms** | 250 KB / 100,000 KB/s = 0.0025 s | 동급 임베디드 SoC 하드웨어 AES 일반 성능 범위 [ARM CryptoCell-312 기준 ~150 MB/s] |
| Cortex-M55 Helium 소프트웨어 AES | ~30–50 MB/s | **~5–8 ms** | 250 KB / 40,000 KB/s ≈ 0.006 s | ARMv8.1-M Helium AES 명령어(VAESEQ/VAESIMCQ) 활용 시 추정. [ARM Helium Technology Reference Manual] |
| 소프트웨어 폴백 (Helium 미활용) | ~5–10 MB/s | **~25–50 ms** | 250 KB / 7,500 KB/s ≈ 0.033 s | Cortex-M 계열 bare-metal AES-256 구현 일반 성능. [mbedTLS 벤치마크 기준 Cortex-M4 @168MHz: ~6 MB/s] |

> Cortex-M55는 ARMv8.1-M 아키텍처로 Helium(MVE) 명령어셋을 지원하며, AES 연산에 VAESEQ/VAESIMCQ 등 Helium AES 명령어 활용이 가능하다. 단, 컴파일러 및 펌웨어 구현에 따라 실제 경로가 달라진다.
>
> **보수적 추정치**: 소프트웨어 AES + Helium 활용 기준 **~10 ms** 이내.

암호화 오버헤드는 어떤 경우에도 수십 ms 이하이며, 전체 파이프라인에서 비지배적이다.

---

## 4. BLE 전송

### 모델별 BLE 스펙

| 모델 | BT 버전 | PHY | 이론적 최대 처리량 | 근거 |
|---|---|---|---|---|
| Galaxy Buds2 Pro | **BLE 5.3** | 2M PHY | 2 Mbps | BES2700YP 데이터시트: "dual-mode BT 5.3" / BLE 2M PHY 이론값 = 2 Mbps [Bluetooth Core Spec 5.x] |
| Galaxy Buds3 Pro | **BLE 5.4** | 2M PHY | 2 Mbps | Samsung 공식 스펙 / BLE 5.4 최대 PHY 처리량 동일 [Bluetooth Core Spec 5.4] |

> BLE 5.4가 5.3 대비 처리량을 증가시키지는 않는다. 5.4의 주요 개선은 채널 분류(Channel Classification), LE PAST 개선 등이며, 최대 PHY 처리량은 동일하게 2 Mbps다.

### 실효 처리량

BLE 2M PHY의 이론적 최대치 2 Mbps에서 프로토콜 오버헤드를 제외한 실효 처리량 계산:

```
[BLE 패킷 구조 및 오버헤드 계산 (DLE 활성, 2M PHY 기준)]

패킷 구성:
  Preamble        :  2 B  (2M PHY 기준)
  Access Address  :  4 B
  LL Header       :  2 B
  L2CAP Header    :  4 B
  ATT Header      :  3 B
  페이로드 (DLE)  : 244 B  (DLE 최대값 251B - ATT헤더 3B - L2CAP헤더 4B)
  MIC             :  4 B  (AES-CCM 인증 태그)
  CRC             :  3 B
  ─────────────────────
  패킷 전체       : 266 B

유효 페이로드 비율 = 244 / 266 = 91.7 %

이론적 최대 처리량 (2M PHY) = 2,000 Kbps
실효 처리량 상한 = 2,000 × 0.917 = 1,834 Kbps

실제 구현에서 Connection Interval 스케줄링 오버헤드, 재전송(ACK 누락),
주파수 호핑 전환 시간 등 추가 손실이 발생하여
Nordic Semiconductor 실측 기준 2M PHY + DLE: ~1,400 Kbps.
[출처: Nordic Semiconductor "Bluetooth Low Energy: A Primer" / nRF5 SDK 실측]
```

- **ATT/GATT 헤더 오버헤드**: 패킷당 7 B (L2CAP 4B + ATT Opcode 1B + Handle 2B)
- **BLE LL 헤더 + CRC**: 패킷당 9 B (LL Header 2B + Preamble 2B + AA 4B + CRC 3B)
- **Connection Interval 제약**: 일반적으로 7.5ms ~ 50ms 설정 → 처리량에 직접 영향
- **실효 처리량 범위**: **700 Kbps ~ 1,400 Kbps** (연결 파라미터 및 간섭 환경에 따라 변동)

### 신호 조건별 전송 시간 추정 (float16 기준, 250 KB = 2,000 Kbits)

| 신호 조건 | 실효 처리량 | 계산 | 전송 시간 | 처리량 근거 |
|---|---|---|---|---|
| 최적 (근거리 ≤ 1m, 장애물 없음) | ~1,400 Kbps | 2,000 / 1,400 | **~1.4 s** | Nordic nRF5340 2M PHY + DLE 실측 최대치 |
| 일반 (주머니 속, 경량 BT 간섭) | ~700 Kbps | 2,000 / 700 | **~2.9 s** | 최적치의 50%로 보수 적용. 신체 감쇠(human body attenuation) ~6 dB → RSSI 저하 → 재전송 증가 |
| 불량 (원거리 ≥ 5m, 다중 간섭) | ~200 Kbps | 2,000 / 200 | **~10 s** | BLE 2.4GHz 대역 다중 기기 간섭 환경. 재전송율 증가로 유효 처리량 급감 |

> Galaxy Buds는 오디오 스트리밍 시 A2DP 또는 LE Audio(LC3)를 별도 채널로 사용한다. Mel-Spectrogram 데이터 전송은 GATT Notification/Write 방식으로 별도 채널을 점유하므로, 오디오 스트리밍과 동시에 진행할 경우 간섭 및 처리량 저하가 발생할 수 있다.

### 전송 전략

Audi-Find의 시나리오상 Buds는 **이미 분실된 상태(BLE 단절)**이므로, 위치 찾기 요청 시점에 실시간으로 전송하는 것은 불가능하다. 따라서:

1. Buds는 BLE 연결 중 10초 순환 버퍼를 상시 유지한다.
2. 연결 단절 직전(또는 Samsung Find 앱이 위치 찾기를 요청하는 시점) BLE가 활성 상태일 때 스냅샷을 전송한다.
3. 단절 후에는 버퍼를 더 이상 갱신하지 않고, 가장 최근 스냅샷을 플래시에 보존한다.

---

## 5. 전체 파이프라인 요약

| 단계 | 담당 | 소요 시간 | 전체 비중 |
|---|---|---|---|
| Mel-Spectrogram 변환 (HiFi4 DSP) | Buds 내부 | ~10 ms | ~0.3% |
| AES-256-GCM 암호화 (Cortex-M55) | Buds 내부 | ~10 ms (보수적) | ~0.3% |
| **BLE 전송 (float16, 250 KB)** | BLE 채널 | **~1.4 s (최적) ~ 10 s (불량)** | **~99%** |
| **총합 (일반 조건)** | | **~3 s** | |

```
[전체 비중 계산, 일반 조건 기준]
연산 합계 = 10 ms + 10 ms = 20 ms = 0.02 s
BLE 전송  = 2.9 s
총합      = 2.92 s

연산 비중 = 0.02 / 2.92 × 100 = 0.7 %
전송 비중 = 2.9  / 2.92 × 100 = 99.3 %
```

연산(변환 + 암호화) 합계는 최대 20ms로 전체의 1% 미만이다. **BLE 전송이 유일한 병목**이다.

### 메모리 사용 (BES2700YP, 4MB SRAM 기준)

| 용도 | 크기 | 계산 근거 |
|---|---|---|
| Raw PCM 순환 버퍼 (10s, 16kHz, 16bit, 모노) | 320 KB | 16,000 × 10 × 2 B = 320,000 B |
| Mel-Spectrogram 버퍼 (float16) | 250 KB | 128 × 1,000 × 2 B = 256,000 B ≈ 250 KB |
| **Audi-Find 필요량** | **570 KB** | 320 + 250 = 570 KB |
| **BES2700YP 탑재 SRAM** | 4,096 KB | BES2700YP 데이터시트 |
| BT 스택 · OS · 오디오 처리 예상 점유 | ~1,500 KB | 추정치. BLE 스택(~200–400 KB) + ANC DSP 처리 버퍼 + RTOS 힙. 동급 SoC 구현 사례 기반. **실측 필요.** |
| **실질 여유** | **~2,026 KB** | 4,096 − 570 − 1,500 = 2,026 KB |

570 KB는 BES2700YP 전체 SRAM의 13.9%이며, 기존 시스템 점유분을 제외하더라도 여유 메모리 범위 내에 있다. 단, BT 스택 실제 점유량은 펌웨어 구현에 따라 달라지므로 정확한 가용 메모리는 실측이 필요하다.

---

## References

- [BES2700YP 공식 데이터시트 — Bestechnic](https://www.bestechnic.com/en/article/78/64.html)
- [BES2700YP 분석 — CNX Software](https://www.cnx-software.com/2025/06/02/bestechnic-bes2700yp-arm-cortex-m55-bluetooth-audio-soc-targets-headphones-earbuds-portable-speakers/)
- [Qwen3.5-Omni Technical Report §2.3 (arXiv 2604.15804)](https://arxiv.org/abs/2604.15804)
- [ARM Cortex-M55 / Helium(MVE) Technology Reference Manual](https://www.arm.com/products/silicon-ip-cpu/cortex-m/cortex-m55)
- [Tensilica HiFi 4 DSP 벤치마크 — Cadence](https://www.cadence.com/en_US/home/tools/ip/tensilica-ip/hifi-dsps/hifi-4.html)
- [BLE Throughput 실측 — Nordic Semiconductor Infocenter](https://infocenter.nordicsemi.com/topic/sds_s132/SDS/s1xx/ble_data_throughput/ble_data_throughput.html)
- [Bluetooth Core Specification 5.4 — Bluetooth SIG](https://www.bluetooth.com/specifications/specs/core-specification-5-4/)
- [TLS 1.3 RFC 8446 — IETF](https://www.rfc-editor.org/rfc/rfc8446)
- [mbedTLS AES 벤치마크 — Mbed TLS](https://mbed-tls.readthedocs.io/en/latest/kb/cryptography/aes-performance/)
- [Galaxy Buds2 Pro teardown — qucox.com](https://www.qucox.com/samsung-galaxy-buds2-pro-teardown/)
