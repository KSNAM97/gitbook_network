# 🚑 EIGRP 트러블슈팅 치트시트

> **Tag:** #EIGRP #Routing #Troubleshooting #DUAL #SIA #FeasibleSuccessor #CheatSheet
> **핵심 요약:** EIGRP 장애는 **Interface/L2 → EIGRP 활성 여부 → AS/K값 → Passive → 인증 → Neighbor → Topology → RIB** 순서로 확인한다. Neighbor 장애는 설정 정합성과 연결 상태를 확인하고, SIA는 Active Route의 Reply가 지연되는 원인을 찾아 Stub·Summary·계층화로 Query 범위를 줄인다.

---

## 1. 🎯 핵심 진단 순서

```properties
[1] Interface up/up
        ↓
[2] network 명령에 Interface 포함
        ↓
[3] AS·K값 확인
        ↓
[4] Passive Interface 확인
        ↓
[5] 인증·ACL·Subnet 확인
        ↓
[6] Neighbor Table
        ↓
[7] Topology Table
        ↓
[8] Routing Table
```

---

## 2. 🛠️ 증상별 즉시 대응표

### 2-1. Neighbor

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| Neighbor 없음 | AS 불일치 | 양단 AS 확인 |
| Neighbor 없음 | K값 불일치 | 기본 K값 확인 |
| Hello 미송신 | Passive Interface | 해당 Interface 해제 |
| 인증 후 Down | Key/Mode 불일치 | Key와 Lifetime 확인 |
| Uptime 반복 Reset | Link Loss·인증·SIA | Log/Interface 확인 |
| Hold 만료 | Hello Loss·혼잡 | Packet Loss/CPU 확인 |
| Protocol 88 차단 | ACL | EIGRP Permit |
| NBMA에서 미형성 | Mapping/L2 문제 | DLCI/Mapping 확인 |

### 2-2. 경로 및 수렴

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| Neighbor 정상, 경로 없음 | Network/Filter/Summary | Topology 확인 |
| Topology에는 있고 RIB에 없음 | AD·Metric·더 좋은 경로 | `show ip route` |
| Backup 전환 시 Query | FS 없음 | FC 확인 |
| SIA 발생 | Reply 미수신 | Stub/Summary/WAN 개선 |
| Unequal Load Balance 안 됨 | FC 또는 Variance 불충족 | RD/FD/Variance 확인 |
| Spoke 간 경로 없음 | Split Horizon | Hub Multipoint에서 해제 |

---

## 3. 🔍 핵심 진단 명령어

```cisco
show ip interface brief
show ip eigrp neighbors
show ip eigrp neighbors detail
show ip protocols
show ip eigrp interfaces detail
show ip eigrp topology
show ip eigrp topology all-links
show ip eigrp topology active
show ip route eigrp
show running-config | section router eigrp
```

---

## 4. 🚨 트러블슈팅 시나리오

### 시나리오 1. Neighbor 미형성

#### 1단계: Interface와 Subnet

```cisco
show ip interface brief
show interfaces Serial1/0
```

양단 확인:

- Interface up/up
- 동일 IP Subnet
- MTU 및 L2 상태
- Frame Relay/NBMA Mapping

---

#### 2단계: EIGRP Process

```cisco
show ip protocols
```

확인 항목:

- EIGRP AS
- Routing for Networks
- Passive Interface
- K값
- Router ID
- Filter/Distribute-list

---

#### 3단계: Passive Interface

```cisco
router eigrp 100
 no passive-interface Serial1/0
```

---

#### 4단계: 인증

```cisco
show key chain
show running-config interface Serial1/0
show logging | include EIGRP|DUAL
```

정상 예시:

```cisco
key chain EIGRP_AUTH
 key 20240101
  key-string <EIGRP_SECRET>
!
interface Serial1/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_AUTH
```

---

#### 5단계: ACL

EIGRP는 TCP/UDP가 아니라 IP Protocol 88을 사용합니다.

```cisco
ip access-list extended ACL_ROUTING
 permit eigrp host <NEIGHBOR_IP> any
```

검증:

```cisco
show ip access-lists
show ip interface Serial1/0
```

---

### 시나리오 2. Neighbor Uptime이 계속 초기화됨

#### 확인

```cisco
show ip eigrp neighbors
show logging | include DUAL|EIGRP
show interfaces Serial1/0
show processes cpu sorted
```

주요 원인:

- Link Flapping
- Packet Loss
- 인증 Key Lifetime 전환 오류
- Queue Congestion
- SIA
- Duplicate Address
- Router CPU 과부하
- Hold Timer 만료

> Hello/Hold 값이 다르다는 사실만으로 Neighbor가 실패한다고 판단하지 않습니다. Hello가 실제로 손실되는지 확인합니다.

---

### 시나리오 3. Neighbor는 정상인데 경로가 없음

```cisco
show ip eigrp topology all-links
show ip route <PREFIX>
show ip protocols
```

원인 후보:

- Outbound/Inbound Distribute-list
- Route-map 또는 Prefix-list
- Summary로 상세 Prefix 은닉
- Split Horizon
- 해당 Connected Prefix가 EIGRP에 포함되지 않음
- Neighbor가 해당 경로를 광고하지 않음

---

### 시나리오 4. Topology에는 경로가 있지만 RIB에 없음

```cisco
show ip eigrp topology <PREFIX>/<LENGTH>
show ip route <PREFIX>
```

가능한 원인:

- Connected Route AD 0
- Static Route AD 1
- 다른 EIGRP 경로의 더 낮은 Metric
- 다른 Protocol의 더 낮은 AD
- 경로가 Active 상태
- RIB Failure 또는 정책 제한

---

### 시나리오 5. FS가 없는 이유

```cisco
show ip eigrp topology all-links
```

Feasibility Condition:

```properties
Neighbor RD < 현재 Successor FD
```

예시:

```text
Successor FD = 100000
대체 경로 RD = 90000   → FS 가능
대체 경로 RD = 120000  → FS 불가
```

> FS가 없다는 것은 대체 물리 경로가 없다는 뜻이 아닙니다. 대체 경로가 있어도 Feasibility Condition을 만족하지 않을 수 있습니다.

---

### 시나리오 6. SIA 발생

로그 예시:

```text
%DUAL-3-SIA: Route 10.20.30.0/24 stuck-in-active
```

#### 확인

```cisco
show ip eigrp topology active
show ip eigrp neighbors detail
show interfaces
show processes cpu sorted
show logging | include SIA|DUAL
```

#### 근본 조치 1: Stub

```cisco
router eigrp 100
 eigrp stub connected summary
```

#### 근본 조치 2: Summary

```cisco
interface FastEthernet0/0
 ip summary-address eigrp 100 13.13.0.0 255.255.0.0
```

#### 근본 조치 3: 인프라 문제 해결

- WAN Packet Loss 제거
- CPU 과부하 해결
- 과도한 Query Domain 분리
- 불안정 Link 복구
- 재분배 Loop/경로 수 감소

> `timers active-time` 증가는 진단 또는 제한적인 임시 조치일 뿐, SIA의 근본 대책이 아닙니다.

---

### 시나리오 7. Frame Relay Spoke 간 통신 불가

Hub Multipoint에서 확인:

```cisco
show ip eigrp interfaces detail Serial1/0.123
```

필요 시:

```cisco
interface Serial1/0.123 multipoint
 no ip split-horizon eigrp 100
```

> Spoke 또는 불필요한 Interface 전체에 적용하지 않습니다. 가능하면 Point-to-Point Subinterface를 사용합니다.

---

### 시나리오 8. Unequal-cost Load Balancing이 동작하지 않음

```cisco
show ip eigrp topology all-links
show ip route <PREFIX>
show running-config | section router eigrp
```

설정:

```cisco
router eigrp 100
 maximum-paths 4
 variance 2
```

확인 조건:

1. 대체 경로가 Feasibility Condition 만족
2. 대체 Metric이 Successor FD × Variance 이하
3. `maximum-paths`가 2 이상
4. Traffic Sharing 설정 확인

---

## 5. 🐞 Debug

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

> 플랫폼에 따라 Debug 명령 문법이 다를 수 있으므로 `debug eigrp ?`로 확인합니다.

---

## 6. 🔧 현장 Quick Check

```cisco
show ip interface brief
show ip eigrp neighbors
show ip protocols
show ip eigrp interfaces detail
show ip eigrp topology all-links
show ip eigrp topology active
show ip route eigrp
show logging | include EIGRP|DUAL|SIA
```

---

> 📌 **핵심 요약**
> - Neighbor 실패: Interface·AS·K값·Passive·인증·Subnet·ACL
> - Timer 값은 반드시 일치해야 하는 조건이 아님
> - FS = `RD < Successor FD`
> - FS 부재는 Query 조건, SIA는 Reply Timeout
> - SIA 근본책 = Stub·Summary·계층화·WAN 품질 개선
> - FR Split Horizon 해제는 Hub Multipoint에서만
> - 관련: 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol) · 2-2. 🐎 EIGRP 통합 정리 · 2-4. ⚡ EIGRP 명령어 퀵 레퍼런스 · 1-1. ♟️ RIP (Routing Information Protocol) v1 & v2 · 3-1. 🌲 OSPF (Open Shortest Path First)
