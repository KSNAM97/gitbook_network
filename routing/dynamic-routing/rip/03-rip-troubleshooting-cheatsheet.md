# 🚑 RIP 트러블슈팅 치트시트

> **Tag:** #Routing #RIP #Troubleshooting #Convergence #Authentication #CheatSheet
> **핵심 요약:** RIP 장애는 **Interface → RIP 활성 Network → Send/Receive Version → Passive Interface → Authentication → Update 수신 → RIB 설치 → Return Path** 순서로 확인한다. 학습 장애는 설정과 Update 수신을 보고, 수렴 장애는 Metric 16·Triggered Update·Timer·Split Horizon을 확인한다.

---

## 1. 🎯 핵심 진단 순서

```properties
[1] Interface가 up/up인가?
        ↓
[2] Interface 주소가 RIP network 범위에 포함되는가?
        ↓
[3] 양단 Send/Receive Version이 일치하는가?
        ↓
[4] Routing Link가 Passive인가?
        ↓
[5] RIPv2 인증이 일치하는가?
        ↓
[6] Update가 실제 송수신되는가?
        ↓
[7] 경로가 RIP Database와 RIB에 설치되는가?
        ↓
[8] Return Path가 존재하는가?
```

---

## 2. 🛠️ 증상별 즉시 대응표

### 2-1. 경로 학습

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| RIP 경로 없음 | Network 범위 오류 | `show ip protocols` | Major Network 수정 |
| Update 미수신 | Version 불일치 | Debug/Protocols | 양단 Version 2 |
| Update 미송신 | Passive Interface | Protocols | Passive 해제 |
| Update 폐기 | 인증 불일치 | Debug/Key Chain | Key 설정 일치 |
| 경로 수신 후 RIB 미설치 | 더 낮은 AD 경로 | `show ip route` | Route Source 비교 |
| 특정 Subnet 누락 | Auto Summary | Protocols/Debug | `no auto-summary` |
| RIB는 정상, Ping 실패 | Return Path/ACL | Route/Ping | 역방향 확인 |

---

### 2-2. 수렴 및 Loop

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 경로가 늦게 제거 | Update 손실·Invalid/Flush Timer | Debug·Timer 확인 |
| Metric이 계속 증가 | Routing Loop | Split Horizon·Topology 확인 |
| Metric 16 수신 | Route Poison | 장애 원인 확인 |
| Spoke 간 경로 없음 | Hub Multipoint Split Horizon | Hub에서만 해제 검토 |
| Route Flapping | Timer 불일치·불안정 Link | 기본 Timer/Link 복구 |
| Triggered WAN 동작 이상 | 양단 설정 불일치 | 양단 `ip rip triggered` 확인 |

---

## 3. 🔍 핵심 진단 명령어

```cisco
show ip interface brief
show ip protocols
show ip route rip
show ip rip database
show running-config | section router rip
show ip interface Serial1/0
show key chain
debug ip rip
```

Debug 종료:

```cisco
undebug all
```

---

## 4. 🚨 트러블슈팅 시나리오

### 시나리오 1. 이웃 경로가 학습되지 않음

#### 1단계: Interface 확인

```cisco
show ip interface brief
```

정상 기준:

```properties
Status = up
Protocol = up
IP Address = 양단 동일 Subnet
```

---

#### 2단계: RIP Process 확인

```cisco
show ip protocols
```

확인 항목:

- Routing Protocol is RIP
- Sending/Receiving Version
- Automatic Summarization
- Routing for Networks
- Passive Interfaces
- Routing Information Sources

---

#### 3단계: Version 통일

```cisco
router rip
 version 2
```

> 명시적인 Version 정책이 다르면 Update를 수신하지 못하거나 예상과 다른 Classful 경로가 생성될 수 있습니다.

---

#### 4단계: Network 범위 확인

잘못된 예:

```cisco
router rip
 network 172.16.1.0
```

권장:

```cisco
router rip
 network 172.16.0.0
```

> Classic IOS RIP의 `network` 명령은 Classful Major Network 기준입니다.

---

#### 5단계: Passive Interface 확인

```cisco
show ip protocols
```

Routing Link가 Passive이면 해제합니다.

```cisco
router rip
 no passive-interface Serial1/0
```

> Passive Interface에 연결된 Network가 다른 Interface를 통해 광고되는 것은 가능합니다. Passive는 해당 Interface의 RIP Update 송신을 억제합니다.

---

#### 6단계: Update 확인

```cisco
debug ip rip
```

정상 수신 예시:

```text
RIP: received v2 update from 13.13.12.2 on Serial1/0
```

정상 송신 예시:

```text
RIP: sending v2 update to 224.0.0.9 via Serial1/0
```

---

### 시나리오 2. 인증 설정 후 경로가 사라짐

#### 확인

```cisco
show key chain
show running-config interface Serial1/0
debug ip rip
```

예상 오류:

```text
ignored v2 packet from 13.13.12.2 (invalid authentication)
```

#### 정상 설정

```cisco
key chain RIP_AUTH
 key 20110117
  key-string <RIP_SECRET>
!
interface Serial1/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP_AUTH
```

양단 확인:

- Authentication Mode
- 활성 Key ID
- Key-string
- Key Lifetime
- Router 시간/NTP

---

### 시나리오 3. RIP Update를 받지만 RIB에 경로가 없음

#### 확인

```cisco
show ip rip database
show ip route <PREFIX>
show ip protocols
```

가능한 원인:

1. Connected Route가 존재
2. Static Route AD 1이 존재
3. EIGRP AD 90이 존재
4. OSPF AD 110이 존재
5. RIP Metric이 16
6. 동일 Prefix에 더 좋은 RIP Metric이 존재

예시:

```text
S 10.10.10.0/24 [1/0] via 192.0.2.1
R 10.10.10.0/24 [120/2] via 198.51.100.1
```

Static Route가 같은 Prefix에 대해 우선하므로 RIP Route는 RIB에 설치되지 않습니다.

---

### 시나리오 4. Auto Summary로 경로가 잘못 요약됨

#### 증상

- 불연속 Network에서 동일 Major Network가 여러 위치에 존재
- 특정 Subnet으로 Blackhole 발생
- Debug에서 Classful Summary만 확인

#### 확인

```cisco
show ip protocols
debug ip rip
```

#### 조치

```cisco
router rip
 no auto-summary
```

검증:

```cisco
show ip route rip
debug ip rip
```

---

### 시나리오 5. 장애 경로가 늦게 사라짐

#### 확인

```cisco
debug ip rip
show ip rip database
show ip route rip
```

Metric 16이 보이면 Route Poison이 전파되고 있는 것입니다.

```text
172.16.3.0/24 via 13.13.12.2, metric 16
```

> 직접 Interface Down이 감지되면 Triggered Update로 빠르게 전파될 수 있습니다. Update 자체가 손실되거나 Neighbor와의 연결이 조용히 끊긴 경우 Invalid/Flush Timer에 의존할 수 있습니다.

Timer 확인:

```cisco
show ip protocols
```

기본 복구:

```cisco
router rip
 timers basic 30 180 180 240
```

---

### 시나리오 6. Frame Relay Spoke 간 경로가 학습되지 않음

#### 토폴로지

```text
Spoke1 ── Hub Multipoint ── Spoke2
```

#### 원인

Hub가 Spoke1에서 배운 경로를 같은 Multipoint Interface로 Spoke2에 다시 광고하지 않는 Split Horizon 문제일 수 있습니다.

#### 확인

```cisco
show ip interface Serial1/0 | include split
```

필요한 경우 Hub Multipoint Interface에서만 검토:

```cisco
interface Serial1/0
 no ip split-horizon
```

> Split Horizon 해제는 Loop 위험을 증가시킬 수 있습니다. 가능하면 Point-to-Point Subinterface로 분리하는 설계가 더 안전합니다.

---

### 시나리오 7. `ip rip triggered` 설정 후 Update가 예상과 다름

#### 원인

`ip rip triggered`는 일반 Triggered Update 활성화 명령이 아니라 Periodic Update를 억제하는 Triggered Extension입니다.

#### 확인

```cisco
show ip protocols
show running-config interface Serial1/0
debug ip rip
```

양쪽 Interface에 동일하게 적용됐는지 확인합니다.

```cisco
interface Serial1/0
 ip rip triggered
```

일반 LAN 또는 지속 연결 WAN에서 필요하지 않다면 제거합니다.

```cisco
interface Serial1/0
 no ip rip triggered
```

---

### 시나리오 8. Unicast RIP Neighbor가 동작하지 않음

#### 설정

```cisco
router rip
 passive-interface Serial1/0
 neighbor 13.13.12.2
```

#### 확인

```cisco
ping 13.13.12.2
show ip route 13.13.12.2
debug ip rip
```

확인 항목:

- Neighbor IP Reachability
- 양방향 Neighbor 설정
- RIPv2 Version
- 인증 설정
- Interface ACL에서 UDP 520 허용 여부

---

## 5. 🔧 현장 Quick Check

```cisco
show ip interface brief
show ip protocols
show ip route rip
show ip rip database
show key chain
debug ip rip
```

추가 확인:

```cisco
show access-lists
show ip route <PREFIX>
ping <DESTINATION> source <SOURCE>
traceroute <DESTINATION> source <SOURCE>
```

---

> 📌 **핵심 요약**
> - 학습 장애: Interface → Network → Version → Passive → Authentication
> - Update는 받지만 RIB에 없으면 AD·Metric·Connected/Static 확인
> - Auto Summary는 불연속 Network에서 Blackhole 원인
> - Metric 16은 Route Poison 및 Unreachable
> - Flush 240초는 마지막 정상 Update부터 계산
> - `ip rip triggered`는 저속 WAN용 Triggered Extension
> - 관련: 1-1. ♟️ RIP (Routing Information Protocol) v1 & v2 · 1-2. ♟️ RIP 통합 정리 · 1-4. ⚡ RIP 명령어 퀵 레퍼런스 · 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol) · 3-1. 🌲 OSPF (Open Shortest Path First)
