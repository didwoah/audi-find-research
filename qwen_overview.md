# Qwen3.5-Omni-Plus 개요 — Audi-Find 관점

> Audi-Find의 백본 모델인 Qwen3.5-Omni-Plus의 아키텍처와 동작 방식을 정리한다.
> 출처: Qwen3.5-Omni Technical Report (arXiv 2604.15804, 2026-04-22).

---

## 1. 핵심 아이디어

Qwen3.5-Omni는 텍스트, 오디오, 이미지, 비디오를 **하나의 모델**에서 동시에 처리하고, 텍스트와 음성으로 동시에 응답할 수 있는 **완전 통합형 멀티모달 LLM**이다.

기존 멀티모달 모델들은 텍스트 이해와 음성 생성을 별도 모듈로 처리했다. Qwen3.5-Omni는 이를 단일 엔드투엔드 파이프라인으로 통합하여, 오디오 입력을 받아 텍스트 또는 음성으로 즉시 응답하는 실시간 상호작용이 가능하다.

---

## 2. 아키텍처: Thinker-Talker

모델은 두 개의 주요 컴포넌트로 구성된다.

```
입력 (오디오 / 이미지 / 텍스트)
    ↓
[AuT Encoder]  [Vision Encoder]
    ↓                ↓
    ↓ ← 통합 → ↓
[Thinker: Hybrid MoE Transformer]  ← 이해 및 추론
    ↓
[Talker: Hybrid MoE Transformer]   ← 음성 생성 (선택)
    ↓
텍스트 출력 / 음성 출력 (동시 가능)
```

### Thinker

이해와 추론을 담당하는 LLM 본체다. Hybrid Attention MoE(Mixture-of-Experts) 구조를 채택하여 수백 B 규모의 파라미터를 갖지만, MoE 특성상 추론 시 활성화되는 파라미터는 일부에 불과하다. 오디오, 이미지, 텍스트를 통합 시퀀스로 받아 텍스트 응답을 생성한다.

### Talker

Thinker의 출력을 받아 음성 토큰(RVQ 코덱)으로 변환하는 음성 생성 모듈이다. Audi-Find에서는 텍스트 응답만 사용하므로 Talker는 비활성화 상태로 운용된다.

---

## 3. 오디오 입력 처리: AuT (Audio Transformer)

AuT는 Qwen3.5-Omni가 자체 개발한 오디오 인코더로, 4,000만 시간의 오디오-텍스트 데이터로 학습되었다.

### 처리 흐름

```
Raw 오디오 (임의 길이)
  → 16kHz 리샘플링
  → 128채널 Mel-Spectrogram (25ms window, 10ms hop)
  → AuT Encoder
      - 32× Self-Attention Layer
      - 4× Downsampling Conv2D
  → 6.25Hz 오디오 토큰 (1토큰 = 160ms)
  → Thinker(LLM)에 입력
```

### 핵심 스펙

| 항목 | 수치 |
|---|---|
| 입력 포맷 | 128채널 Mel-Spectrogram |
| Mel 파라미터 | 25ms window, 10ms hop |
| 출력 토큰 레이트 | **6.25Hz** (토큰당 160ms) |
| 최대 입력 길이 | **10시간** (256k 토큰) |
| 학습 데이터 | 4,000만 시간 오디오-텍스트 |

### 10초 입력 시 토큰 수

```
10초 × 6.25 토큰/초 = 62.5 토큰 ≈ 63 토큰
```

전체 컨텍스트(256k 토큰) 대비 0.025%만 차지하므로, 텍스트 프롬프트와 함께 충분히 넣을 수 있다.

---

## 4. 텍스트 입력 처리

Qwen3.5 토크나이저를 사용하며, 어휘 크기는 250k(바이트 수준 BPE)다. GPS 후보 위치 목록, 지도 정보, 추론 지시문은 모두 일반 텍스트 토큰으로 입력된다.

---

## 5. 멀티모달 입력 통합 방식

Thinker는 오디오 토큰과 텍스트 토큰을 **단일 시퀀스**로 인터리빙하여 처리한다. 각 모달리티에는 타임스탬프가 삽입되어 시간적 위치를 명시한다.

```
시퀀스 예시:
[<0.0s> 오디오토큰 × 63개 <10.0s>]
[텍스트: "GPS 후보 위치: A) 서울역 1호선 플랫폼 B) ..."]
[텍스트: "음향 환경을 분석하여 위치를 추론하고 근거를 설명하세요."]
```

---

## 6. Audi-Find에서의 활용

### 입력 구성

| 입력 종류 | 형태 | 내용 |
|---|---|---|
| 오디오 | 128ch Mel-Spectrogram (float16) | Buds 주변 음향 10초 스냅샷 |
| 텍스트 | 자연어 프롬프트 | GPS 후보 위치 목록 + 추론 지시 |

### 프롬프트 구조 예시

```
[오디오 토큰: 10초 주변음 스냅샷]

다음은 Galaxy Buds의 외부 마이크가 수집한 10초간의 주변 음향입니다.
현재 GPS 신호 기반 추정 위치에서 반경 100m 이내의 장소 후보는 다음과 같습니다:

A) 서울역 1호선 플랫폼 (지하 2층)
B) 서울역 버스 환승센터 (지상 1층)
C) 서울역 롯데마트 지하 1층

위 음향 특성과 각 장소의 음향 환경을 비교하여,
Buds가 있을 가능성이 가장 높은 위치를 선택하고
판단 근거를 구체적으로 설명하세요.
```

### 출력

Thinker가 Chain-of-Thought 방식으로 추론한 뒤 자연어 텍스트를 반환한다.

```
"수집된 음향에서 규칙적인 열차 도착음(저주파 진동)과
안내방송 에코가 감지됩니다. 이는 지하 공간의 반향 특성과 일치하며,
버스 환승센터(외부 소음 우세)나 마트(인공 냉각음 우세)와 구별됩니다.
따라서 A) 서울역 1호선 플랫폼일 가능성이 가장 높습니다."
```

---

## 7. 성능 요약

| 벤치마크 | 점수 | 비교 |
|---|---|---|
| **MMAU (전체)** | **82.2%** | Gemini-3.1 Pro 81.1% 상회, SOTA |
| MMAR | 80.0% | — |
| MMSU | 82.8% | — |
| VoiceBench | 93.1% | Gemini-3.1 Pro 88.9% 상회 |
| ASR LibriSpeech WER | 1.11% | SOTA 수준 |
| First-Packet Latency (Plus) | 435ms | 오디오 입력 기준 |

---

## 8. 클라우드 배포 특성

| 항목 | 내용 |
|---|---|
| 파라미터 규모 | 수백 B (MoE, 활성 파라미터는 일부) |
| GPU 요구사항 | 고성능 클라우드 필수 (A100 80GB 이상 권장) |
| API 제공 | ✅ Alibaba Cloud Model Studio |
| 오픈소스 | ✅ HuggingFace 공개 |
| 스트리밍 지원 | ✅ Chunked Prefilling |

Audi-Find는 상용 API를 통해 Qwen3.5-Omni-Plus를 클라우드에서 호출하는 방식으로 운용한다. 자체 서버 배포 없이 즉시 프로토타이핑 가능하다.

---

## References

- [Qwen3.5-Omni Technical Report (arXiv 2604.15804)](https://arxiv.org/abs/2604.15804)
- [Qwen3.5-Omni HuggingFace](https://huggingface.co/Qwen/Qwen3.5-Omni)
- [Alibaba Cloud Model Studio API](https://www.alibabacloud.com/help/en/model-studio/qwen-omni)
- [MMAU Benchmark (arXiv 2406.09154)](https://arxiv.org/abs/2406.09154)
