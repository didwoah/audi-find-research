# Audi-Find: 오디오 암호화 설계 — 취약점 분석 및 개선 설계

> 제안된 암호화 아이디어의 취약점을 분석하고, ECIES 기반 개선 설계를 상세히 기술한다.
> 참조: find_encryption.md (Samsung/Apple 암호화 구현 비교)

---

## 1. 원본 아이디어 정리

```
[버즈]
  Mel-Spectrogram 생성 → 암호화 (키: K_i)
  BLE 브로드캐스트: [ 헤더(소유자 식별자 + 키 발급 시간) | 암호화된 Mel-Spectrogram ]

[Finder Galaxy 기기]
  BLE 수신 → 클라우드에 업로드

[클라우드]
  암호화된 Mel-Spectrogram 보관 (복호화 키 없음)

--- Find 요청 ---

[클라우드]
  헤더의 소유자 식별자로 검색 → 스마트폰에 T_i 제시하며 복호화 키 요청

[스마트폰]
  K_i 암호화하여 클라우드 전달

[클라우드]
  K_i로 Mel-Spectrogram 복호화 → Qwen3.5-Omni-Plus 추론 → 위치 반환
```

---

## 2. 취약점 분석

### 취약점 1 ★★★★★ — 클라우드가 암호문 + 복호화 키를 동시에 보유

Find 요청 시 클라우드는 보관 중인 암호문(C)과 스마트폰이 전달한 복호화 키(K_i)를 동시에 가진다. 이 순간 클라우드는 Mel-Spectrogram을 완전히 복호화할 수 있는 상태가 된다.

```
공격 시나리오:
  내부자 위협 — K_i 수신 직후 C를 복호화하여 저장
  서버 침해  — K_i + C가 동시에 노출
  폐기 불검증 — "즉시 폐기" 주장의 검증 수단 없음
```

이 취약점이 존재하는 한 "클라우드는 오디오를 볼 수 없다"는 주장이 성립하지 않는다.

### 취약점 2 ★★★★ — 헤더의 평문 소유자 식별자

```
헤더: [ 소유자 식별자(plaintext) | 키 발급 시간 T_i ]
```

클라우드와 Finder 기기 모두 "누구의 오디오인지"를 알 수 있다.
소유자 식별자 + 타임스탬프 + Finder GPS = 소유자 이동 패턴 추적 가능.

### 취약점 3 ★★★ — 복호화 키 전달 채널 공격 표면

```
재전송 공격  — 캡처된 K_i 패킷 재전송으로 과거 오디오 복호화
키 에스크로  — 클라우드가 K_i를 폐기하지 않고 보관 시 이후 언제든 복호화 가능
MITM        — cert pinning 없는 TLS 구현 시 K_i 탈취 가능
```

### 취약점 4 ★★★ — 키 교체 동기화 실패

```
버즈가 암호화에 사용한 키: K_5
스마트폰이 동기화한 최신 키: K_4  ← BLE 단절이 교체 직전에 발생
→ 클라우드의 암호문은 K_5로 암호화 → K_4로 복호화 불가 → Find 실패
```

### 취약점 5 ★★ — BLE 광고에 Mel-Spectrogram 직접 포함 불가

BLE 광고 패킷 최대 페이로드: 31 bytes (Extended Advertising: 255 bytes).
암호화된 Mel-Spectrogram은 최소 125 KB → 브로드캐스트 불가.
GATT 연결 후 데이터 전송 방식이어야 한다.

---

## 3. 개선 설계 — 핵심 원칙

> **클라우드는 C(암호문)와 K(복호화 키)를 절대 동시에 보유하지 않는다.**
>
> 복호화는 소유자 스마트폰에서 수행한다(ECIES).
> 클라우드는 복호화된 평문을 수신하여 LALM 추론만 담당한다.
> 클라우드 내부 평문 노출은 TEE(신뢰 실행 환경)로 격리한다.

---

## 4. 적용 기술 상세

### 4.1 ECDH (Elliptic Curve Diffie-Hellman)

**개념**

타원 곡선 위의 점 연산을 이용해 두 당사자가 공개 채널에서 공유 비밀(Shared Secret)을 도출하는 키 교환 프로토콜이다.

```
타원 곡선 위의 점 G (생성원, Generator)가 공개값.
Alice: 개인키 d_A (비밀), 공개키 P_A = d_A · G (공개)
Bob:   개인키 d_B (비밀), 공개키 P_B = d_B · G (공개)

공유 비밀:
  Alice가 계산: d_A · P_B = d_A · (d_B · G) = d_A · d_B · G
  Bob이 계산:   d_B · P_A = d_B · (d_A · G) = d_A · d_B · G
  → 동일한 결과. 제3자는 d_A, d_B 없이는 계산 불가 (ECDLP 난제)
```

**Audi-Find 적용**

버즈가 임시 키쌍(d_eph, P_eph)을 생성하고, 소유자 스마트폰의 공개키 P_owner와 ECDH를 수행한다. 도출된 공유 비밀로 대칭키를 만든다.

**사용 곡선**: Curve25519 (X25519 ECDH)
- 128-bit 보안 강도
- 소형 임베디드(버즈)에서 연산 효율 우수
- side-channel 공격 저항성 설계 (Montgomery ladder 연산)

---

### 4.2 ECIES (Elliptic Curve Integrated Encryption Scheme)

**개념**

ECDH로 임시 공유 비밀을 만들고, 이를 대칭키로 변환하여 데이터를 암호화하는 하이브리드 암호화 방식이다. 수신자의 공개키만 알면 누구든 암호화할 수 있고, 복호화는 수신자의 개인키 없이는 불가능하다.

```
[암호화 — 버즈에서 실행]

1. 임시 키쌍 생성:  (d_eph, P_eph)  ← 스냅샷마다 새로 생성
2. ECDH:            S = d_eph · P_owner
3. 키 파생:         K_enc ‖ K_mac = HKDF(S, "audi-find-v1", 48 bytes)
4. 암호화:          C = AES-256-GCM(Mel-Spectrogram, K_enc)
                    → 인증 태그(GCM Tag, 16 bytes) 포함
5. 전송 페이로드:   [ 인덱스 | P_eph | C ]
                    ← d_eph는 절대 전송하지 않음

[복호화 — 소유자 스마트폰에서 실행]

1. ECDH:            S = d_owner · P_eph  (= d_eph · P_owner, ECDH 교환법칙)
2. 키 파생:         K_enc ‖ K_mac = HKDF(S, "audi-find-v1", 48 bytes)
3. 복호화:          Mel-Spectrogram = AES-256-GCM 복호화(C, K_enc)
                    → GCM 인증 태그 검증 포함 (무결성 확인)
```

**핵심 보안 특성**

클라우드는 P_eph와 암호문 C만 보관한다. 복호화에 필요한 d_owner는 소유자 스마트폰에만 존재하며, 클라우드에 전달되지 않는다. 따라서 클라우드는 구조적으로 복호화가 불가능하다.

---

### 4.3 AES-256-GCM

**개념**

AES(Advanced Encryption Standard)의 256-bit 키 모드에 GCM(Galois/Counter Mode)을 결합한 인증 암호화(AEAD) 방식이다.

```
[AES-256-GCM 구조]

입력:  평문(Mel-Spectrogram), 키(K_enc, 32 bytes), Nonce(IV, 12 bytes), AAD(추가 인증 데이터)
출력:  암호문(C) + 인증 태그(GCM Tag, 16 bytes)

동작:
  CTR 모드로 평문 암호화 → C
  GHASH 함수로 C + AAD 인증 → Tag

복호화 시:
  Tag 검증 실패 → 복호화 거부 (무결성 + 인증 보장)
```

**Audi-Find 선택 이유**

- **기밀성(Confidentiality)**: AES-256 CTR 모드로 Mel-Spectrogram 암호화
- **무결성(Integrity)**: GCM Tag로 전송 중 데이터 변조 탐지
- **인증(Authentication)**: AAD에 인덱스를 포함시켜 인덱스-암호문 쌍의 결합을 인증
- BES2700YP의 Cortex-M55 Helium 명령어(VAESEQ/VAESIMCQ)로 하드웨어 가속 가능

```
AAD 구성 예시:
  AAD = [ 인덱스(32 bytes) | 타임스탬프(8 bytes) | 버전(2 bytes) ]
  → 인덱스를 바꿔치기해도 Tag 검증 실패 → 공격 방어
```

---

### 4.4 HKDF (HMAC-based Key Derivation Function)

**개념**

하나의 공유 비밀(ECDH 출력 등 고엔트로피 값)에서 목적별로 독립된 여러 키를 안전하게 파생하는 함수다. RFC 5869에 정의되어 있으며, HMAC-SHA-256을 내부 PRF로 사용한다.

```
[HKDF 2단계]

1. Extract 단계:
   PRK = HMAC-SHA-256(salt, 입력 키 재료(IKM))
   → 균일 분포의 의사랜덤 키(PRK) 생성

2. Expand 단계:
   OKM = T(1) ‖ T(2) ‖ ...
   T(i) = HMAC-SHA-256(PRK, T(i-1) ‖ info ‖ i)

Audi-Find 적용:
   IKM  = ECDH 공유 비밀 S
   info = "audi-find-v1"  ← 도메인 분리 레이블
   L    = 48 bytes
   출력  = K_enc(32 bytes) ‖ K_mac(16 bytes)
```

단일 ECDH 결과를 그대로 AES 키로 사용하면 엔트로피 편향 위험이 있다. HKDF를 통해 균일 분포의 독립 키를 파생한다.

---

### 4.5 롤링 익명 인덱스 — SHA-256 기반

**개념**

Apple Find My의 서버 인덱스 방식을 차용한다. 소유자를 직접 식별하는 정보 대신, 주기적으로 교체되는 임시 공개값의 해시를 인덱스로 사용한다.

```
[인덱스 생성]

마스터 시크릿: SK_0  ← 256-bit 난수, 버즈-스마트폰 최초 페어링 시 생성
카운터:        i     ← 스냅샷마다 증가

롤링 공개값: A_i = KDF(SK_{i-1}, "audi-find-index", i)
서버 인덱스: Index_i = SHA-256(A_i)   ← 32 bytes

BLE 광고: [ Index_i | 서비스 UUID ]   ← 31 bytes 이내
GATT 전송: [ Index_i | P_eph | C ]    ← 클라우드 업로드 단위
```

```
[Find 요청 시 인덱스 조회]

스마트폰:
  SK_0 (페어링 시 동기화) → A_i 재계산 → SHA-256(A_i) → 서버에 인덱스 제시
  클라우드: Index_i로 [ P_eph | C ] 반환

클라우드 관점:
  Index_i = SHA-256(A_i)만 보관. A_i 자체도 모름.
  → 어느 사용자의 데이터인지 알 수 없음.
```

**보안 특성**

SHA-256의 단방향성 덕분에 Index_i로부터 A_i나 SK_0을 역산할 수 없다. 카운터 i가 증가할 때마다 Index_i가 완전히 달라져 연속 스냅샷을 연결(link)할 수 없다.

---

### 4.6 키 윈도우 — 동기화 실패 대응

**개념**

버즈와 스마트폰의 키 동기화가 BLE 단절로 인해 어긋날 수 있다. 버즈가 최근 N개의 임시 키쌍을 보관하여 동기화 실패를 허용 범위 내에서 복원한다.

```
[버즈 내부 키 버퍼]

순환 버퍼: [ (d_eph_1, P_eph_1, T_1), ..., (d_eph_N, P_eph_N, T_N) ]

스냅샷 생성 시:
  새 임시 키쌍 (d_eph_i, P_eph_i) 생성
  버퍼에 추가, 가장 오래된 항목 삭제

[Find 요청 시]
  클라우드가 P_eph_i를 포함한 페이로드 반환
  → 스마트폰은 P_eph_i에 대응하는 복호화를 수행 (키 식별이 P_eph_i로 이루어짐)
  → 동기화 시점과 무관하게 복호화 가능

N 권장값: 3 (키 교체 주기 15분 × 3 = 45분치 커버)
메모리: (32 + 32 + 8) bytes × 3 = 216 bytes (BES2700YP 4MB SRAM 기준 무시 가능)
```

---

### 4.7 TLS 1.3 + Certificate Pinning

**개념**

**TLS 1.3 (RFC 8446)**: 스마트폰과 클라우드 LALM 엔진 사이의 전송 계층 보안 프로토콜이다. 1-RTT 핸드셰이크, 전방 비밀성(Forward Secrecy) 필수, 취약 암호 스위트 제거가 특징이다.

```
[TLS 1.3 핸드셰이크 — 전방 비밀성]

ClientHello  → 지원 암호 스위트, 키 공유값(ECDHE)
ServerHello  ← 선택된 암호 스위트, 키 공유값, 서버 인증서
               (이후 모든 통신 암호화됨)

→ 세션 키는 ECDHE로 매 세션 새로 생성. 서버 개인키가 유출돼도 과거 세션 복호화 불가.
```

**Certificate Pinning**

Samsung Find 앱이 특정 서버 인증서(또는 공개키 해시)만을 신뢰하도록 하드코딩한다.

```
[Pinning 구현 — SPKI(SubjectPublicKeyInfo) 핀 방식]

앱 내 하드코딩:
  EXPECTED_PIN = SHA-256(서버 공개키의 DER 인코딩)

TLS 핸드셰이크 중 검증:
  수신된 서버 인증서 공개키 → SHA-256 계산 → EXPECTED_PIN과 비교
  불일치 → 연결 즉시 차단

방어 대상:
  신뢰할 수 없는 CA가 발급한 가짜 인증서를 통한 MITM 공격
  공공 WiFi에서의 SSL Inspection 장비
```

---

### 4.8 TEE (Trusted Execution Environment)

**개념**

클라우드 서버에서 LALM 추론이 실행될 때 복호화된 Mel-Spectrogram(평문)이 서버 메모리에 존재한다. TEE는 이 평문이 OS, 하이퍼바이저, 인프라 관리자 등 서버 운영자조차 접근할 수 없는 격리된 메모리 영역에서만 처리되도록 보장한다.

```
[TEE 동작 원리]

일반 서버 프로세스:
  OS → 하이퍼바이저 → 앱 프로세스
  ↕ OS/관리자 접근 가능

TEE (예: AWS Nitro Enclave):
  Enclave = 격리된 가상 머신
    - 호스트 OS/EC2 인스턴스와 메모리 완전 분리
    - 외부 네트워킹 없음 (vsock만 허용)
    - 원격 증명(Remote Attestation): 특정 코드만 실행 중임을 암호학적으로 증명
    - AWS 직원도 Enclave 내부 메모리 접근 불가
```

```
[Audi-Find TEE 적용 흐름]

스마트폰 → vsock → Nitro Enclave (격리)
                      복호화된 Mel-Spectrogram 수신
                      → Qwen3.5-Omni-Plus 추론
                      → 위치 결과만 외부로 반환
                      → Mel-Spectrogram 메모리에서 즉시 삭제
```

**가용 TEE 옵션**

| 플랫폼 | 제품명 | 특징 |
|---|---|---|
| AWS | Nitro Enclaves | EC2 기반, vsock 통신, 원격 증명 |
| Azure | Confidential Computing (AMD SEV-SNP) | VM 레벨 격리 |
| Google | Confidential VMs (AMD SEV) | GKE 통합 |

---

### 4.9 BLE 광고/GATT 역할 분리

**개념**

BLE는 두 가지 통신 모드를 가진다.

```
[광고 모드 — Advertising]
  연결 없이 브로드캐스트. 페이로드 최대 31 bytes.
  모든 주변 BLE 기기가 수신 가능.
  용도: 기기 존재 알림, 최소 식별 정보 전달.

[연결 모드 — GATT (Generic Attribute Profile)]
  1:1 BLE 연결 후 데이터 교환.
  DLE 활성화 시 패킷당 최대 244 bytes 페이로드.
  대용량 데이터 전송 가능.
  용도: 파일 전송, 대용량 센서 데이터.
```

```
[Audi-Find BLE 분리 구조]

광고 페이로드 (31 bytes):
  [ 서비스 UUID (16 bytes) | Index_i 앞 12 bytes ]
  → Finder 기기에 "이 기기가 Audi-Find 스냅샷을 보유 중"임을 알림

연결 후 GATT 전송 (250 KB):
  Finder가 버즈에 BLE 연결 요청
  GATT Notify/Indicate로 [ Index_i | P_eph | C ] 수신
  클라우드에 업로드
```

---

## 5. 개선된 전체 프로토콜 흐름

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 0: 초기화 (최초 페어링 시)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

버즈 ↔ 스마트폰:
  마스터 시크릿 SK_0 생성 (버즈에서 난수 생성)
  SK_0 → 스마트폰에 BLE 보안 채널으로 전달 → 보관
  스마트폰의 EC Curve25519 키쌍 생성: (d_owner, P_owner)
  P_owner → 버즈에 전달 → 저장 (암호화에 사용)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 1: 오디오 수집 및 암호화 (BLE 연결 중 상시 동작)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[버즈]
  ANC 외부 마이크 → Raw PCM (16kHz, 10초 순환 버퍼)
  Mel-Spectrogram 변환 (128ch, 25ms window, 10ms hop) → 250 KB (float16)

  임시 키쌍 생성:   (d_eph_i, P_eph_i)  ← 스냅샷마다 새로 생성
  롤링 인덱스:       A_i = KDF(SK_{i-1}, "audi-find-index", i)
                    Index_i = SHA-256(A_i)

  공유 비밀:        S = ECDH(d_eph_i, P_owner)   [Curve25519]
  키 파생:          K_enc ‖ K_mac = HKDF(S, "audi-find-v1", 48 bytes)
  암호화:           C = AES-256-GCM(Mel-Spectrogram, K_enc, AAD=Index_i)

  BLE 광고:        [ 서비스 UUID | Index_i[:12] ]   (31 bytes)
  키 버퍼:         [ (d_eph_i, P_eph_i, T_i) ] × N개 순환 보관

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 2: Finder → 클라우드 업로드
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[주변 Galaxy 기기 (Finder)]
  광고 감지 → 버즈에 BLE 연결
  GATT로 [ Index_i | P_eph_i | C ] 수신 (250 KB, ~3초)
  클라우드에 업로드 (Finder 자신의 신원·GPS 미포함)

[클라우드]
  [ Index_i | P_eph_i | C ] 저장
  → K_enc 없음. 복호화 불가.
  → Index_i가 SHA-256 값이므로 소유자 식별 불가.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 3: Find 요청 — 스마트폰 복호화 → LALM 추론
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[소유자 스마트폰]
  A_i 재계산 → Index_i = SHA-256(A_i) → 클라우드에 제시
  클라우드 → 스마트폰: [ P_eph_i | C ] 반환

  (스마트폰에서 복호화)
  S = ECDH(d_owner, P_eph_i)
  K_enc = HKDF(S, "audi-find-v1", 48 bytes)[:32]
  Mel-Spectrogram = AES-256-GCM 복호화(C, K_enc, AAD=Index_i)
  → GCM 태그 검증 통과 → 무결성 확인

  TLS 1.3 + cert pinning 채널로
  → Mel-Spectrogram + GPS 후보 텍스트 → 클라우드 TEE 전송

[클라우드 TEE (Nitro Enclave)]
  Mel-Spectrogram + 프롬프트 수신
  Qwen3.5-Omni-Plus 추론 실행
  위치 결과만 외부 반환
  Mel-Spectrogram 즉시 메모리 삭제

[Samsung Find 앱]
  자연어 위치 결과 + 지도 표시
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 6. 취약점 해소 결과

| 취약점 | 원본 | 개선 후 | 적용 기술 |
|---|---|---|---|
| 클라우드 키+암호문 동시 보유 | ❌ | ✅ 제거 | ECIES (스마트폰 복호화) |
| 헤더 소유자 직접 식별 | ❌ | ✅ 익명화 | SHA-256 롤링 인덱스 |
| 재전송 공격 / MITM | ❌ | ✅ 방어 | TLS 1.3 + cert pinning |
| 키 교체 동기화 실패 | ❌ | ✅ 허용 오차 | 키 윈도우 (N=3) |
| BLE 광고 크기 초과 | ❌ | ✅ 해결 | 광고/GATT 분리 |
| 클라우드 내부 평문 노출 | (미고려) | ✅ 격리 | TEE (Nitro Enclave) |

---

## 7. 남은 한계

**한계 1: 스마트폰이 온라인이어야 복호화 가능**

스마트폰이 네트워크에서 끊긴 상태(비행기 모드 등)에서는 Find 불가.
버즈 분실 상황에서 소유자가 스마트폰을 들고 있는 것이 일반적이므로 현실적 문제는 제한적이다.

**한계 2: 스마트폰과 버즈 동시 분실**

d_owner가 스마트폰에만 있는 경우, 스마트폰도 함께 분실 시 복호화 불가.
대응: Samsung 계정 서버에 d_owner를 E2E 암호화하여 백업, 다른 Galaxy 기기에서 복원 가능하도록 설계.

**한계 3: TEE 도입 비용**

AWS Nitro Enclaves 등 TEE 기반 추론은 일반 인스턴스 대비 운영 복잡도와 비용이 높다.
초기 단계에서는 TLS 1.3 + 즉시 폐기 정책으로 우선 운영하고, 서비스 확대 시 TEE로 전환하는 단계적 접근이 현실적이다.

---

## References

- [ECIES — SEC 1 v2.0, Standards for Efficient Cryptography](https://www.secg.org/sec1-v2.pdf)
- [X25519 ECDH — RFC 7748](https://www.rfc-editor.org/rfc/rfc7748)
- [AES-GCM — NIST SP 800-38D](https://csrc.nist.gov/publications/detail/sp/800-38d/final)
- [HKDF — RFC 5869](https://www.rfc-editor.org/rfc/rfc5869)
- [TLS 1.3 — RFC 8446](https://www.rfc-editor.org/rfc/rfc8446)
- [AWS Nitro Enclaves](https://aws.amazon.com/ec2/nitro/nitro-enclaves/)
- [Apple Find My 암호화 — Matthew Green (2019)](https://blog.cryptographyengineering.com/2019/06/05/how-does-apple-privately-find-your-offline-devices/)
- [Samsung Find 프라이버시 분석 — USENIX Security 2024 (arXiv 2210.14702)](https://arxiv.org/abs/2210.14702)
- [ARM Cortex-M55 Helium AES 명령어 — ARM TRM](https://www.arm.com/products/silicon-ip-cpu/cortex-m/cortex-m55)
