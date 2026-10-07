# 라우팅 기초 명령어 퀵 레퍼런스

> **Tag:** #Network #QuickReference #Routing #Static #FloatingStatic #IPSLA #RIB #CEF #AD #CheatSheet
> **핵심 요약:** Static Route·Floating Static·IP SLA Track·RIB/FIB 검증 명령어를 빠르게 조회하기 위한 암기 카드. Static은 목적지 기반 경로이며, Ethernet에서는 Next-hop 또는 Fully Specified Static을 권장한다.

---

## 1. Static Route 문법

### 1-1. Default Route

```cisco
ip route 0.0.0.0 0.0.0.0 <NEXT_HOP>
```

예시:

```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

---

### 1-2. Next-hop 방식

```cisco
ip route <DESTINATION> <MASK> <NEXT_HOP>
```

예시:

```cisco
ip route 10.20.30.0 255.255.255.0 10.1.1.2
```

권장 환경:

```properties
Ethernet
Multi-Access Network
일반적인 Static Route
```

---

### 1-3. Exit Interface 방식

```cisco
ip route <DESTINATION> <MASK> <EXIT_INTERFACE>
```

예시:

```cisco
ip route 10.20.30.0 255.255.255.0 Serial0/0
```

권장 환경:

```properties
HDLC
PPP
Point-to-Point Link
```

---

### 1-4. Fully Specified Static

```cisco
ip route <DESTINATION> <MASK> <EXIT_INTERFACE> <NEXT_HOP>
```

예시:

```cisco
ip route 10.20.30.0 255.255.255.0 GigabitEthernet0/0 10.1.1.2
```

권장 환경:

```properties
Ethernet/Multi-Access에서 Interface와 Next-hop을 모두 명시할 때
```

---

### 1-5. Floating Static

```cisco
ip route <DESTINATION> <MASK> <NEXT_HOP> <AD>
```

예시:

```cisco
ip route 10.20.30.0 255.255.255.0 10.1.1.2
ip route 10.20.30.0 255.255.255.0 10.2.2.2 200
```

---

### 1-6. Track 연동 Static

```cisco
ip route <DESTINATION> <MASK> <NEXT_HOP> track <TRACK_ID>
```

예시:

```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 1
```

---

### 1-7. Static Route 삭제

```cisco
no ip route <DESTINATION> <MASK> <NEXT_HOP>
```

예시:

```cisco
no ip route 10.20.30.0 255.255.255.0 10.1.1.2
```

> 삭제 명령은 기존 설정과 Interface, Next-hop, AD, Track 옵션까지 일치해야 할 수 있습니다. `show running-config | include ^ip route`로 원본을 먼저 확인합니다.

---

## 2. 경로 선택 빠른 조회

### 2-1. RIB 설치

```properties
동일 Prefix + 서로 다른 출처
→ AD 비교

동일 Prefix + 같은 프로토콜
→ Metric 비교
```

### 2-2. Packet Forwarding

```properties
Longest Prefix Match
→ 가장 구체적인 Prefix 사용
```

예시:

```text
0.0.0.0/0
10.0.0.0/8
10.20.0.0/16
10.20.30.0/24  ← 10.20.30.5가 사용
```

---

## 3. Administrative Distance

| 경로 출처 | 기본 AD |
|---|---:|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| EIGRP Internal | 90 |
| OSPF | 110 |
| RIP | 120 |
| EIGRP External | 170 |
| iBGP | 200 |

```properties
낮은 AD가 우선
단, 동일 Prefix 후보끼리 비교
```

---

## 4. Static vs Dynamic

| 구분 | Static | Dynamic |
|---|---|---|
| 등록 | 관리자 수동 | 프로토콜 자동 |
| 제어 패킷 | 없음 | 있음 |
| 자원 부담 | 낮음 | 상대적으로 높음 |
| 장애 대응 | 기본 수동 | 자동 수렴 |
| 확장성 | 낮음 | 높음 |
| 환경 | Stub·Default·경계 | 캠퍼스·DC·WAN |

---

## 5. Next-hop vs Exit Interface

| 방식 | 환경 | 주의점 |
|---|---|---|
| Next-hop | Ethernet/Multi-Access | 재귀 조회 |
| Exit Interface | P2P | HDLC/PPP에 적합 |
| Exit IF + Next-hop | Ethernet | 명확한 L2/L3 Next-hop |

위험 가능 설정:

```cisco
ip route 10.20.30.0 255.255.255.0 GigabitEthernet0/0
```

권장:

```cisco
ip route 10.20.30.0 255.255.255.0 10.1.1.2
```

또는:

```cisco
ip route 10.20.30.0 255.255.255.0 GigabitEthernet0/0 10.1.1.2
```

---

## 6. IP SLA + Track

### 6-1. SLA 생성

```cisco
ip sla 1
 icmp-echo <PROBE_IP> source-interface <PRIMARY_INTERFACE>
 frequency 5
```

### 6-2. SLA 시작

```cisco
ip sla schedule 1 life forever start-time now
```

### 6-3. Track 연결

```cisco
track 1 ip sla 1 reachability
```

### 6-4. Primary/Backup Route

```cisco
ip route <PROBE_IP> 255.255.255.255 <PRIMARY_NEXT_HOP>
ip route 0.0.0.0 0.0.0.0 <PRIMARY_NEXT_HOP> track 1
ip route 0.0.0.0 0.0.0.0 <BACKUP_NEXT_HOP> 200
```

전체 예시:

```cisco
ip route 198.51.100.10 255.255.255.255 203.0.113.1
!
ip sla 1
 icmp-echo 198.51.100.10 source-interface GigabitEthernet0/0
 frequency 5
!
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 1
ip route 0.0.0.0 0.0.0.0 192.0.2.1 200
```

> Probe가 Backup Route로 우회해 Track이 Flap하지 않도록 Probe Host Route를 Primary에 고정합니다.

---

## 7. Serial Link 기본 설정

DCE 측:

```cisco
interface Serial1/0
 encapsulation hdlc
 clock rate 64000
 bandwidth 64
 ip address 192.168.12.1 255.255.255.0
 no shutdown
```

DTE 측:

```cisco
interface Serial1/0
 encapsulation hdlc
 bandwidth 64
 ip address 192.168.12.2 255.255.255.0
 no shutdown
```

DCE/DTE 확인:

```cisco
show controllers serial 1/0
```

> `bandwidth`는 Metric/QoS Reference이고, 실제 Clock은 DCE의 `clock rate`가 결정합니다.

---

## 8. RIB 검증

```cisco
show ip route
show ip route <DESTINATION_IP>
show ip route static
show ip route connected
show ip route summary
show ip protocols
show running-config | include ^ip route
```

특정 목적지:

```cisco
show ip route 10.20.30.5
```

확인 항목:

```properties
선택 Prefix
Route Source
AD/Metric
Next-hop
Outgoing Interface
Route Age
```

---

## 9. CEF/FIB/Adjacency 검증

```cisco
show ip cef summary
show ip cef <DESTINATION>
show ip cef <DESTINATION> detail
show adjacency
show adjacency detail
show arp
```

```properties
RIB        = Control Plane 경로 선택
FIB        = 목적지 Prefix Forwarding 정보
Adjacency  = Next-hop L2 Rewrite 정보
```

---

## 10. 연결성 검증

```cisco
show ip interface brief
show interfaces
ping <NEXT_HOP>
ping <DESTINATION>
ping <DESTINATION> source <INTERFACE_OR_IP>
traceroute <DESTINATION>
show cdp neighbors
```

Serial:

```cisco
show controllers serial <INTERFACE>
show interfaces serial <INTERFACE>
```

---

## 11. IP SLA/Track 검증

```cisco
show ip sla configuration
show ip sla statistics
show track
show track 1
show ip route 0.0.0.0
```

---

## 12. 증상별 Quick Check

| 증상 | 먼저 확인 |
|---|---|
| 목적지 경로 없음 | `show ip route <IP>` |
| RIB에는 있지만 Ping 실패 | CEF·ARP·Return Path |
| Next-hop Ping 실패 | L1/L2·VLAN·주소 |
| Ethernet ARP 증가 | Exit IF Static 여부 |
| Floating 전환 안 됨 | Primary가 RIB에 남는지 |
| Track Flapping | Probe가 Backup으로 우회하는지 |
| Serial down/down | Cable·DCE/DTE |
| Serial up/down | Clock·Encapsulation |
| 엉뚱한 경로 | Longest Prefix Match |
| AD 예상과 다름 | 동일 Prefix인지 확인 |

---

## 13. 핵심 주의사항

```properties
Static Route ≠ PBR
Static Route ≠ Next-hop 생존 보장
RIB 경로 존재 ≠ 실제 통신 보장
bandwidth 명령 ≠ 물리 속도 제한
Floating Static ≠ End-to-End 자동 감시
```

```properties
Silent Failure
→ IP SLA + Track

Ethernet Multi-Access
→ Next-hop 또는 Fully Specified Static

RIB 통과 후 실제 전달
→ CEF FIB + Adjacency 확인
```

---

> 📌 **핵심 요약**
> - Default: `ip route 0.0.0.0 0.0.0.0 <NEXT_HOP>`
> - Ethernet: Next-hop 또는 Fully Specified Static
> - P2P: Exit Interface 사용 가능
> - Backup: Floating Static + 필요 시 IP SLA Track
> - RIB 설치: AD·Metric
> - Packet Forwarding: Longest Prefix Match
> - 검증: RIB → FIB → Adjacency → Return Path
> - 관련: 1.  📖   라우팅(Routing) 기초 개념 · 2. 📍 Static Route (정적 라우팅) · 3. ⚡  라우터의 스위칭 방식 (CEF) · 4. 🧭 라우팅 기초 개념 통합 정리 · 5. 🚑 라우팅 기초 트러블슈팅 치트시트
