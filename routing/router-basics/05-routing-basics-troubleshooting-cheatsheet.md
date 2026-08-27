# 🚑 라우팅 기초 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #Routing #Static #RIB #FIB #CEF #IPSLA #CheatSheet
> **핵심 요약:** Static Routing 장애는 **Interface → Connected Route → 목적지 RIB → CEF/Adjacency → Next-hop → Return Path → ACL/NAT** 순서로 확인한다. RIB에 Static Route가 보인다고 실제 통신이 보장되는 것은 아니며, Ethernet에서는 Next-hop ARP와 CEF Adjacency까지 확인해야 한다.

---

## 1. 🎯 핵심 진단 흐름

### Q1. 통신이 안 될 때 어떤 순서로 확인하는가?

👉 **핵심 답변:** 아래 순서를 고정해서 확인합니다.

```properties
[1] Interface up/up
        ↓
[2] Connected Route 존재
        ↓
[3] 목적지에 대한 RIB 경로
        ↓
[4] CEF FIB와 Adjacency
        ↓
[5] Next-hop 도달성
        ↓
[6] Return Path
        ↓
[7] ACL·NAT·Firewall
```

핵심 명령어:

```cisco
show ip interface brief
show ip route <DESTINATION>
show ip cef <DESTINATION> detail
show adjacency detail
show arp
ping <NEXT_HOP>
ping <DESTINATION> source <SOURCE>
traceroute <DESTINATION>
```

---

### Q2. RIB에 경로가 있는데 왜 통신이 실패하는가?

👉 **핵심 답변:** RIB는 경로 선택 결과만 보여줍니다. 실제 통신에는 다음 조건이 모두 필요합니다.

```properties
목적지 경로
+ Next-hop 해석
+ L2 Adjacency
+ 정상 Interface
+ Return Path
+ ACL/NAT/Firewall 허용
```

---

## 2. 🛠️ 증상별 즉시 대응표

### 2-1. 경로 및 통신

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 목적지 경로 없음 | Static 누락·Next-hop 미해석 | `show ip route` | 경로/Connected 확인 |
| RIB에는 있지만 통신 실패 | ARP·Adjacency·Return Path | CEF/ARP/Ping | L2와 역방향 확인 |
| Ping이 편도 | Return Path 누락 | 상대 RIB | 역방향 경로 추가 |
| Next-hop Ping 실패 | L1/L2/IP 오류 | Interface/ARP | Link·VLAN·주소 확인 |
| 엉뚱한 경로 사용 | 더 구체적인 Prefix | `show ip route <IP>` | Longest Match 확인 |
| Ethernet ARP 증가 | Exit IF만 지정 | Running config/ARP | Next-hop 방식으로 변경 |
| Static이 안 사라짐 | Connected Route가 살아 있음 | RIB/ARP | IP SLA + Track |
| Static이 사라짐 | Recursive Next-hop 해석 실패 | Next-hop Route | Underlay 경로 복구 |

---

### 2-2. Floating Static

| 증상 | 원인 | 조치 |
|---|---|---|
| Backup이 평소 활성 | AD가 너무 낮음 | Backup AD 상향 |
| 장애 시 전환 안 됨 | Primary가 RIB에 남음 | Track 연동 |
| 전환 후 반복 Flap | SLA Probe가 Backup으로 우회 | Probe Host Route 고정 |
| 전환 후 원복 안 됨 | SLA/Track Down 유지 | Probe·Source·Route 확인 |
| Backup Next-hop 미해석 | Backup Underlay 경로 없음 | Connected/Recursive Route 확인 |

---

### 2-3. Serial Link

| 상태 | 의미 | 주요 확인 |
|---|---|---|
| `down/down` | 물리 계층 문제 | Cable·DCE/DTE·장비 |
| `up/down` | L2 문제 | Clock·Encapsulation·Keepalive |
| `administratively down` | Shutdown 상태 | `no shutdown` |
| `up/up`이나 IP 실패 | L3/ACL/주소 문제 | IP·Mask·Route·ACL |

---

## 3. 🔍 필수 진단 명령어

### Interface/L1·L2

```cisco
show ip interface brief
show interfaces
show controllers serial 1/0
show cdp neighbors
```

### RIB

```cisco
show ip route
show ip route static
show ip route <DESTINATION>
show ip route summary
show running-config | include ^ip route
```

### FIB/Adjacency

```cisco
show ip cef <DESTINATION>
show ip cef <DESTINATION> detail
show adjacency
show adjacency detail
show arp
```

### 통신

```cisco
ping <NEXT_HOP>
ping <DESTINATION>
ping <DESTINATION> source <INTERFACE_OR_IP>
traceroute <DESTINATION>
```

### IP SLA/Track

```cisco
show ip sla configuration
show ip sla statistics
show track
show track 1
```

---

## 4. 🚨 트러블슈팅 시나리오

### 시나리오 1. Ping 요청은 가지만 응답이 없음

#### 증상

```cisco
R1# ping 192.168.3.254 source 192.168.1.254
.....
```

#### 진단

R3에서 출발지 LAN 경로를 확인합니다.

```cisco
R3# show ip route 192.168.1.254
```

경로가 없다면 Return Path 누락입니다.

#### 조치

```cisco
R3(config)# ip route 192.168.1.0 255.255.255.0 192.168.23.2
```

중간 Router도 양쪽 경로를 모두 가져야 합니다.

```cisco
R2# show ip route 192.168.1.0
R2# show ip route 192.168.3.0
```

> Static Route는 요청과 응답 양쪽의 경로를 각각 확보해야 합니다.

---

### 시나리오 2. RIB에는 Static이 있는데 통신이 안 됨

#### 확인

```cisco
show ip route 10.20.30.5
show ip cef 10.20.30.5 detail
show adjacency detail
show arp
```

#### 원인 후보

- Next-hop ARP 미해결
- 잘못된 VLAN
- Ethernet Link는 Up이나 상대 장비 Down
- CEF Adjacency가 `incomplete`
- Return Path 누락
- ACL/Firewall 차단

Next-hop 확인:

```cisco
ping 10.1.1.2
show arp 10.1.1.2
```

> Connected Route가 살아 있으면 Next-hop 장비가 응답하지 않아도 Recursive Static Route가 RIB에 남을 수 있습니다. 단순히 `S` 코드가 보인다고 Next-hop이 살아 있는 것은 아닙니다.

---

### 시나리오 3. Ethernet에 Exit Interface만 지정

#### 위험 설정

```cisco
ip route 10.20.30.0 255.255.255.0 GigabitEthernet0/0
```

#### 증상

- 목적지 IP별 ARP 발생
- Incomplete ARP 증가
- Proxy ARP에 따라 통신 여부가 달라짐
- CPU/ARP Table 부담 가능

#### 조치

```cisco
no ip route 10.20.30.0 255.255.255.0 GigabitEthernet0/0
ip route 10.20.30.0 255.255.255.0 10.1.1.2
```

또는 Fully Specified Static:

```cisco
ip route 10.20.30.0 255.255.255.0 GigabitEthernet0/0 10.1.1.2
```

검증:

```cisco
show ip route 10.20.30.5
show ip cef 10.20.30.5 detail
show arp
```

---

### 시나리오 4. Floating Static이 활성화되지 않음

#### 설정

```cisco
ip route 10.20.30.0 255.255.255.0 10.1.1.2
ip route 10.20.30.0 255.255.255.0 10.2.2.2 200
```

#### 원인

Primary Next-hop이 같은 Ethernet Segment에 있고 Interface/Connected Route가 살아 있으면 Primary Static이 RIB에서 제거되지 않을 수 있습니다.

#### 확인

```cisco
show ip route 10.20.30.0
show arp 10.1.1.2
ping 10.1.1.2
```

#### 조치

IP SLA + Track을 사용합니다.

```cisco
ip route 198.51.100.10 255.255.255.255 10.1.1.2
!
ip sla 1
 icmp-echo 198.51.100.10 source-interface GigabitEthernet0/0
 frequency 5
!
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
!
ip route 10.20.30.0 255.255.255.0 10.1.1.2 track 1
ip route 10.20.30.0 255.255.255.0 10.2.2.2 200
```

검증:

```cisco
show ip sla statistics
show track 1
show ip route 10.20.30.0
```

---

### 시나리오 5. IP SLA가 Up/Down을 반복

#### 원인

Primary Track이 Down되면서 Primary Route가 제거된 후 SLA Probe가 Backup Route로 성공할 수 있습니다.

```properties
Primary Down
→ Primary Route 제거
→ Probe가 Backup으로 전송
→ Probe 성공
→ Primary Route 복귀
→ 다시 실패
→ 반복 Flapping
```

#### 조치

Probe 목적지 Host Route를 Primary로 고정합니다.

```cisco
ip route 198.51.100.10 255.255.255.255 10.1.1.2
```

확인:

```cisco
show ip route 198.51.100.10
show ip sla statistics
show track 1
```

---

### 시나리오 6. Serial Interface 장애

#### 상태 확인

```cisco
show ip interface brief
show interfaces Serial1/0
show controllers serial 1/0
```

#### DCE인 경우

```cisco
interface Serial1/0
 clock rate 64000
```

#### Encapsulation 확인

```cisco
show interfaces Serial1/0 | include Encapsulation
```

양쪽을 일치시킵니다.

```cisco
interface Serial1/0
 encapsulation hdlc
```

또는:

```cisco
interface Serial1/0
 encapsulation ppp
```

> Encapsulation 또는 Keepalive 불일치는 일반적으로 `up/down`과 같은 Line Protocol 장애를 유발합니다.

---

### 시나리오 7. `bandwidth 64`를 설정했는데 실제 속도가 바뀌지 않음

#### 원인

```cisco
bandwidth 64
```

명령은 Routing Metric, QoS, 관리 도구가 참조하는 논리적 Bandwidth 값을 변경합니다. 실제 전송 Clock이나 물리 속도를 직접 제한하지 않습니다.

#### 확인

```cisco
show interfaces Serial1/0
show controllers serial 1/0
```

실제 DCE Clock:

```cisco
interface Serial1/0
 clock rate 64000
```

Traffic Shaping이 필요하면 QoS 정책을 별도로 구성해야 합니다.

---

### 시나리오 8. AD가 낮은 Static 대신 다른 경로를 사용

#### 원인

더 구체적인 Prefix가 존재할 수 있습니다.

```text
S  0.0.0.0/0 [1/0] via 203.0.113.1
O  10.20.30.0/24 [110/20] via 10.1.1.2
```

목적지 `10.20.30.5`는 OSPF `/24`를 사용합니다. Static Default의 AD가 낮더라도 `/24`가 더 구체적입니다.

#### 확인

```cisco
show ip route 10.20.30.5
show ip cef 10.20.30.5
```

> AD는 동일 Prefix 후보끼리 비교할 때 사용합니다. 서로 다른 Prefix Length 간에는 Longest Prefix Match가 우선입니다.

---

### 시나리오 9. Source별로 다른 경로를 Static으로 만들려고 함

Static Route는 목적지 기반이므로 다음 요구는 일반 Static만으로 처리할 수 없습니다.

```properties
Source 10.1.0.0/16 → ISP-A
Source 10.2.0.0/16 → ISP-B
```

PBR 예시:

```cisco
ip access-list standard SRC_TO_ISP_B
 permit 10.2.0.0 0.0.255.255
!
route-map PBR_ISP_B permit 10
 match ip address SRC_TO_ISP_B
 set ip next-hop 192.0.2.1
!
interface GigabitEthernet0/1
 ip policy route-map PBR_ISP_B
```

> PBR은 별도 설계가 필요한 기능이며 Static Route와 목적이 다릅니다.

---

## 5. 🔧 현장 Quick Check

```cisco
show ip interface brief
show ip route <DESTINATION>
show ip cef <DESTINATION> detail
show arp
ping <NEXT_HOP>
ping <DESTINATION> source <SOURCE>
traceroute <DESTINATION>
show access-lists
```

Floating Static:

```cisco
show ip sla statistics
show track
show ip route <PREFIX>
```

Serial:

```cisco
show controllers serial <IF>
show interfaces serial <IF>
```

---

> 📌 **핵심 요약**
> - 진단 순서: **Interface → RIB → FIB → Adjacency → Next-hop → Return Path**
> - RIB에 Static이 보여도 Next-hop 생존을 보장하지 않음
> - Ethernet은 Next-hop 또는 Fully Specified Static 권장
> - Floating Static은 Primary Route가 RIB에서 빠져야 활성화
> - Silent Failure는 IP SLA + Track
> - AD는 동일 Prefix 후보, 포워딩은 Longest Prefix Match
> - 관련: 1.  📖   라우팅(Routing) 기초 개념 · 2. 📍 Static Route (정적 라우팅) · 3. ⚡  라우터의 스위칭 방식 (CEF) · 4. 🧭 라우팅 기초 개념 통합 정리 · 6. ⚡ 라우팅 기초 명령어 퀵 레퍼런스
