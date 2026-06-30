# Audi-Find: 아이디어 상세 설명

---

## 문제 인식

Samsung Find는 분실된 Galaxy Buds의 위치를 GPS와 BLE(Bluetooth Low Energy) 신호를 기반으로 추적한다. 이 방식은 야외 개방 공간에서는 잘 동작하지만, 지하철역 내부나 대형 건물 안과 같이 GPS 신호가 약하거나 끊기는 환경에서는 위치 오차가 크게 발생한다. BLE 신호 역시 주변 기기 밀도나 구조물 간섭에 따라 불안정해질 수 있어, 실내 환경에서 신뢰도 있는 위치 추적은 여전히 어려운 문제다.

Audi-Find는 이 문제를 해결하기 위해 Galaxy Buds에 이미 탑재된 마이크를 활용한다. Buds의 Noise Cancellation 기능이 활성화되면 외부 마이크가 주변 소리를 상시 수집하고 있다. 이 주변 음향 신호(Ambient Audio)는 현재 아무런 위치 정보 목적으로 활용되지 않고 버려진다. Audi-Find는 이 신호를 위치 추론의 추가 단서로 사용하자는 아이디어에서 출발한다.

---

## 핵심 가설

특정 장소는 고유한 음향 지문(Acoustic Fingerprint)을 가진다. 지하철 플랫폼에는 열차 도착음과 안내방송이, 카페 실내에는 커피 머신 소리와 배경 음악이, 공원에는 바람 소리와 새 소리가 섞여 있다. 이 음향 패턴은 GPS 후보 위치와 결합했을 때 위치 판별의 정확도를 높일 수 있는 강력한 단서가 된다.

---

## 전체 동작 흐름

### 1단계: Galaxy Buds에서 오디오 수집

사용자가 Samsung Find 앱에서 Buds 위치 찾기를 요청하는 순간, 혹은 Buds가 마지막으로 연결되어 있던 시점에 수집된 주변 음향이 사용된다. Buds의 외부 마이크는 Noise Cancellation 동작 중 이미 주변 소리를 수집하고 있으므로, 별도의 하드웨어 추가 없이 기존 마이크 입력을 활용한다. 수집된 Raw PCM 오디오는 Buds 내부 메모리 버퍼에 순환 방식으로 저장되며, 특정 길이(예: 10초)의 스냅샷 형태로 유지된다.

### 2단계: 주파수 도메인 변환 및 암호화 (Buds → Smartphone)

Buds에서는 Raw 오디오를 그대로 전송하지 않는다. 보안과 대역폭 효율을 위해 Buds 내부에서 Raw PCM을 Mel-Spectrogram 또는 FFT 기반의 주파수 도메인 벡터로 변환한 후, TLS 1.3 및 인증서 피닝(Certificate Pinning) 방식으로 암호화하여 BLE를 통해 스마트폰으로 전송한다. 이 단계에서 Raw 오디오는 절대 기기 밖으로 나가지 않으며, 암호화된 주파수 벡터만 이동한다.

### 3단계: 스마트폰에서 클라우드로 전송

스마트폰은 Buds로부터 받은 암호화된 주파수 벡터를 복호화하지 않은 채로 클라우드 서버에 전달한다. 동시에 스마트폰의 GPS 좌표와 Map API(Google Maps 또는 Kakao Map)를 통해 추출한 주변 장소 정보를 함께 전송한다. Map API는 현재 GPS 좌표를 기반으로 반경 내 주요 시설물, 장소 유형, 도로 구조 등을 텍스트 형태로 반환하며, 이를 통해 위치 후보 목록(예: "서울역 1호선 플랫폼", "서울역 버스 환승센터", "서울역 롯데마트 지하 1층")이 구성된다.

### 4단계: 클라우드에서 오디오 임베딩 추출

클라우드 서버는 암호화된 주파수 벡터를 복호화하고, 이를 오디오 인코더에 입력하여 고차원 오디오 임베딩을 추출한다. 백본 모델로는 Qwen3.5-Omni-Plus를 사용한다. 이 모델의 AuT(Audio Transformer) 인코더는 40M 시간 분량의 오디오-텍스트 데이터로 사전학습되어 있으며, 128채널 Mel-Spectrogram을 입력받아 6.25Hz 레이트의 오디오 토큰 시퀀스로 변환한다.

### 5단계: LLM 기반 위치 추론

오디오 임베딩과 GPS 기반 위치 후보 텍스트가 하나의 프롬프트로 결합되어 Qwen3.5-Omni-Plus의 LLM(Hybrid MoE Thinker)에 입력된다. 프롬프트는 다음과 같은 구조를 가진다.

> "다음은 Galaxy Buds의 마이크가 수집한 주변 음향의 특성입니다. GPS 신호에 기반한 위치 후보는 다음과 같습니다: A) 서울역 1호선 플랫폼, B) 서울역 환승 통로, C) 서울역 롯데마트 지하 1층. 음향 환경을 분석하여 Buds가 있을 가능성이 가장 높은 위치를 선택하고, 판단 근거를 구체적으로 설명하세요."

LLM은 오디오 임베딩에서 감지된 소리의 패턴(규칙적인 열차 도착음, 고주파 마찰음, 반향 특성 등)과 각 후보 위치의 음향적 특성을 대조하여 Chain-of-Thought 방식으로 추론하고, 최종 위치와 그 근거를 자연어로 출력한다.

### 6단계: 결과 반환 및 UI 표시

클라우드의 추론 결과는 스마트폰의 Samsung Find 앱으로 반환된다. 앱은 "Buds가 서울역 1호선 플랫폼 근처에 있을 가능성이 높습니다"와 같은 자연어 설명과 함께 지도 위에 추론된 위치를 표시하고, 해당 방향으로의 길 안내를 제공한다.

---

## 보안 설계 원칙

Raw 오디오는 어떤 경우에도 기기 외부로 전송되지 않는다. Buds 내부에서만 Mel-Spectrogram 변환이 이루어지며, 변환된 벡터는 ECIES(Elliptic Curve Integrated Encryption Scheme) 방식으로 암호화되어 BLE를 통해 전달된다.

**핵심 원칙: 클라우드는 암호문과 복호화 키를 동시에 보유하지 않는다.**

```
[암호화 주체: Buds]
  임시 키쌍 (d_eph, P_eph) 생성 → 스냅샷마다 교체
  ECDH(d_eph, P_owner) → 공유 비밀 → HKDF → AES-256-GCM 키 파생
  Mel-Spectrogram 암호화 → [ SHA-256(A_i) 인덱스 | P_eph | 암호문 ] 형태로 클라우드 보관

[복호화 주체: 소유자 스마트폰]
  Find 요청 시 클라우드에서 [ P_eph | 암호문 ] 수신
  ECDH(d_owner, P_eph) → 공유 비밀 → 복호화
  → 복호화된 Mel-Spectrogram을 TLS 1.3 + cert pinning 채널로 클라우드 TEE에 전송

[추론 주체: 클라우드 TEE (AWS Nitro Enclave 등)]
  Mel-Spectrogram + GPS 후보 텍스트 → Qwen3.5-Omni-Plus 추론
  → 위치 결과만 반환, Mel-Spectrogram 즉시 메모리 삭제
```

클라우드는 암호문만 보관하며 복호화 키(d_owner)에 접근할 수 없다. 익명 인덱스(SHA-256 기반 롤링값)를 사용하여 클라우드가 데이터 소유자를 식별하는 것도 방지한다. 상세 설계는 audi_find_encryption_design.md 참조.

---

## 기대 효과

GPS 단독 방식은 실내에서 수십 미터 이상의 오차를 가질 수 있으나, 음향 지문 기반 추론을 결합하면 동일 건물 내에서도 "지하 1층 카페"와 "지하 2층 플랫폼"을 구분하는 수준의 세밀한 위치 판별이 가능해질 것으로 기대한다. 이는 GPS 단독 시스템이 절대로 제공할 수 없는 층별·구역별 구분 능력이다.

---

## 미결 사항

### 확정 대기 중

**[확정] 오디오 수집 구간: 10초**
BLE 연결이 끊기는 순간의 가장 최근 10초 스냅샷을 사용한다. 순환 버퍼(circular buffer) 방식으로 Buds 내부에 상시 유지하다가 연결 단절 시 고정(freeze)하여 전송한다. 10초로 설정한 근거는 TUT 2017, TAU 2019/2020/2022 등 주요 ASC 벤치마크가 10초 세그먼트를 표준 단위로 사용하기 때문이다. 이를 통해 시뮬레이션 결과를 기존 벤치마크 수치와 직접 비교할 수 있다. 상세 근거는 audio_window.md 참조.

**[보류] Fine-tuning 여부**
Qwen3.5-Omni-Plus를 위치 판별 태스크에 맞게 fine-tuning할 것인지, 아니면 프롬프트 엔지니어링만으로 zero-shot 운용할 것인지는 초기 시뮬레이션 성능을 확인한 후 결정한다.

### 조사 필요 (처리해야 할 문제)

**[확정] GPS 후보 위치 반경 기준**
Samsung Find(SmartThings Find)의 위치 오차를 조사한 결과, Offline Finding(BLE 크라우드소싱) 방식의 경우 실제 위치와 표시 위치가 최대 **100미터** 차이날 수 있으며, 실내 BLE 탐지 유효 반경은 약 **10미터** 수준으로 줄어든다. 실내 GPS 자체의 오차는 콘크리트 건물 기준 **10미터 이내**로 보고된다.

이를 종합하여 Map API 후보 장소 선정 반경을 **100미터**로 설정한다. 이는 Find의 최대 오차 범위를 포괄하는 보수적인 기준이며, 실내 환경에서 GPS 오차와 BLE 크라우드소싱 오차를 모두 커버한다. 반경 100미터 이내의 장소들을 후보 목록으로 구성하여 LLM에 제공한다.

### 제외 항목

**[제외] 레이턴시 목표**: 현재 설계 범위에서 고려하지 않기로 결정.

### 기타 세부 사항 (추후 정리)

- Mel-Spectrogram 파라미터 (윈도우 크기, 홉 크기, 멜 필터 수) — Buds 연산 제약 내 최적값 설정 필요
- BLE 연결 단절 후 마지막 수집 벡터의 유효 기간 정의

---

## References

- [Samsung SmartThings Find 위치 정확도 공식 안내](https://www.samsung.com/us/support/answer/ANS00080656/)
- [SmartThings Find vs Apple Find My 실측 비교](https://www.startupbooted.com/smartthings-find)
- [실내 GPS 정확도 연구 — Indoor Positioning Using GPS Revisited](https://www.researchgate.net/publication/221016004_Indoor_Positioning_Using_GPS_Revisited)
- [GPS vs Indoor Positioning 비교](https://www.blueiot.com/blog/gps-vs-indoor-positioning.html)
- [SALMONN 논문 (ICLR 2024)](https://arxiv.org/abs/2310.13289)
- [Qwen3.5-Omni Technical Report](https://arxiv.org/abs/2604.15804)
- [TAU Urban Acoustic Scenes 2022 (DCASE)](https://dcase.community/challenge2022/task-low-complexity-acoustic-scene-classification)
