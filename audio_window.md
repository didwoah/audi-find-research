# Audi-Find: 오디오 수집 구간 결정

> 오디오 수집 구간(Audio Window) 설계 근거 정리.

---

## 결정 사항

**오디오 수집 구간: 10초**

---

## 근거

### 1. 벤치마크 데이터셋 정합성

Audi-Find의 시뮬레이션과 성능 비교는 공개된 ASC(Acoustic Scene Classification) 벤치마크 데이터셋을 기반으로 한다. 주요 데이터셋의 세그먼트 길이는 다음과 같다.

| 데이터셋 | 세그먼트 길이 |
|---|---|
| TUT Urban Acoustic Scenes 2017 (DCASE 2017) | **10초** |
| TAU Urban Acoustic Scenes 2019 Mobile (DCASE 2019) | **10초** |
| TAU Urban Acoustic Scenes 2020 Mobile (DCASE 2020) | **10초** |
| TAU Urban Acoustic Scenes 2022 Mobile (DCASE 2022) | 1초 (10초 기반 재분할) |

TAU 2022의 1초 세그먼트도 TAU 2020의 10초 클립을 재분할한 것으로, 원본 녹음 단위는 동일하게 10초다. 즉 현존하는 ASC 벤치마크는 사실상 **10초를 표준 단위**로 삼는다.

수집 구간을 10초로 설정하면 Audi-Find의 시뮬레이션 결과를 기존 벤치마크 수치와 직접 비교할 수 있다. 10초가 아닌 다른 구간(예: 5초, 30초)을 사용할 경우 공정한 비교가 불가능하다.

### 2. 모델 입력 호환성

Qwen3.5-Omni-Plus의 AuT 인코더는 10ms hop size 기반 Mel-Spectrogram을 입력받으며, 출력 토큰 레이트는 6.25Hz(토큰당 160ms)다. 10초 오디오는 약 **62.5토큰**으로 변환되며, 최대 입력(256k 토큰, 10시간)에 비해 극히 작은 분량이므로 컨텍스트 제약이 없다.

| 항목 | 수치 |
|---|---|
| 수집 구간 | 10초 |
| 출력 토큰 수 | ~62.5 토큰 (10초 × 6.25Hz) |
| 최대 입력 용량 | 256k 토큰 (10시간) |
| 컨텍스트 여유 | 충분 |

### 3. 실용성

10초는 장소의 음향 특성을 포착하기에 충분한 길이이면서, Buds 내부 순환 버퍼에 저장하기에도 현실적인 크기다. 배터리 소모나 버퍼 용량 측면에서 불필요하게 긴 구간(예: 30초 이상)을 유지할 이유가 없다.

---

## 수집 방식

Buds가 마지막으로 연결되어 있던 시점의 **가장 최근 10초 스냅샷**을 사용한다. 순환 버퍼(circular buffer) 방식으로 Buds 내부에 상시 유지하며, BLE 연결이 끊기는 순간의 버퍼 내용을 고정(freeze)하여 전송에 사용한다.

---

## References

- [TAU Urban Acoustic Scenes 2022 (DCASE 2022 Task 1)](https://dcase.community/challenge2022/task-low-complexity-acoustic-scene-classification)
- [TAU Urban Acoustic Scenes 2020 (DCASE 2020 Task 1)](https://dcase.community/challenge2020/task1-acoustic-scene-classification)
- [TAU Urban Acoustic Scenes 2019 (DCASE 2019 Task 1)](https://dcase.community/challenge2019/task1-acoustic-scene-classification)
- [TUT Acoustic Scenes 2017 (DCASE 2017 Task 1)](https://dcase.community/challenge2017/task1-acoustic-scene-classification)
- [Qwen3.5-Omni Technical Report (arXiv 2604.15804)](https://arxiv.org/abs/2604.15804)
