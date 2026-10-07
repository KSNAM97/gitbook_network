# RIP 통합 정리 — Distance Vector·v1 vs v2·Loop 방지 한눈에

> **Tag:** #Routing #RIP #DistanceVector #IGP #Convergence #Summary
> **핵심 요약:** RIP는 **Hop Count(최대 15)** 를 Metric으로 사용하는 Distance Vector IGP로, Cisco IOS 기본 AD는 120이며 UDP 520을 사용한다. 30초 주기 Routing Update와 최대 Hop 제한 때문에 대규모 환경에는 부적합하지만, Split Horizon·Route Poison·Poison Reverse·Triggered Update 등 Distance Vector의 핵심 루프 방지 원리를 이해하는 데 중요하다. 현대적인 RIP 환경에서는 RIPv2와 `no auto-summary`를 사용하고, 실제 운영에서는 OSPF·EIGRP·BGP로의 마이그레이션을 우선 검토한다.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. RIP는 왜 거의 사용하지 않는데 아직 배우나요?

👉 **핵심 답변:** RIP는 **최대 15 Hop이라는 확장성 한계**와 주기적 Routing Update에 의존하는 느린 수렴 때문에 현대 백본에서는 거의 사용하지 않지만, Distance Vector의 기본 동작과 라우팅 루프 방지 원리를 이해하는 데 가장 적합한 프로토콜입니다.

- **꼬리 질문 대비 설명:**
  - 소규모 지점망, 레거시 CPE, 오래된 산업 장비 등에서 여전히 발견될 수 있음
  - Split Horizon, Route Poison, Poison Reverse, Count-to-Infinity의 원형을 학습할 수 있음
  - RIP의 AD 120은 동일 Prefix를 OSPF·EIGRP 등과 동시에 학습할 때 Route Preference 판단에 사용됨
  - 신규 대규모 네트워크에는 RIP보다 OSPF, IS-IS, EIGRP 또는 BGP가 적합

---

### Q2. RIPv1과 RIPv2의 결정적 차이는?

👉 **핵심 답변:** RIPv1은 Update에 Subnet Mask를 포함하지 않는 Classful 프로토콜이고, RIPv2는 Subnet Mask와 Route Tag, Next-hop 정보를 포함하는 Classless 프로토콜입니다.

| 구분 | RIPv1 | RIPv2 |
|---|---|---|
| 주소 체계 | Classful | Classless |
| Subnet Mask 전송 | 미지원 | 지원 |
| VLSM/CIDR | 부적합 | 지원 |
| 기본 전송 | Broadcast | Multicast |
| 목적지 | 255.255.255.255 | 224.0.0.9 |
| 인증 | 미지원 | Plain Text·MD5 |
| Route Tag | 미지원 | 지원 |

> Cisco IOS에서는 RIPv2 Automatic Summary가 기본 활성인 버전이 있으므로 VLSM과 불연속 네트워크에서는 `no auto-summary`를 명시하는 것이 안전합니다.

---

### Q3. RIP의 수렴이 느린 이유는?

👉 **핵심 답변:** RIP는 전체 토폴로지를 알지 못하고 이웃이 알려준 Distance와 Vector를 신뢰하며, 경로 손실 감지에도 Timer를 사용하기 때문입니다.

```properties
정상 Update 수신
    ↓
180초 동안 Update 없음
    ↓
Invalid 상태 및 Metric 16 처리
    ↓
마지막 정상 Update부터 240초 후
    ↓
Route Flush
```

- Update Timer: 기본 30초
- Invalid Timer: 기본 180초
- Holddown Timer: 기본 180초
- Flush Timer: 기본 240초

> Flush 240초는 Invalid 180초 이후 추가로 240초를 기다린다는 뜻이 아닙니다. 일반적으로 마지막 정상 Update 수신 시점부터 240초가 지나면 경로를 제거합니다.

직접 연결 Interface가 Down되거나 Metric 변화가 감지되면 주기 Update를 기다리지 않고 Triggered Update를 보낼 수 있으므로 모든 장애가 항상 180~240초 걸리는 것은 아닙니다.

---

### Q4. 일반 Triggered Update와 `ip rip triggered`는 무엇이 다른가?

#### 일반 RIP Triggered Update

RIP는 경로 Metric이 변경되면 30초 주기 Update를 기다리지 않고 변경된 정보를 이웃에게 빠르게 전달할 수 있습니다.

```properties
경로 상태 변화
    ↓
Triggered Update 생성
    ↓
변경 경로를 빠르게 광고
```

#### Triggered Extension

```cisco
interface Serial1/0
 ip rip triggered
```

`ip rip triggered`는 일반 Triggered Update를 켜는 명령이 아니라, 저속·비용 기반 WAN에서 Periodic Update를 억제하고 변화가 있을 때 Update를 전송하는 **Triggered Extensions to RIP** 기능입니다.

- On-demand Circuit이나 특정 Point-to-Point WAN에 적합
- 양쪽 Router가 Triggered RIP를 지원하고 동일하게 구성해야 함
- 일반 LAN이나 모든 P2P Link에 무조건 적용하는 성능 튜닝 명령이 아님

---

### Q5. RIP의 루프 방지 기술은?

| 기술 | 동작 | 목적 |
|---|---|---|
| Split Horizon | 배운 Interface로 같은 경로를 재광고하지 않음 | 단순 2노드 Loop 방지 |
| Route Poison | 장애 경로를 Metric 16으로 광고 | 경로 무효 상태 전파 |
| Poison Reverse | 배운 방향으로 Metric 16을 명시적으로 광고 | 잘못된 역방향 경로 방지 |
| Triggered Update | Metric 변화 시 빠르게 Update | 수렴 지연 감소 |
| Holddown | 불안정한 대체 정보의 즉시 수용 제한 | Loop/Flapping 완화 |
| Maximum 16 | Count-to-Infinity 상한 설정 | 무한 Metric 증가 제한 |

> 실제 Poison Reverse의 동작과 출력은 구현 및 Interface 유형에 따라 달라질 수 있습니다.

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. RIPv2 기본 설정

```cisco
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 network 172.16.0.0
 passive-interface Loopback172
 passive-interface FastEthernet0/0
```

#### `network` 명령의 역할

```properties
1. 해당 Major Network에 포함되는 Interface에서 RIP 활성화
2. 해당 Connected Network를 RIP Update 대상으로 포함
```

Classic Cisco IOS RIP의 `network` 명령은 일반적으로 Classful Major Network 형태로 입력합니다.

```cisco
network 172.16.0.0
```

> `network` 명령이 Prefix-list처럼 정확한 광고 Prefix를 직접 지정하는 것은 아닙니다. 실제 광고 경로는 활성 Interface, Connected Route, Redistribution, Summary 정책에 따라 결정됩니다.

---

### 2-2. Passive Interface

```cisco
router rip
 passive-interface FastEthernet0/0
```

Passive Interface는 해당 Interface로 RIP Update를 송신하지 않게 합니다.

```properties
Passive Interface
= RIP Update 송신 억제
≠ Connected Network 광고 자체 제거
```

해당 Connected Network는 다른 RIP Interface를 통해 계속 광고될 수 있습니다.

다수 Interface가 있는 환경의 권장 패턴:

```cisco
router rip
 passive-interface default
 no passive-interface Serial1/0
 no passive-interface Serial1/1
```

---

### 2-3. RIPv2 MD5 인증

```cisco
key chain RIP_AUTH
 key 20110117
  key-string <RIP_SECRET>
!
interface Serial1/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP_AUTH
```

양쪽 Router에서 다음 항목을 확인합니다.

- Key-chain 이름은 로컬 참조이므로 달라도 구성 가능하지만 관리상 통일 권장
- 활성 Key ID
- Key-string
- Send/Accept Lifetime
- Authentication Mode

> MD5는 Plain Text보다 안전하지만 현대 암호 기준에서는 레거시입니다. RIP 자체를 신뢰할 수 있는 내부 구간에 한정하고, 장기적으로 더 현대적인 Routing Protocol이나 별도 보호 구조를 검토합니다.

---

### 2-4. Unicast RIP Update

```cisco
router rip
 passive-interface Serial1/0
 neighbor 13.13.12.2
```

- Broadcast/Multicast Update를 억제하고 지정 Neighbor로 Unicast Update
- NBMA 또는 특수 Topology에서 사용
- 양방향 Neighbor와 Reachability를 확인해야 함

---

### 2-5. Triggered RIP Extension

```cisco
interface Serial1/0
 ip rip triggered
```

적용 조건:

- Point-to-Point 또는 On-demand WAN
- 양단 Triggered RIP 지원
- Periodic Update 비용을 줄여야 하는 환경
- 사전 Lab 검증 완료

---

### 2-6. Timer 조정

```cisco
router rip
 timers basic 30 180 180 240
```

문법:

```cisco
timers basic <UPDATE> <INVALID> <HOLDDOWN> <FLUSH>
```

> Timer 변경은 같은 RIP Domain에서 일관되게 설계해야 합니다. 단순히 값을 줄이면 Update 손실, Route Flapping, CPU 증가가 발생할 수 있습니다.

---

### 2-7. 주요 파라미터

```properties
Protocol Type = Distance Vector IGP
Algorithm     = Bellman-Ford 계열
Metric        = Hop Count
Maximum       = 15 Hop
Unreachable   = Metric 16
Cisco AD      = 120
Transport     = UDP 520
RIPv1         = Broadcast / Classful
RIPv2         = Multicast 224.0.0.9 / Classless / Authentication
```

---

## 3. 검증 및 트러블슈팅

### 3-1. 기본 검증

```cisco
show ip route rip
show ip protocols
show ip rip database
show running-config | section router rip
show ip interface Serial1/0
show key chain
```

### 3-2. Packet 확인

```cisco
debug ip rip
```

종료:

```cisco
undebug all
```

> 운영망에서는 Debug 출력량과 CPU 부하에 주의합니다.

---

### 3-3. End-to-End 검증

```cisco
ping 172.16.3.3 source 13.13.10.254
traceroute 172.16.3.3 source 13.13.10.254
show ip route 172.16.3.3
```

---

### 3-4. 정상 Debug 예시

```text
RIP: sending v2 update to 224.0.0.9 via Serial1/0 (13.13.12.1)
      13.13.10.0/24 via 0.0.0.0, metric 1, tag 0
      172.16.1.0/24 via 0.0.0.0, metric 1, tag 0
```

확인 항목:

- RIPv2 전송
- 목적지 224.0.0.9
- Subnet Mask 포함
- Metric이 1~15
- 의도하지 않은 Classful Summary가 없는지 확인

---

## 4. 대표 함정

| 함정 | 결과 | 권장 조치 |
|---|---|---|
| v1/v2 정책 불일치 | Update 수신 거부·Classful 문제 | 양단 `version 2` |
| Auto Summary | 불연속 Network Blackhole | `no auto-summary` |
| 잘못된 Major Network | Interface에서 RIP 미활성 | Classful Network 확인 |
| Routing Link가 Passive | Update 미송신 | `no passive-interface` |
| 인증 불일치 | Update 폐기 | Key ID/String/Lifetime 확인 |
| Hop Count 16 | Unreachable | Topology·Loop 확인 |
| Timer 과도한 축소 | Flapping·Update 손실 | 기본값 복구 |
| FR Hub Split Horizon | Spoke 간 광고 제한 | Hub Multipoint에서만 검토 |
| `ip rip triggered` 오용 | Periodic Update 동작 변화 | WAN 조건 확인 |

---

> 📌 **핵심 요약**
> - RIP = Distance Vector, AD 120, UDP 520
> - Metric = Hop Count, 15까지 도달 가능, 16은 Unreachable
> - RIPv2 + `no auto-summary` 권장
> - 루프 방지 = Split Horizon·Poison·Triggered Update·Holddown
> - Passive Interface는 송신 억제이며 Network 광고 제거가 아님
> - `ip rip triggered`는 저속 WAN용 Triggered Extension
> - 관련: 1-1. ♟️ RIP (Routing Information Protocol) v1 & v2 · 1-3. 🚑 RIP 트러블슈팅 치트시트 · 1-4. ⚡ RIP 명령어 퀵 레퍼런스 · 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol) · 3-1. 🌲 OSPF (Open Shortest Path First)
