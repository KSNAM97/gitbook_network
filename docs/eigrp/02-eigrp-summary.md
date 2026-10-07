# EIGRP 통합 정리 — DUAL·3개 테이블·Successor/FS 한눈에

> **Tag:** #EIGRP #Routing #IGP #Cisco #DUAL #AdvancedDistanceVector #Summary
> **핵심 요약:** EIGRP는 Cisco가 개발한 **Advanced Distance Vector IGP**로, DUAL 알고리즘을 이용해 루프 없는 경로를 계산한다. 현재 최적 경로는 Successor, Feasibility Condition을 만족하는 Loop-free Backup은 Feasible Successor다. FS가 유효하면 장애 시 Query 없이 빠르게 전환하고, FS가 없으면 경로를 Active 상태로 전환해 이웃에게 Query한다. SIA는 단순히 FS가 없어서가 아니라 Active 경로에 대한 Reply가 제한 시간 내 돌아오지 않을 때 발생하며, Stub·주소 요약·계층화 설계로 Query 범위를 줄이는 것이 핵심이다.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. OSPF가 있는데 왜 EIGRP를 사용하나요?

👉 **핵심 답변:** EIGRP는 DUAL 기반의 빠른 수렴, 부분적·제한적 Update, 복합 Metric, Unequal-cost Load Balancing 등의 장점이 있어 Cisco 중심의 캠퍼스나 WAN에서 유용합니다.

- **꼬리 질문 대비 설명:**
  - EIGRP는 흔히 Hybrid라고 설명하지만 Cisco 공식 분류에 가까운 표현은 **Advanced Distance Vector**입니다.
  - 멀티벤더 환경과 표준 중심 설계에는 OSPF/IS-IS가 일반적으로 적합
  - EIGRP 동작은 RFC 7868에 정보 제공 목적의 RFC로 공개됐지만 실제 구현 생태계는 Cisco 중심
  - OSPF는 Reference Bandwidth로 Cost를 계산하고, EIGRP는 기본 K값에서 최소 Bandwidth와 누적 Delay를 사용
  - 변화된 경로 정보를 필요한 이웃에게만 전달하는 Partial/Bounded Update로 대역폭을 절감

---

### Q2. EIGRP의 3개 테이블은 무엇인가요?

| 테이블 | 역할 | 명령어 |
|---|---|---|
| Neighbor Table | Hello로 형성된 인접 Router 관리 | `show ip eigrp neighbors` |
| Topology Table | 목적지별 Successor·FS·기타 학습 경로 관리 | `show ip eigrp topology` |
| Routing Table | 선택된 Successor를 RIB에 설치 | `show ip route eigrp` |

전체 학습 경로 확인:

```cisco
show ip eigrp topology all-links
```

> 기본 `show ip eigrp topology` 출력은 플랫폼에 따라 Successor와 Feasible Successor 중심으로 표시될 수 있습니다. 모든 경로를 확인하려면 `all-links`를 사용합니다.

---

### Q3. Successor와 Feasible Successor의 차이는?

👉 **핵심 답변:** Successor는 현재 최적 경로이고, Feasible Successor는 Feasibility Condition을 만족하는 Loop-free Backup 경로입니다.

```properties
Successor
= 가장 낮은 Feasible Distance를 가진 현재 최적 경로
= Routing Table에 설치되는 경로

Feasible Successor
= Feasibility Condition을 만족하는 대체 경로
= Successor 장애 시 Query 없이 빠르게 전환 가능
```

#### Feasibility Condition

```properties
Neighbor의 Reported Distance < 현재 Successor의 Feasible Distance
```

```text
RD < Successor FD
```

- RD: 이웃이 목적지까지 도달하는 데 필요하다고 보고한 Metric
- FD: 현재 Router가 목적지까지 계산한 최저 Metric
- RD는 Advertised Distance라고도 하지만 Administrative Distance와 혼동하지 않도록 RD 표현 권장

> 최적 경로가 아니라고 모두 FS가 되는 것은 아닙니다. Feasibility Condition을 만족하지 않는 경로도 Topology Table에 존재할 수 있지만 즉시 사용할 수 있는 FS는 아닙니다.

---

### Q4. FS가 있으면 정말 즉시 전환되나요?

FS가 현재도 유효하고 Feasibility Condition을 만족하면 DUAL은 Query 없이 해당 경로를 새 Successor로 선택할 수 있습니다.

```properties
Successor 장애
    ↓
유효한 FS 존재
    ↓
Local Computation
    ↓
새 Successor 선택
    ↓
RIB/FIB 갱신
```

> 실제 전환 시간에는 장애 감지 시간, Neighbor 제거, DUAL 처리, RIB/FIB 갱신 시간이 포함됩니다. FS가 있다고 물리 장애 감지까지 0초가 되는 것은 아닙니다.

---

### Q5. Passive와 Active 상태는 무엇인가요?

| 상태 | 의미 |
|---|---|
| Passive | 경로가 안정적이고 DUAL 계산이 완료된 정상 상태 |
| Active | Successor를 잃고 FS가 없어 이웃에게 Query 중인 상태 |

```properties
Successor 장애
    ↓
FS 없음
    ↓
경로 Active
    ↓
이웃에게 Query
    ↓
모든 필요한 Reply 수신
    ↓
새 경로 선택 후 Passive
```

> EIGRP에서 Passive는 정상 상태입니다. `passive-interface`와는 다른 개념입니다.

---

### Q6. SIA는 왜 발생하나요?

👉 **핵심 답변:** Active 상태의 경로에 대해 필요한 Reply가 Active Timer 내에 돌아오지 않을 때 SIA가 발생합니다.

주요 원인:

- Query 범위가 지나치게 큼
- 저속·혼잡 WAN
- Packet Loss
- Router CPU 과부하
- Neighbor가 다른 Query 처리를 기다리는 구조
- 잘못된 Hub-and-Spoke 설계
- 불안정한 Link
- 과도한 재분배 또는 경로 수

```properties
FS 부재
→ Query 시작의 조건

Reply 미수신
→ SIA의 직접 원인
```

근본 대책:

- EIGRP Stub
- 주소 요약
- 계층화된 Topology
- Query 범위 제한
- WAN 품질 및 CPU 문제 해결

---

## 2. 표준 설정 템플릿

### 2-1. Classic EIGRP 기본 설정

```cisco
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface Serial1/1
 no passive-interface FastEthernet0/0
 network 13.13.1.1 0.0.0.0
 network 13.13.10.1 0.0.0.0
 network 150.1.13.1 0.0.0.0
```

> 최신 IOS/IOS XE에서는 Auto Summary가 기본 비활성인 경우가 많지만 Classic EIGRP 설정에서는 의도를 명확히 하기 위해 `no auto-summary`를 명시할 수 있습니다.

`network <IP> 0.0.0.0`은 해당 IP를 가진 Interface에서 EIGRP를 활성화합니다. Host Route만 광고한다는 의미가 아니라 해당 Interface의 Connected Prefix가 EIGRP 대상이 됩니다.

---

### 2-2. Named EIGRP 참고

```cisco
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
  network 13.13.1.1 0.0.0.0
  network 13.13.10.1 0.0.0.0
  af-interface default
   passive-interface
  exit-af-interface
  af-interface Serial1/1
   no passive-interface
  exit-af-interface
 exit-address-family
```

> Named Mode는 Address Family 기반이며 Wide Metric, HMAC-SHA-256 등 일부 현대 기능을 지원합니다. 정확한 기능은 플랫폼과 IOS XE 버전을 확인합니다.

---

### 2-3. Frame Relay Multipoint

```cisco
interface Serial1/0.123 multipoint
 no ip split-horizon eigrp 100
```

> Hub Multipoint Interface에서 Spoke A의 경로를 Spoke B로 광고해야 할 때 적용합니다. Frame Relay는 레거시 기술이며, 가능하면 Point-to-Point Subinterface로 분리하는 것이 안전합니다.

---

### 2-4. Classic EIGRP MD5 인증

```cisco
key chain EIGRP_AUTH
 key 20240101
  key-string <EIGRP_SECRET>
!
interface Serial1/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_AUTH
```

- 양단 Authentication Mode 일치
- 활성 Key ID와 Key-string 일치
- Key Lifetime 사용 시 NTP 동기화
- 인증 적용 순간 Neighbor Reset 가능

> MD5는 레거시 방식입니다. 지원 플랫폼의 Named EIGRP에서는 HMAC-SHA-256 등 더 강한 인증 방식을 검토합니다.

---

### 2-5. Hello/Hold Timer

```cisco
interface Serial1/0.123
 ip hello-interval eigrp 100 5
 ip hold-time eigrp 100 15
```

- 고속 Interface 및 Point-to-Point: 일반적으로 5/15초
- 저속 NBMA Interface: 일반적으로 60/180초
- 정확한 기본값은 Interface 종류와 Bandwidth에 따라 확인

> EIGRP의 Hello/Hold Timer는 OSPF처럼 양단 값이 반드시 같아야 Neighbor가 형성되는 것은 아닙니다. 각 Router가 자신의 Hold Time을 Hello에 알립니다. 다만 운영 예측성과 장애 감지 일관성을 위해 양단을 동일하게 구성하는 것이 권장됩니다.

---

### 2-6. EIGRP Stub

```cisco
router eigrp 100
 eigrp stub connected summary
```

일반적으로 Hub-and-Spoke의 Spoke Router에 적용합니다.

```properties
Connected = Connected Route 광고
Summary   = Summary Route 광고
```

필요에 따라:

```cisco
eigrp stub connected static summary redistributed
```

> 실제로 필요한 Route 유형만 광고하도록 최소 범위로 구성합니다.

---

### 2-7. 주소 요약

```cisco
interface FastEthernet0/0
 ip summary-address eigrp 100 13.13.0.0 255.255.0.0
```

효과:

- Routing Table 축소
- 상세 Prefix 변화 은닉
- Query 범위 제한
- Local Summary Route를 Null0로 생성

> 로컬에서 생성되는 EIGRP Summary Null0 Route의 기본 AD가 5입니다. 이 값을 모든 원격 Summary Route의 일반적인 AD라고 해석하면 안 됩니다.

---

### 2-8. Unequal-cost Load Balancing

```cisco
router eigrp 100
 maximum-paths 4
 variance 2
```

조건:

```properties
1. 대체 경로가 Feasibility Condition 만족
2. 대체 경로 Metric ≤ Successor Metric × Variance
3. Maximum-paths 범위 안
```

> `variance`만 설정한다고 Feasibility Condition을 만족하지 않는 경로가 Load Balancing에 사용되는 것은 아닙니다.

---

### 2-9. Metric 기본 공식

Classic Metric의 기본 K값:

```properties
K1=1, K2=0, K3=1, K4=0, K5=0
```

단순화된 기본 공식:

```properties
Classic EIGRP Metric
= 256 × [(10^7 / Minimum Bandwidth in Kbps)
         + Sum of Delay in tens of microseconds]
```

- Bandwidth는 경로 중 가장 낮은 Interface Bandwidth
- Delay는 경로 전체의 누적 Delay
- Load와 Reliability는 기본 K값에서는 사용하지 않음
- MTU는 Metric 계산에 직접 사용되지 않음
- Named Mode의 Wide Metric은 계산 방식과 Scale이 다름

---

### 2-10. 주요 파라미터

| 항목 | 값/설명 |
|---|---|
| IP Protocol | 88 |
| Multicast | 224.0.0.10 |
| AD Internal | 90 |
| Local Summary Null0 AD | 5 |
| AD External | 170 |
| 기본 Metric | Minimum BW + Cumulative Delay |
| 기본 K값 | 1,0,1,0,0 |
| 일반 Hello/Hold | 5/15초 |
| 저속 NBMA Hello/Hold | 60/180초 |
| Update 방식 | Partial·Bounded·Reliable |
| Transport | EIGRP RTP |

---

## 3. 검증 및 트러블슈팅

### 3-1. 공통 검증

```cisco
show ip eigrp neighbors
show ip protocols
show ip eigrp topology
show ip eigrp topology all-links
show ip route eigrp
show ip eigrp interfaces detail
show running-config | section router eigrp
```

Named Mode:

```cisco
show eigrp address-family ipv4 neighbors
show eigrp address-family ipv4 topology
```

---

### 3-2. Neighbor 미형성 원인

| 원인 | 확인 |
|---|---|
| AS 불일치 | `show ip protocols` |
| K값 불일치 | `show ip protocols` |
| Passive Interface | `show ip protocols` |
| 인증 불일치 | Interface/Key Chain |
| Subnet 불일치 | Interface IP/Mask |
| Protocol 88 차단 | ACL |
| Duplicate Router ID | Log/Topology |
| Secondary Address 문제 | Interface 주소 설계 |
| L2/NBMA Mapping 문제 | DLCI/Mapping/L2 |

> Hello/Hold Timer 값이 서로 다르다는 이유만으로 Neighbor가 반드시 실패하는 것은 아닙니다.

---

### 3-3. SIA 대응

#### Stub

```cisco
router eigrp 100
 eigrp stub connected summary
```

#### Summary

```cisco
interface FastEthernet0/0
 ip summary-address eigrp 100 13.13.0.0 255.255.0.0
```

#### Active 상태 확인

```cisco
show ip eigrp topology active
show ip eigrp topology all-links
show ip eigrp neighbors detail
```

> Active Timer 증가는 근본 해결책이 아닙니다. 먼저 Query 범위, Packet Loss, CPU, WAN 지연 및 Stub/Summary 설계를 수정합니다.

---

### 3-4. Debug

```cisco
debug eigrp packets hello
debug eigrp packets update
debug eigrp packets query
debug eigrp packets reply
debug eigrp packets ack
```

종료:

```cisco
undebug all
```

> 명령 문법은 IOS/IOS XE 버전에 따라 `debug eigrp packet` 또는 `debug eigrp packets` 형태가 다를 수 있으므로 `debug eigrp ?`로 확인합니다.

---

## 4. 대표 함정

| 함정 | 결과 | 권장 조치 |
|---|---|---|
| EIGRP를 Link-State로 분류 | 개념 혼동 | Advanced Distance Vector |
| 모든 대체 경로를 FS로 판단 | 잘못된 Failover 예상 | RD < Successor FD 확인 |
| FS가 없으면 바로 SIA | 잘못된 원인 분석 | Reply Timeout이 직접 원인 |
| Timer 불일치=Neighbor 실패 | 잘못된 진단 | AS/K/Auth/Subnet 우선 |
| Summary AD 5 일반화 | AD 해석 오류 | Local Null0 Summary로 한정 |
| Active Timer 증가 | SIA 원인 은폐 | Stub·Summary·WAN 개선 |
| Frame Relay 전체에서 SH 해제 | Loop 위험 | Hub Multipoint에서만 |
| MD5를 현대 보안으로 간주 | 인증 강도 부족 | HMAC/망 격리 검토 |
| `variance`만 설정 | 경로 미사용 | FC와 Metric 범위 확인 |

---

> 📌 **핵심 요약**
> - EIGRP = **Advanced Distance Vector + DUAL**
> - Successor = 현재 최적 경로
> - FS = FC를 만족하는 Loop-free Backup
> - FC = `RD < Successor FD`
> - FS 부재는 Query 시작 조건, Reply 미수신이 SIA 직접 원인
> - SIA 근본 대책 = Stub·주소 요약·계층화·WAN 품질 개선
> - AD = Internal 90 / External 170 / Local Summary Null0 5
> - 관련: 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol) · 2-3. 🚑 EIGRP 트러블슈팅 치트시트 · 2-4. ⚡ EIGRP 명령어 퀵 레퍼런스 · 1-1. ♟️ RIP (Routing Information Protocol) v1 & v2 · 3-1. 🌲 OSPF (Open Shortest Path First)
