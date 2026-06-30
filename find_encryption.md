# Samsung Find vs Apple Find My: 암호화 구현 비교

> Samsung SmartThings Find(Offline Finding)와 Apple Find My Network의
> 암호화 프로토콜 및 개인정보 보호 설계를 비교 정리한다.
> 출처: 학술 분석 논문(arXiv 2210.14702, USENIX Security 2024), Apple 공식 보안 문서, 암호학 블로그.

---

## 1. 공통 동작 구조

두 시스템 모두 **크라우드소싱 기반 Offline Finding** 방식을 사용한다.

```
분실 기기
  → BLE 신호 브로드캐스트 (공개키 또는 식별자 포함)
  → 주변 기기(Finder)가 신호 수신
  → Finder가 자신의 GPS 위치를 암호화하여 서버에 업로드
  → 분실 기기 소유자가 서버에서 암호화된 위치를 내려받아 복호화
```

핵심 목표는 **중계자(Finder)와 서버가 위치 정보를 알 수 없도록** 설계하는 것이다.

---

## 2. Apple Find My

### 2.1 설계 철학

Apple은 "서버가 위치를 알 수 없는(server-blind)" 구조를 핵심 원칙으로 삼는다.
Apple 서버는 암호화된 위치 블롭을 저장하지만, 복호화 키를 절대 보유하지 않는다.

### 2.2 키 생성 및 구조

분실 기기에서 **오프라인 찾기** 설정이 활성화될 때 키 쌍이 생성된다.

```
[초기 설정]

마스터 시크릿  SK₀  ← 256-bit 난수, 기기 내부에만 저장
카운터         i   ← 0으로 초기화

[키 파생 — 주기적 롤링]

SKᵢ  =  KDF(SKᵢ₋₁,  "update")     ← 이전 시크릿으로 다음 시크릿 유도
         (KDF: NIST SP 800-56A 기반 Concatenation KDF)

(uᵢ, vᵢ)  =  KDF(SKᵢ,  "diversify")   ← 두 정수 유도

dᵢ  =  uᵢ + vᵢ   (mod 곡선 위수)      ← EC P-224 개인키
Pᵢ  =  dᵢ · G                          ← EC P-224 공개키 (28 bytes)
```

- **키 교체 주기**: 약 **15분**마다 카운터 i를 증가시켜 새 키 쌍 생성
- **추적 방지**: 키가 15분마다 바뀌므로 동일 기기를 공개키로 추적하는 것이 불가능
- **SK₀는 Apple 서버에 절대 전송되지 않음**: iCloud Keychain을 통해 소유자의 다른 기기에만 E2E 암호화 동기화

### 2.3 BLE 브로드캐스트

```
분실 기기 BLE 페이로드:
  [ 공개키 Pᵢ (28 bytes) | 기타 메타데이터 ]

→ P-224 공개키를 단일 BLE 광고 패킷에 포함 가능한 크기로 설계
```

### 2.4 Finder 기기의 위치 암호화 (ECIES)

주변의 Apple 기기(Finder)가 BLE 신호를 수신하면 자신의 현재 위치를 암호화하여 업로드한다.

```
[위치 암호화 — ECIES (Elliptic Curve Integrated Encryption Scheme)]

1. Finder가 임시 EC P-224 키쌍 생성: (d_eph, P_eph)
2. ECDH: 공유 비밀 = d_eph · Pᵢ  (분실 기기의 공개키와 연산)
3. 공유 비밀 → KDF → 대칭키 (AES-GCM 또는 AES-CCM)
4. 위치 정보(위도·경도·타임스탬프) → 대칭키로 암호화
5. 암호문 + P_eph → Samsung 서버에 업로드

[서버 인덱스]
SHA-256(Pᵢ)  → Apple 서버에서 암호문을 조회하는 인덱스로 사용
```

서버는 암호문과 인덱스만 저장한다. **Pᵢ의 해시가 인덱스이므로 서버는 어느 기기의 위치인지 알 수 없다.**

### 2.5 소유자의 위치 복호화

```
소유자 기기(iPhone/Mac):
  1. 동일한 SKᵢ 재현 → dᵢ 재계산
  2. 서버 인덱스 = SHA-256(Pᵢ) → 서버에서 암호문 다운로드
  3. ECDH: dᵢ · P_eph → 동일한 공유 비밀 복원
  4. 공유 비밀 → 대칭키 → 위치 복호화
```

### 2.6 Apple Find My 암호화 요약

| 항목 | 내용 |
|---|---|
| **기기 키쌍 곡선** | EC P-224 |
| **키 교체 주기** | ~15분 (카운터 기반 롤링) |
| **위치 암호화** | ECIES (임시 ECDH + 대칭 암호화) |
| **서버 인덱스** | SHA-256(공개키) |
| **마스터 시크릿 저장** | 기기 내부 + iCloud Keychain (E2E 암호화) |
| **Apple 서버 접근 가능 정보** | 암호화된 위치 블롭만. 복호화 키 없음 |
| **Finder 익명성** | 업로드 요청에 인증 정보 없음 → Finder 신원 미상 |

---

## 3. Samsung SmartThings Find (Offline Finding)

### 3.1 설계 철학

Samsung도 크라우드소싱 구조를 사용하지만, Apple과 달리 **ECDH 기반 대칭키를 분실 기기와 소유자 기기가 사전에 공유**하는 방식으로 구현한다. 서버 업로드 전 위치를 암호화하는 "Encrypt Offline Location" 옵션을 별도로 제공한다.

### 3.2 등록(Registration) 단계 — 공유 비밀 수립

```
[기기 등록 시 ECDH 키 교환]

분실 기기(태그)                    소유자 기기(스마트폰)
  키쌍 생성: (d_tag, P_tag)
  P_tag 전송 ─────────────────────→ 수신
                                     키쌍 생성: (d_owner, P_owner)
  수신 ←───────────────────────────  P_owner 전송

  공유 비밀 = ECDH(d_tag, P_owner)   공유 비밀 = ECDH(d_owner, P_tag)
  ※ Curve25519 사용, 결과 동일       ※ ECDH 교환법칙

[마스터 시크릿 파생]
masterSecret = 공유 비밀의 앞 16 bytes

[서브키 파생 (총 6개, 각 16 bytes)]
PIDK = KDF(masterSecret, "privacy")   ← Privacy ID Key: BLE 광고 식별자 생성용
기타 5개 서브키 → 통신 채널 암호화, 인증 등 각기 다른 역할
```

- **곡선**: Curve25519
- **해시**: SHA-256
- **암호**: AES/CBC/PKCS7 (SUPPORTED_CIPHER 특성으로 협상)

### 3.3 BLE 브로드캐스트

```
분실 기기 BLE 페이로드:
  PIDK 기반으로 생성된 익명 식별자 포함
  → 주변 Galaxy 기기가 신호 수신 후 Samsung 서버에 위치 전달
```

### 3.4 위치 암호화 및 서버 업로드

```
[Finder 기기 동작]
  BLE 신호 수신 → 자신의 GPS 위치 확인
  → AES/CBC/PKCS7 으로 위치 암호화
    - IV: Java SecureRandom 기반 랜덤 nonce (서버 접속마다 새로 생성)
  → 암호화된 위치 + 식별자 → Samsung 서버 업로드

[인증 핸드셰이크]
  서버가 NONCE 특성에 IV 제공
  → 클라이언트는 "smartthings" 문자열을 해당 IV + 기기 시크릿키로 암호화
  → 올바른 암호문을 서버에 전달하면 핸드셰이크 완료
```

### 3.5 Encrypt Offline Location (선택 기능)

Samsung은 기본 암호화 외에 추가 보호 옵션을 제공한다.

```
활성화 시:
  오프라인 위치 → Finder 기기의 스마트폰을 떠나기 전에 암호화
  → 서버는 이중 암호화된 위치만 보관
  → 소유자가 위치 확인 시 6자리 PIN 입력 필요
```

### 3.6 Samsung SmartThings Find 암호화 요약

| 항목 | 내용 |
|---|---|
| **키 교환** | ECDH on Curve25519 |
| **해시** | SHA-256 |
| **위치 암호화** | AES/CBC/PKCS7 |
| **IV 생성** | Java SecureRandom (서버 접속마다 갱신) |
| **서브키 수** | 6개 (16 bytes 각, masterSecret에서 파생) |
| **추가 보호** | Encrypt Offline Location (6자리 PIN) |
| **Samsung 서버 접근 가능 정보** | 암호화된 위치 블롭. 기본값에서 복호화 가능성 연구됨 |

---

## 4. 비교 요약

| 항목 | Apple Find My | Samsung SmartThings Find |
|---|---|---|
| **키 교환 방식** | EC P-224 롤링 키쌍 (단방향) | ECDH Curve25519 (쌍방 공유 비밀) |
| **위치 암호화** | ECIES (비대칭 → 대칭) | AES/CBC/PKCS7 (대칭) |
| **키 교체 주기** | **~15분** (카운터 기반 자동) | 등록 시 1회. 주기적 롤링 미확인 |
| **서버 인덱스** | SHA-256(공개키) → 서버가 기기 특정 불가 | PIDK 기반 익명 식별자 |
| **서버의 복호화 가능 여부** | ❌ 불가 (복호화 키 없음) | △ 조건부 (연구에서 취약점 일부 지적) |
| **Finder 익명성** | ✅ 업로드 요청에 인증 정보 없음 | △ 서버 인증 핸드셰이크 포함 |
| **마스터 시크릿 동기화** | iCloud Keychain (E2E 암호화) | Samsung 서버 경유 가능성 |
| **추가 보호 옵션** | 없음 (기본 설계가 최고 수준) | Encrypt Offline Location + PIN |
| **학술 분석** | Cryptography Engineering 블로그 (2019) | USENIX Security 2024 (arXiv 2210.14702) |

### 핵심 차이

**Apple**: 공개키가 15분마다 바뀌고, 서버 인덱스는 공개키의 해시값이다. 서버는 어떤 기기의 위치인지, 누가 소유자인지 알 수 없으며, 복호화 키도 없다. 설계 자체가 서버 맹목(server-blind)이다.

**Samsung**: ECDH로 소유자와 기기가 공유 비밀을 수립한 뒤 AES 대칭키로 암호화한다. 구조적으로 서버가 중개 역할을 하며, USENIX Security 2024 논문에서 일부 프라이버시 취약점(재식별 가능성 등)이 지적되었다.

---

## 5. Audi-Find 관련 시사점

Audi-Find는 Galaxy Buds의 마이크 데이터를 클라우드로 전송하는 별도 파이프라인을 설계한다.
Samsung Find의 기존 암호화 구조(ECDH + AES-CBC)는 위치 좌표 전송에 최적화되어 있으며,
Mel-Spectrogram 벡터(250 KB) 전송에는 다른 채널과 키 관리 방식이 필요하다.

Audi-Find에서는 Apple의 접근법을 참고하여 **서버가 Mel-Spectrogram 원문을 절대 복호화할 수 없는**
설계를 목표로 한다 — TLS 1.3 + cert pinning + 서버 측 즉시 폐기.

---

## References

- [Privacy Analysis of Samsung's Crowd-Sourced Bluetooth Location Tracking System (arXiv 2210.14702)](https://arxiv.org/abs/2210.14702)
- [Security and Privacy Analysis of Samsung's Crowd-Sourced Bluetooth Location — USENIX Security 2024](https://www.usenix.org/system/files/usenixsecurity24-yu-tingfeng.pdf)
- [Find My Security — Apple Platform Security Guide](https://support.apple.com/guide/security/find-my-security-sec6cbc80fd0/web)
- [How does Apple (privately) find your offline devices? — Matthew Green, Cryptography Engineering Blog (2019)](https://blog.cryptographyengineering.com/2019/06/05/how-does-apple-privately-find-your-offline-devices/)
- [Systematizing the Offline Finding Landscape — Cornell Tech (2025)](https://tech.cornell.edu/wp-content/uploads/2025/09/SETS-Draft-Summary.pdf)
- [Securing the invisible thread: BLE tracker security in AirTags and SmartTags — Springer Nature (2025)](https://link.springer.com/article/10.1007/s43995-025-00248-4)
- [Samsung SmartThings Find 공식 안내](https://support.smartthings.com/hc/en-us/articles/10863369660052-SmartThings-Find)
- [BES2700YP 데이터시트 — Bestechnic](https://www.bestechnic.com/en/article/78/64.html)
