# Audi-Find: 베이스라인 모델 비교 — SALMONN vs Qwen3.5-Omni-Plus

> 클라우드 전용 배포 전제. 성능 우선, 텍스트 출력 필수.
> 논문 PDF 및 웹 검색 교차 검증 완료.

---

## 모델 개요

### SALMONN
- **논문**: SALMONN: Towards Generic Hearing Abilities for Large Language Models
- **학회**: ICLR 2024
- **arXiv**: 2310.13289
- **개발**: Tsinghua University / ByteDance

### Qwen3.5-Omni-Plus
- **논문**: Qwen3.5-Omni Technical Report
- **arXiv**: 2604.15804
- **개발**: Alibaba Cloud (Qwen Team, 2026)
- **공개일**: 2026년 4월

---

## 아키텍처 비교

| 항목 | SALMONN | Qwen3.5-Omni-Plus |
|---|---|---|
| **오디오 인코더** | Whisper-Large-v2 + BEATs (듀얼) | AuT (Audio Transformer, 자체 개발) |
| **LLM 백본** | Vicuna-13B (LLaMA 기반) | Hybrid MoE Transformer (Qwen3.5 기반) |
| **연결 모듈** | Window-level Q-Former | Cross-attention (직접 연결) |
| **총 파라미터** | ~14B | 수백 B (MoE, 활성화 파라미터 별도) |
| **비전 인코더** | ❌ 없음 | ✅ SigLIP2 (멀티모달) |
| **음성 출력** | ❌ 없음 | ✅ Talker (RVQ 기반 TTS) |
| **최대 오디오 입력** | 30초 | **10시간** (256k 토큰) |
| **오디오 토큰 레이트** | 50Hz → 슬라이딩 윈도우 | **6.25Hz** (4× 다운샘플링) |

### 오디오 처리 방식 상세

**SALMONN**
```
Audio → Whisper-Large-v2 (음성)
      → BEATs (환경음)
      → Concat (frame-level)
      → Window-level Q-Former (L=17프레임 ≈ 0.33초/윈도우, N=1 쿼리)
      → 88 tokens / 30초
      → Vicuna-13B + LoRA
```

**Qwen3.5-Omni-Plus**
```
Audio (16kHz) → 128-ch Mel-Spectrogram (25ms window, 10ms hop)
              → AuT Encoder (32× Self-Attention + 4× Conv2D 다운샘플링)
              → 6.25Hz 토큰 (160ms / token)
              → Hybrid MoE Thinker (LLM)
              → 텍스트 응답
```

---

## 성능 비교

### 음향 환경 분류 관련 태스크

| 벤치마크 | SALMONN | Qwen3.5-Omni-Plus | 비고 |
|---|---|---|---|
| **MMAU (전체)** | 미평가 | **82.2** | Gemini-3.1 Pro 81.1 대비 SOTA |
| **ESC-50 (zero-shot)** | 16.4% | 미공개 | SALMONN 논문 기준 |
| **TUT 2017 Urban ASC** | 7.8% | 미평가 | GAMA 논문 기준 |
| **DCASE 2017** | 18.0% (Mi-F1) | 미평가 | GAMA 논문 기준 |

### 일반 오디오 이해 태스크

| 벤치마크 | SALMONN | Qwen3.5-Omni-Plus | 비고 |
|---|---|---|---|
| **MMAU** | 미평가 | **82.2** | vs Gemini 81.1 |
| **MMAR** | 미평가 | 80.0 | |
| **MMSU** | 미평가 | **82.8** | |
| **AIR-Bench** | SOTA (2024 기준) | 미비교 | SALMONN 논문 기준 |
| **ClothoAQA** | SOTA (2024 기준) | 미비교 | SALMONN 논문 기준 |
| **VoiceBench** | 미평가 | **93.1** | vs Gemini 88.9 |
| **ASR (LibriSpeech)** | 2.1% WER | **1.11% WER** | Qwen3.5 우세 |

### Audi-Find 핵심 능력: GPS 텍스트 결합 추론

두 모델 모두 LLM 기반이므로 GPS 후보 텍스트를 프롬프트로 주입 가능:

```
프롬프트 예시:
"다음은 Galaxy Buds의 주변 음향 분석 결과입니다.
 GPS 후보 위치: A) 서울역 지하철 플랫폼, B) 한강공원, C) 스타벅스 실내
 음향 특성을 근거로 가장 가능성 높은 위치를 선택하고 이유를 설명하세요."
```

| 능력 | SALMONN | Qwen3.5-Omni-Plus |
|---|---|---|
| GPS 텍스트 프롬프트 주입 | ✅ | ✅ |
| Chain-of-Thought 추론 | ✅ (제한적) | ✅ (강화됨) |
| 자연어 근거 생성 | ✅ | ✅ |
| 음성+환경음 동시 처리 | ✅ (Whisper+BEATs 분리) | ✅ (AuT 통합) |

---

## 클라우드 배포 관점 비교

| 항목 | SALMONN | Qwen3.5-Omni-Plus |
|---|---|---|
| **모델 크기** | ~14B | 수백B (MoE) |
| **GPU 요구사항** | A100 40GB 이상 | 고성능 클라우드 필수 |
| **최대 입력 길이** | 30초 | **10시간** |
| **First-Packet 레이턴시** | 수백ms ~ 수초 | **435ms (Plus, 1 Conc.)** |
| **오픈소스** | ✅ (GitHub) | ✅ (HuggingFace) |
| **API 제공** | ❌ | ✅ (Alibaba Cloud) |
| **상용 API 사용 가능** | ❌ | ✅ |

---

## 종합 판단

### SALMONN의 강점
- 환경음(BEATs)과 음성(Whisper)을 **명시적으로 분리 처리** → 두 신호가 혼재된 Galaxy Buds 입력에 구조적으로 적합
- ICLR 2024 정식 게재 → 재현성 검증 완료
- 상대적으로 가벼운 14B 모델

### SALMONN의 약점
- ESC-50 16.4%, TUT 2017 7.8% — 환경음 분류 수치 매우 낮음
- 최대 30초 입력 제한
- 상용 API 없음, 자체 서버 배포 필수
- 2023년 모델 기반 → LLM 추론 품질 한계

### Qwen3.5-Omni-Plus의 강점
- MMAU 82.2% — 현재 오디오 이해 SOTA (Gemini-3.1 Pro 81.1% 상회)
- **10시간 오디오 입력** 지원 → 연속 모니터링 가능
- 상용 API 제공 → 클라우드 배포 즉시 가능
- 최신 MoE 아키텍처 → 추론 품질 및 언어 능력 우수
- 멀티모달 (텍스트+오디오+비전) → 향후 GPS 지도 이미지 결합 확장 가능

### Qwen3.5-Omni-Plus의 약점
- TAU Urban ASC, ESC-50 직접 평가 수치 없음 (환경음 분류 능력 미검증)
- MoE 대형 모델 → 클라우드 비용 높음
- AuT 단일 인코더 → 음성/환경음 분리 처리 안 함

---

## 결론

**Qwen3.5-Omni-Plus를 1차 백본으로 채택, SALMONN을 비교 baseline으로 설정.**

| 역할 | 모델 | 이유 |
|---|---|---|
| **주 백본** | **Qwen3.5-Omni-Plus** | MMAU SOTA, 상용 API, 10시간 입력, 추론 품질 |
| **비교 baseline** | **SALMONN** | 환경음/음성 분리 구조, ICLR 검증, 구조적 대조군 |

핵심 실험 질문: **GPS 컨텍스트 텍스트를 추가했을 때 두 모델의 위치 판별 정확도가 얼마나 향상되는가?**
이 향상폭이 Audi-Find의 핵심 기여 지표가 된다.
