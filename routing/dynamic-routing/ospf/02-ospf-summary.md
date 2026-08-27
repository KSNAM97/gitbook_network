# OSPF 통합 정리 — SPF·Area·LSA·Neighbor State

> **Tag:** #Routing #OSPF #LinkState #IGP #Area #LSA #Summary  
> **핵심 요약:** OSPF는 **Dijkstra SPF 알고리즘을 사용하는 개방형 Link-State IGP**다. IP Protocol 89를 사용하며 기본 AD는 110이다. Area 계층화를 통해 Type 1·2 LSA의 범위를 제한하고, ABR 요약과 Stub/NSSA를 이용해 LSDB와 경로 정보를 제어한다. 장애 진단은 **Neighbor State가 어디에서 멈췄는지** 확인하는 것부터 시작한다.

---

## 1. 핵심 기술 개념

### Q1. OSPF를 엔터프라이즈 코어에서 사용하는 이유는?

👉 **핵심 답변:** OSPF는 RFC 기반 개방형 표준이고, Area 계층 설계를 통해 멀티벤더 환경에서 확장성과 예측 가능한 운영을 제공하기 때문이다.

- Cisco, Juniper, Arista, Huawei 등 다양한 장비에서 지원
- Area 0을 중심으로 계층적 토폴로지 구성
- Type 1·2 LSA는 해당 Area 내부에만 유지
- ABR이 Area 간 경로를 Type 3 LSA로 전달
- Stub/NSSA와 경로 요약을 이용해 LSDB 및 경로 수를 제어
- 동일 Area 내부 링크 변경은 해당 Area에서 SPF 계산을 유발하지만, 상세 Type 1·2 LSA가 다른 Area로 그대로 전파되지는 않음

> [!NOTE]
> Area 설계가 모든 변화를 완전히 차단하는 것은 아니다. 요약 경로의 도달 가능성이나 비용이 변경되면 ABR이 생성하는 Type 3 LSA도 변경될 수 있다.

---

### Q2. OSPF의 경로 계산 과정은?

```text
Hello 교환
  ↓
Neighbor/Adjacency 형성
  ↓
LSA 교환 및 LSDB 동기화
  ↓
Dijkstra SPF 계산
  ↓
최적 경로를 RIB에 설치
  ↓
CEF/FIB에 반영
```

OSPF 라우터는 같은 Area 안에서 동일한 LSDB를 구성한 후, 자신을 SPF Tree의 Root로 두고 최단 경로를 계산한다.

### Cost 공식

```text
Cost = Reference Bandwidth / Interface Bandwidth
```

Cisco IOS의 기본 Reference Bandwidth는 100Mbps이므로, 고속 링크가 혼재된 환경에서는 도메인 전체에 동일한 값을 설정한다.

```cisco
router ospf 100
 auto-cost reference-bandwidth 10000
```

위 설정은 10Gbps를 기준 대역폭으로 사용한다.

---

### Q3. OSPF Neighbor State가 중요한 이유는?

Neighbor State가 멈춘 위치에 따라 장애 범위를 빠르게 좁힐 수 있다.

```text
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

| State | 의미 | 정체 시 주요 확인 사항 |
|---|---|---|
| Down | Hello 미수신 | 인터페이스, ACL, Passive, Network Type |
| Init | 상대 Hello에 자신의 RID가 없음 | 단방향 통신, 멀티캐스트 문제 |
| 2-Way | 양방향 Hello 확인 | DROTHER 간이면 정상, DR/BDR 관계 확인 |
| ExStart | DBD Master/Slave 결정 | MTU, 중복 Router-ID |
| Exchange | DBD 교환 | MTU, DBD Sequence 문제 |
| Loading | 필요한 LSA 요청 | LSR/LSU 손실, 불안정한 링크 |
| Full | LSDB 동기화 완료 | 정상 인접성 |

> [!IMPORTANT]
> Broadcast/NBMA 네트워크에서 **DROTHER끼리 2-Way인 것은 정상**이다. 모든 Neighbor가 반드시 Full이어야 하는 것은 아니다.

---

### Q4. NBMA 환경에서 인접성이 실패하는 이유는?

Frame Relay 같은 NBMA 환경은 브로드캐스트와 멀티캐스트 전달이 제한될 수 있으며, 기본 Non-Broadcast Network Type에서는 Neighbor를 자동 탐색하지 않는다.

주요 해결 방법:

1. **Non-Broadcast 유지**
   - Hub를 DR로 지정
   - Spoke의 Priority를 0으로 설정
   - `neighbor` 명령으로 유니캐스트 Hello 전송

2. **Point-to-Multipoint 사용**
   - DR/BDR 선출 없음
   - Hub-and-Spoke 환경을 논리적인 P2P 링크 집합으로 처리

3. **Broadcast Network Type 사용**
   - 하부망이 멀티캐스트/브로드캐스트 복제를 지원해야 함
   - Frame Relay에서는 `broadcast` 매핑 필요

> [!CAUTION]
> DMVPN은 Phase, NHRP Multicast Mapping 및 설계에 따라 멀티캐스트 전달 방식이 달라진다. 단순히 “DMVPN은 멀티캐스트를 전달하지 않는다”고 일반화하면 안 된다.

---

## 2. OSPF 구성 요소

### 2-1. Router 역할

| 역할 | 설명 |
|---|---|
| Internal Router | 모든 인터페이스가 같은 Area에 속함 |
| Backbone Router | 하나 이상의 인터페이스가 Area 0에 속함 |
| ABR | Area 0과 다른 Area를 연결 |
| ASBR | 다른 프로토콜 또는 외부 경로를 OSPF로 재분배 |
| DR/BDR | Multi-access Network에서 LSA 교환을 대표 |

### 2-2. LSA 유형

| Type | 이름 | 생성 주체 | 범위 |
|---|---|---|---|
| 1 | Router LSA | 모든 OSPF Router | Area 내부 |
| 2 | Network LSA | DR | Area 내부 |
| 3 | Summary LSA | ABR | Area 간 |
| 4 | ASBR Summary LSA | ABR | ASBR 도달성 |
| 5 | AS External LSA | ASBR | Stub/NSSA 제외 도메인 |
| 7 | NSSA External LSA | NSSA ASBR | NSSA 내부, ABR에서 Type 5 변환 |

### 2-3. Area 유형

| Area 유형 | 주요 동작 |
|---|---|
| Normal | Type 1·2·3·4·5 허용 |
| Stub | Type 5 외부 LSA 차단, ABR이 기본 경로 제공 |
| Totally Stub | 대부분의 Type 3과 Type 4·5 차단, 기본 경로 사용 |
| NSSA | Type 5 차단, 내부 ASBR이 Type 7 생성 가능 |
| Totally NSSA | 대부분의 Type 3과 Type 4·5 차단, Type 7 허용 |

> [!NOTE]
> Totally Stub와 Totally NSSA는 Cisco 계열 확장 기능이다. 멀티벤더 환경에서는 지원 여부를 확인한다.

---

## 3. 표준 설정 템플릿

### 3-1. 기본 OSPF 구성

```cisco
router ospf 100
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Serial1/0
 network 13.13.12.1 0.0.0.0 area 0
 network 172.16.1.1 0.0.0.0 area 0
 auto-cost reference-bandwidth 10000
```

- `network` 명령은 해당 주소와 일치하는 **인터페이스에서 OSPF를 활성화하고 Area를 지정**한다.
- 서버/LAN 인터페이스가 Passive 상태여도 해당 인터페이스가 OSPF에 포함되면 연결 Prefix는 광고된다.
- 인접성이 필요한 인터페이스만 `no passive-interface`로 해제한다.

### 3-2. 인터페이스 단위 구성

```cisco
interface Serial1/0
 ip ospf 100 area 0
 ip ospf cost 10
```

### 3-3. Loopback Prefix 광고

```cisco
interface Loopback172
 ip address 172.16.1.1 255.255.255.0
 ip ospf 100 area 0
 ip ospf network point-to-point
```

OSPF는 Loopback을 기본적으로 /32 Host Route로 광고한다. 원래 Prefix Length로 광고해야 한다면 `point-to-point` Network Type을 사용할 수 있다.

### 3-4. 인증

```cisco
router ospf 100
 area 0 authentication message-digest
!
interface Serial1/0
 ip ospf message-digest-key 100 md5 <KEY>
```

또는 인터페이스 단위로 설정할 수 있다.

```cisco
interface Serial1/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 100 md5 <KEY>
```

> [!CAUTION]
> OSPFv2 MD5 인증은 호환성을 위해 많이 사용되는 레거시 방식이다. 플랫폼이 지원한다면 Key Chain 기반의 HMAC-SHA 인증 또는 OSPFv3/IPsec 등 더 강한 방식을 검토한다.

### 3-5. ABR 및 ASBR 요약

```cisco
router ospf 100
 area 13 range 100.100.0.0 255.255.248.0
 summary-address 200.200.8.0 255.255.248.0
```

| 명령어 | 적용 위치 | 대상 |
|---|---|---|
| `area X range` | ABR | Inter-Area Type 3 요약 |
| `summary-address` | ASBR | External Type 5/7 요약 |

### 3-6. 외부 경로 재분배

```cisco
router ospf 100
 redistribute eigrp 100 metric-type 1 subnets
```

- `subnets`가 없으면 일부 IOS 환경에서 Classful Major Network만 재분배될 수 있음
- E1은 외부 Cost와 ASBR까지의 내부 Cost를 함께 계산
- E2는 기본적으로 외부 Metric을 중심으로 비교

### 3-7. NBMA Hub-and-Spoke

#### Non-Broadcast 방식

```cisco
! Hub
interface Serial1/0.123 multipoint
 ip address 13.13.9.2 255.255.255.0
 ip ospf network non-broadcast
 ip ospf priority 255
!
router ospf 1
 neighbor 13.13.9.1
 neighbor 13.13.9.3
```

```cisco
! Spoke
interface Serial1/0.123 multipoint
 ip ospf network non-broadcast
 ip ospf priority 0
```

#### Point-to-Multipoint 방식

```cisco
interface Serial1/0.123 multipoint
 ip ospf network point-to-multipoint
```

멀티캐스트 전달이 불가능한 환경이라면 다음 방식을 검토한다.

```cisco
interface Serial1/0.123 multipoint
 ip ospf network point-to-multipoint non-broadcast
!
router ospf 1
 neighbor 13.13.9.1
```

### 3-8. Stub / NSSA

```cisco
! 모든 Area 14 라우터
router ospf 100
 area 14 stub
```

```cisco
! ABR
router ospf 100
 area 14 stub no-summary
```

```cisco
! 모든 Area 14 라우터
router ospf 100
 area 14 nssa
```

```cisco
! NSSA ABR
router ospf 100
 area 14 nssa no-summary
```

NSSA에서 기본 경로가 필요하면 설계에 따라 다음 명령을 사용한다.

```cisco
router ospf 100
 area 14 nssa default-information-originate
```

### 3-9. Virtual Link

```cisco
! R2
router ospf 100
 area 13 virtual-link 3.3.3.3
```

```cisco
! R3
router ospf 100
 area 13 virtual-link 2.2.2.2
```

- 명령에 사용하는 값은 상대편 ABR의 Router-ID
- Transit Area는 Stub Area일 수 없음
- Virtual Link는 임시 복구 또는 마이그레이션 수단으로 사용
- 장기적으로 Area 0 연결 구조를 재설계하는 것이 권장됨

---

## 4. 검증 명령어

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip ospf interface <interface>
show ip ospf database
show ip ospf database router
show ip ospf database network
show ip ospf database summary
show ip ospf database external
show ip ospf border-routers
show ip route ospf
show ip protocols
```

운영 중 Debug는 주의해서 사용한다.

```cisco
debug ip ospf adj
debug ip ospf hello
debug ip ospf packet
undebug all
```

---

## 5. 핵심 정리

- OSPF는 **AD 110, IP Protocol 89**를 사용하는 Link-State IGP
- Area 0은 OSPF Backbone
- Type 1·2 LSA는 Area 내부에 유지
- DROTHER 간 2-Way는 정상
- ExStart/Exchange 반복은 MTU와 Router-ID 중복 확인
- Area 불일치는 Neighbor 형성 실패 원인
- Stub은 외부 Type 5가 필요 없는 Area에 사용
- 외부 경로를 재분배해야 하는 Stub 성격의 Area는 NSSA 사용
- Reference Bandwidth는 도메인 전체에서 동일하게 설정

> - 관련: 3-1. 🌲 OSPF (Open Shortest Path First) · 3-3. 🚑 OSPF 트러블슈팅 치트시트 · 3-4. ⚡ OSPF 명령어 퀵 레퍼런스
> - 비교: 1-1. ♟️ RIP (Routing Information Protocol) v1 & v2 · 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol)
