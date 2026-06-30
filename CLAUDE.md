# Audi-Find: 음성 기반 Galaxy Buds 위치 추적 개선

## 프로젝트 목적

Samsung Find의 GPS/BLE 기반 위치 추적이 실내·음영 지역에서 부정확한 문제를 해결하기 위해, Galaxy Buds의 마이크가 수집하는 **주변 음향 신호(Ambient Audio)** 를 추가 컨텍스트로 활용하여 위치 추론 정확도를 높이는 시스템을 설계·검증한다.

## 핵심 가설

특정 장소는 고유한 **음향 지문(Acoustic Fingerprint)** 을 가지며, 이를 GPS 후보 위치와 결합하면 GPS 단독 대비 위치 판별 정확도가 유의미하게 향상된다.

## 확정된 아키텍처

```
Galaxy Buds
  → Raw Audio 수집 (NC 마이크 활용)
  → Mel-Spectrogram 변환 (Buds 내부)
  → 암호화 (TLS 1.3 + cert pinning)
  → BLE → Smartphone → Cloud 전송

Cloud (모든 모델 클라우드에서 실행)
  → 복호화
  → Qwen3.5-Omni-Plus (AuT 오디오 인코더 + Hybrid MoE LLM)
  → GPS + Map API 텍스트 컨텍스트 결합
  → LLM 추론 (위치 판별 + 근거 생성)
  → 결과 반환 → Galaxy App
```

**보안 원칙**: Raw audio는 절대 전송하지 않음. 암호화된 Mel-Spectrogram 벡터만 클라우드로 전달. 모든 인코더와 LLM은 클라우드에서 실행.

## 백본 모델

- **주 백본**: Qwen3.5-Omni-Plus (MMAU 82.2%, 상용 API 제공)
- **비교 baseline**: SALMONN (ICLR 2024, Whisper+BEATs 듀얼 인코더)

## 작업 목표

1. **아이디어 구체화** — idea.md 기반으로 미결 사항 순차 해소
2. **참고 자료 수집** — 관련 논문·기술 스택 정리
3. **시뮬레이션** — TAU Urban Acoustic Scenes 2022 기반 위치 판별 검증 (Python)
4. **설계 문서화** — 각 단계별 스펙 정리

## 행동 규칙

1. **모든 md 파일 하단에 참조 링크 정리**: 작성하는 md 파일에 외부 자료(논문, 데이터셋, 공식 문서 등)를 인용한 경우, 파일 최하단에 `## References` 섹션으로 링크를 정리한다.
2. **idea.md를 항상 참고**: 아이디어 관련 작업 시 반드시 idea.md의 현재 내용을 기준으로 한다. 미결 사항이 해소되면 idea.md를 즉시 업데이트한다.
3. **수치는 논문 원문 기준**: 성능 수치를 기재할 때는 반드시 원문(PDF 또는 공식 HTML)을 확인한 값만 사용하며, 출처를 명시한다.
4. **클라우드 전용**: 모든 모델 추론은 클라우드에서 실행하는 것을 전제로 설계한다. 온디바이스 추론은 고려하지 않는다.
