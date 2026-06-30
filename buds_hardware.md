# Galaxy Buds 하드웨어 스펙 — Audi-Find 관점

> Audi-Find의 오디오 수집·변환·전송 파이프라인 설계를 위한 하드웨어 기준 정리.
> 내부 칩셋 정보는 공식 비공개이며, 분해(teardown) 분석 결과를 기반으로 한다.
> 출처를 명시하지 않은 수치는 추정값임을 전제한다.

---

## 1. 모델별 개요

Audi-Find의 타겟 디바이스는 ANC 마이크가 탑재된 Galaxy Buds Pro 계열이다.
ANC 미탑재 모델(Buds FE 등)은 외부 마이크가 없어 주변음 수집이 불가하다.

| 모델 | 출시 | BT 버전 | ANC | Audi-Find 적합 |
|---|---|---|---|---|
| Galaxy Buds Pro (2021) | 2021.01 | 5.0 | ✅ | ⭐⭐ (구세대) |
| Galaxy Buds2 Pro (2022) | 2022.08 | 5.3 | ✅ | ⭐⭐⭐ |
| Galaxy Buds3 Pro (2024) | 2024.07 | 5.4 | ✅ | ⭐⭐⭐⭐ (권장) |

---

## 2. Galaxy Buds2 Pro — 내부 하드웨어

### 메인 SoC: BES2700YP (Bestechnic)

분해(teardown) 분석에서 확인된 칩이다. Bestechnic(深圳市貝斯特尼科技)은 중국계 무선 오디오 SoC 전문 팹리스로, Galaxy Buds2 Pro에 이어 여러 플래그십 이어버드에 납품하고 있다.

| 항목 | 수치 |
|---|---|
| **제조사** | Bestechnic (BES) |
| **모델명** | BES2700YP |
| **공정** | **12nm** |
| **CPU** | ARM **Cortex-M55** |
| **DSP** | Tensilica **HiFi 4** |
| **NPU** | BECO NPU (BES 자체 개발, 센서 허브용) |
| **보조 코어** | STAR-MC1 (BT 호스트 서브시스템) |
| **내장 SRAM** | **4 MB** (CPU + BT + 센서허브 공유) |
| **블루투스** | 듀얼모드 BT Classic + **BLE 5.3** |
| **VAD** | ✅ (Voice Activity Detection, 하드웨어) |

> **Audi-Find 관점**: Cortex-M55는 ARM의 DSP 최적화 코어로, SIMD 벡터 연산(Helium 명령어)을 지원한다. 10초 오디오의 Mel-Spectrogram 변환(FFT 1,000회)은 이 코어에서 약 10ms 이내로 처리 가능한 수준이다. 4MB SRAM은 10초 × 16kHz × 16bit = 320KB Raw PCM 버퍼를 보유하기에 충분하다.

### 보조 칩

| 칩 | 제조사 | 역할 |
|---|---|---|
| **MUB02** | Samsung | 전력 관리 (PMIC) |
| **LSM6DSV16BX** | STMicroelectronics | 6축 IMU (가속도계 + 자이로) |
| **골전도 2-in-1 칩** | 미공개 | 360° 오디오 + 통화 노이즈 캔슬링 |

### 공식 스펙 (사용자 향)

| 항목 | 수치 |
|---|---|
| **이어버드 무게** | 5.5 g |
| **마이크 수** | 3개 (외부 2 + 내부 1, 고 SNR) |
| **블루투스** | 5.3 |
| **지원 코덱** | SBC / AAC / SSC (Samsung Seamless, 24bit Hi-Fi) |
| **IP 등급** | IPX7 (수심 1m, 30분) |
| **이어버드 배터리** | 61 mAh |
| **케이스 배터리** | 515 mAh |
| **재생 시간 (ANC ON)** | 5시간 |
| **재생 시간 (ANC OFF)** | 8시간 |
| **케이스 포함 최대** | 29시간 (ANC OFF) |

---

## 3. Galaxy Buds3 Pro — 내부 하드웨어

### 메인 SoC (미공개)

삼성은 Buds3 Pro의 내부 칩셋을 공식 공개하지 않았다. 분해 분석(reverse-costing.com)이 존재하나 칩 모델명 전체 공개는 확인되지 않았다. BT 5.4 지원 및 BES 계열 후속 SoC 탑재 가능성이 높다.

| 항목 | 상태 |
|---|---|
| **칩셋 모델명** | 미공개 (teardown 분석 확인 불가) |
| **블루투스** | **5.4** (확정) |
| **공정** | 미공개 |

### 공식 스펙 (사용자 향)

| 항목 | 수치 |
|---|---|
| **스피커** | 10.5mm 다이나믹 드라이버 + 6.1mm 플래너 트위터 (2-way) |
| **마이크 수** | **3개/이어버드 (총 6개)** — 외부 ANC + 통화 + 내부 피드백 |
| **블루투스** | 5.4 |
| **지원 코덱** | SBC / AAC / SSC / **LC3 (LE Audio)** / Auracast |
| **오디오 해상도** | 24bit / 96kHz (SSC HiFi) |
| **IP 등급** | **IP57** (먼지 완전 차단 + 수심 1m, 30분) |
| **이어버드 배터리** | 53 mAh |
| **케이스 배터리** | 515 mAh |
| **재생 시간 (ANC ON)** | 6시간 |
| **재생 시간 (ANC OFF)** | 7시간 |
| **케이스 포함 최대** | 30시간 (ANC OFF) |

---

## 4. Audi-Find 관점 평가

### 마이크 구성

| 모델 | 마이크 수 | ANC 외부 마이크 | 주변음 수집 가능 |
|---|---|---|---|
| Buds2 Pro | 3개 | ✅ 외부 2개 | ✅ |
| Buds3 Pro | 6개 (3개/이어버드) | ✅ 외부 마이크 포함 | ✅ |

ANC 동작 중 외부 마이크는 주변음을 상시 수집한다. Audi-Find는 이 신호를 활용하므로 **사용자에게 추가적인 하드웨어 동작을 요구하지 않는다.**

### 연산 능력 (Buds2 Pro 기준, Buds3 Pro 추정 동등 이상)

| 작업 | 예상 처리 시간 |
|---|---|
| 10초 Raw PCM 버퍼링 | 연속 진행 (순환 버퍼) |
| Mel-Spectrogram 변환 (FFT 1,000회) | ~10 ms |
| AES-256-GCM 암호화 (250 KB) | ~2–5 ms |
| **합계** | **~15 ms** |

### 메모리 여유 (Buds2 Pro 기준)

| 용도 | 크기 |
|---|---|
| Raw PCM 순환 버퍼 (10초, 16kHz 16bit) | 320 KB |
| Mel-Spectrogram 버퍼 (float16) | 250 KB |
| **필요 메모리** | **570 KB** |
| **탑재 SRAM (BES2700YP)** | 4,096 KB |
| **여유** | ~3.5 MB (OS + BT 스택 + 오디오 처리 분배 필요) |

> 전체 4MB SRAM은 BT 스택, OS, 오디오 처리 등과 공유하므로, Mel-Spectrogram 버퍼를 위한 실제 가용 메모리는 별도 설계가 필요하다. 다만 용량 상 불가능한 수준은 아니다.

---

## References

- [BES2700YP 공식 데이터시트 — Bestechnic](https://www.bestechnic.com/en/article/78/64.html)
- [BES2700YP CNX Software 분석](https://www.cnx-software.com/2025/06/02/bestechnic-bes2700yp-arm-cortex-m55-bluetooth-audio-soc-targets-headphones-earbuds-portable-speakers/)
- [Galaxy Buds2 Pro teardown 분석 — qucox.com](https://www.qucox.com/samsung-galaxy-buds2-pro-teardown/)
- [Galaxy Buds3 Pro teardown — reverse-costing.com](https://www.reverse-costing.com/teardowns/samsung-galaxy-buds3-pro/)
- [Galaxy Buds3 Pro 공식 스펙 — Samsung US](https://www.samsung.com/us/mobile-audio/galaxy-buds3-pro/)
- [Galaxy Buds2 Pro 공식 스펙 — Samsung Canada](https://www.samsung.com/ca/audio-sound/galaxy-buds/galaxy-buds2-pro-graphite-sm-r510nzaaxac/)
- [Yole Group Galaxy Buds2 Pro Teardown Track](https://www.yolegroup.com/product/teardown-track/samsung-galaxy-buds2-pro/)
