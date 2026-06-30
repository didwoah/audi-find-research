# Samsung Find GPS 위치 오차 조사 결과

> Audi-Find의 GPS 후보 위치 반경 기준 설정을 위한 조사.
> 웹 검색 및 공식 문서 기반.

---

## 조사 목적

Map API에서 GPS 좌표 기반으로 위치 후보를 선정할 때, 반경 몇 미터를 기준으로 삼을 것인지 결정하기 위해 Samsung Find(SmartThings Find)의 실내 위치 오차 범위를 조사한다.

---

## Samsung Find 위치 추적 방식

Samsung Find(SmartThings Find)는 두 가지 방식으로 기기 위치를 추적한다.

**1. Online Finding (직접 연결)**
기기가 스마트폰과 BLE로 연결된 상태에서 GPS 또는 Wi-Fi 위치를 직접 수신한다. 실내에서는 GPS 신호 약화로 정확도가 저하된다.

**2. Offline Finding (크라우드소싱)**
기기가 스마트폰과의 직접 연결이 끊어진 상태에서, 주변을 지나는 다른 Galaxy 사용자의 기기가 BLE 신호를 감지하여 위치를 암호화 전송하는 방식이다. 기기가 꺼진 상태에서도 최대 24시간까지 동작한다.

---

## 방식별 위치 오차

| 추적 방식 | 환경 | 오차 범위 | 비고 |
|---|---|---|---|
| GPS (실외) | 개방 공간 | **3~5미터** | 표준 민간 GPS 정확도 |
| GPS (실내) | 목조 건물 | **5미터 이내** | 신호 약화 |
| GPS (실내) | 콘크리트/벽돌 건물 | **10미터 이내** | 다중경로 및 위성 신호 차단 |
| BLE 실내 탐지 | 실내 (장애물 있음) | **~10미터** | 유효 탐지 반경 기준 |
| Offline Finding | 도시 (서울 등) | **중앙값 22미터** | 크라우드소싱 밀도 의존 |
| Offline Finding | 농촌 (밀도 낮음) | **중앙값 4.7미터** | Samsung 기기 밀도 낮음 |
| Offline Finding | 최악의 경우 | **최대 100미터** | 공식 Samsung 안내 기준 |

---

## 실내 환경의 GPS 한계

GPS 신호는 위성에서 수신하는 전파이므로 건물 내부에서는 다음 두 가지 문제가 발생한다.

- **신호 약화**: 콘크리트, 금속 구조물이 신호를 흡수하거나 차단
- **다중경로(Multipath)**: 벽과 구조물에 반사된 신호가 수신기에 도달해 위치 계산 오류 발생

지하철역, 대형 쇼핑몰, 지하 주차장 등에서는 GPS 신호가 거의 수신되지 않아 SmartThings Find가 Offline Finding(BLE 크라우드소싱)으로 전환된다. 이 경우 표시 위치와 실제 위치의 차이가 최대 100미터까지 벌어질 수 있다.

---

## Audi-Find 적용 기준

위 조사 결과를 바탕으로 아래와 같이 결정한다.

**GPS 후보 위치 선정 반경: 100미터**

이는 Samsung Find의 Offline Finding 최대 오차(100미터)를 포괄하는 보수적 기준이다. 반경 100미터 이내의 모든 장소를 Map API를 통해 추출하여 LLM의 위치 후보 목록으로 제공한다.

| 근거 | 수치 |
|---|---|
| GPS 실내 오차 (콘크리트) | ~10m |
| BLE 실내 탐지 반경 | ~10m |
| Offline Finding 최대 오차 | **100m** (Samsung 공식) |
| **채택 기준** | **100m** |

---

## References

- [Samsung 공식: SmartThings Find 위치 정확도 안내](https://www.samsung.com/us/support/answer/ANS00080656/)
- [SmartThings Find vs Apple Find My 실측 비교](https://www.startupbooted.com/smartthings-find)
- [Indoor Positioning Using GPS Revisited (ResearchGate)](https://www.researchgate.net/publication/221016004_Indoor_Positioning_Using_GPS_Revisited)
- [GPS vs Indoor Positioning 비교 — BlueIOT](https://www.blueiot.com/blog/gps-vs-indoor-positioning.html)
- [Samsung 공식: 스마트폰 GPS 부정확 원인 안내](https://www.samsung.com/levant/support/mobile-devices/why-is-in-accurate-gps-location-shown-on-my-phone/)
