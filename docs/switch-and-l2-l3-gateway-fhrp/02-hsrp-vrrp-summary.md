# HSRP & VRRP 통합 정리

> **Tag:** #Network #FHRP #HSRP #VRRP #Redundancy #Cisco  
> **핵심 요약:** FHRP는 단말이 바라보는 Gateway를 가상 IP와 가상 MAC으로 추상화하여 Gateway 단일 장애점을 제거하는 기술입니다. HSRP는 Cisco 전용 Active/Standby 구조이며 Preempt가 기본 비활성이고, VRRP는 RFC 표준 Master/Backup 구조이며 Preempt가 기본 활성입니다. 실무의 성패는 Interface Tracking 설계와 STP Root Bridge 위치 정합성에서 갈립니다.

---

## 1. 핵심 기술 개념

### Q1. FHRP는 무엇을 해결하는가?

👉 **핵심 답변:** FHRP(First Hop Redundancy Protocol)는 단말의 Default Gateway가 되는 첫 홉 장비의 장애를 대비하여, 여러 L3 장비가 하나의 가상 IP와 가상 MAC을 공유하도록 만드는 프로토콜군입니다.

```properties
[FHRP 미적용] 단말 Gateway = 192.168.10.1 (물리 IP)
        ↓
해당 장비 장애 → 단말은 Gateway 상실 → 외부 통신 전면 중단(SPOF)

[FHRP 적용] 단말 Gateway = 192.168.10.254 (가상 IP)
        ↓
Active 장애 → Standby가 동일 가상 IP/MAC 인수
        ↓
단말 설정과 ARP Cache 변경 없이 통신 유지
```

| 구분 | 내용 |
|---|---|
| 해결 | Gateway 장비 장애, 재부팅, IOS Upgrade, LAN Interface 장애 |
| 미해결 | 단말~Access Switch 회선 장애, L2 Loop, Routing 설계 오류 |
| 조건부 | 상위 WAN 회선 장애(Interface Tracking 없이는 감지 불가) |

> FHRP는 Gateway 역할의 이중화이며 경로 전체의 이중화가 아닙니다. 상위 경로 장애 대응은 Tracking과 Routing 설계로 별도 보완해야 합니다.

---

### Q2. 가상 IP와 가상 MAC이 왜 함께 필요한가?

👉 **핵심 답변:** 단말은 Gateway를 IP로 지정하지만 실제 Frame 전달은 MAC 기준이므로, 가상 IP만 인수하고 MAC이 바뀌면 단말 ARP Cache가 만료될 때까지 통신이 끊깁니다.

```properties
단말 ARP Cache : 192.168.10.254 → 0000.0c07.ac01
        ↓
Active 장애 → Standby가 동일 가상 MAC 사용
        ↓
단말은 같은 MAC으로 계속 전송 → ARP 재학습 없이 절체 완료
```

| 프로토콜 | 가상 MAC 형식 | 그룹 표현 | 예시(Group 10) |
|---|---|---|---|
| HSRPv1 | `0000.0c07.acXX` | Hex 1Byte | `0000.0c07.ac0a` |
| HSRPv2 | `0000.0c9f.fXXX` | Hex 3Nibble | `0000.0c9f.f00a` |
| VRRP | `0000.5e00.01XX` | Hex 1Byte | `0000.5e00.010a` |

```cisco
show standby all
show mac address-table vlan <VLAN-ID>
```

> HSRPv1은 가상 MAC 마지막 1Byte만 Group에 사용하므로 Group 범위가 0~255로 제한됩니다. HSRPv2는 3Nibble을 사용하여 0~4095를 표현합니다.

---

### Q3. HSRP, VRRP, GLBP는 무엇이 다른가?

👉 **핵심 답변:** 역할 명칭, Preempt 기본값, 가상 MAC, 전송 방식이 다르며 특히 Preempt 기본값 차이가 실무 사고의 가장 흔한 원인입니다.

| 항목 | HSRP | VRRP | GLBP |
|---|---|---|---|
| 표준 | Cisco 전용 | RFC 표준 | Cisco 전용 |
| 역할 | Active / Standby | Master / Backup | AVG / AVF |
| Priority 기본값 | 100 | 100 | 100 |
| IP Owner Priority | 해당 없음 | 255 고정 | 해당 없음 |
| Preempt 기본값 | 비활성 | 활성 | 활성 |
| 가상 MAC | v1 `0000.0c07.acXX`<br>v2 `0000.0c9f.fXXX` | `0000.5e00.01XX` | `0007.b400.xxyy` 계열 |
| 전송 | UDP 1985 | IP Protocol 112 | UDP 3222 |
| 멀티캐스트 | v1 224.0.0.2<br>v2 224.0.0.102 | 224.0.0.18 | 224.0.0.102 |
| Hello | 3초 | 1초(Advertisement) | 3초 |
| Hold / Dead | 10초 | Master Down 계산값 | 10초 |
| Group 범위 | v1 0~255 / v2 0~4095 | 1~255 | 0~1023 |
| 부하 분산 | 그룹 분할(MHSRP) | 그룹 분할 | 단일 그룹 내 자동 |

선택 기준은 순수 Cisco 환경이면 HSRP, 멀티벤더 또는 표준 요구가 있으면 VRRP, 단일 그룹에서 자동 부하 분산이 필요하면 GLBP입니다.

> GLBP는 하나의 그룹에서 여러 Forwarder가 서로 다른 가상 MAC을 단말에 응답해 부하를 분산합니다. Cisco 전용이며 플랫폼별 지원 여부 확인이 필요하므로 학습 순서는 HSRP와 VRRP를 우선합니다.

---

### Q4. Priority, Preempt, Preempt Delay는 어떻게 동작하는가?

👉 **핵심 답변:** Priority가 높은 장비가 Active가 되지만 HSRP는 Preempt가 기본 비활성이라 이미 Active가 존재하면 역할을 가져오지 않으며, Preempt를 켤 때는 Routing 수렴 전 탈환을 막기 위해 Delay가 필요합니다.

```properties
Priority 기본값 100, 범위 0~255, 높은 값 우선
Priority 동일 시 → 더 높은 Interface IP를 가진 장비가 Active

[Preempt 비활성 = HSRP 기본]
R1(150) 장애 → R2(100) Active → R1 복구 → R1은 Standby 유지
= 설계 의도와 실제 Active 불일치

[Preempt 활성 + Delay 없음]
Interface Up 후 수초 만에 탈환하지만 Routing 수렴은 수십초 소요
= 빈 Routing Table로 Traffic 수신 → Black Hole

[Preempt 활성 + Delay 60초]
Interface Up → Routing 수렴 대기 → 안전하게 Active 탈환
```

```cisco
interface <INTERFACE>
 standby <GROUP> ip <IP-ADDRESS>
 standby <GROUP> priority 150
 standby <GROUP> preempt delay minimum 60
```

```cisco
show standby brief
show standby <INTERFACE> <GROUP>
show ip ospf neighbor
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Delay 값은 해당 망의 Routing 수렴 시간보다 넉넉하게 설정합니다. 대규모 OSPF Area나 BGP 환경에서는 60초보다 큰 값이 필요할 수 있습니다. `show standby brief`의 `P` 열이 Preempt 활성 여부를 나타냅니다.

---

### Q5. Interface Tracking과 Decrement는 어떻게 설계하는가?

👉 **핵심 답변:** FHRP는 자신이 설정된 Interface 상태만 판단하므로 WAN 장애를 감지하려면 Tracking이 필요하며, Decrement는 두 장비의 Priority 차이보다 커야 실제 절체가 일어납니다.

```properties
[Tracking 미설정]
LAN Up → Active 유지 / WAN Down → 상위 경로 없음
단말 Traffic → Active로 전달 → 폐기 → Black Hole(Ping U.U.U)

[Tracking 설정]
WAN Down 감지 → Priority 150 - 100 = 50
        ↓
Standby(100) > Active(50) → Standby가 Preempt로 탈환 → 정상 경로 우회

[설계 공식] Decrement > (Active Priority - Standby Priority)

150 vs 100, Decrement 100 → 50 < 100 → 절체 성공
150 vs 100, Decrement 30  → 120 > 100 → 절체 실패(Black Hole 유지)

[다중 회선 설계: Active Priority 150]
회선 1개 Down(-30) → 120 : Active 유지 (잔여 회선 존재, 정상)
회선 2개 Down(-60) → 90  : Standby(100)가 탈환 (경로 소진, 정상)
```

Track Object 방식을 권장하며 Interface 직접 추적도 가능합니다.

```cisco
track 10 interface <INTERFACE> line-protocol
track 20 interface <INTERFACE> line-protocol
!
interface <INTERFACE>
 standby <GROUP> priority 150
 standby <GROUP> preempt delay minimum 60
 standby <GROUP> track 10 decrement 30
 standby <GROUP> track 20 decrement 30
!
! 간이 문법
interface <INTERFACE>
 standby <GROUP> track <INTERFACE> <DECREMENT>
```

```cisco
show track brief
show standby all
show ip interface brief
```

> Decrement 기본값은 10이므로 값을 생략하면 대부분의 설계에서 절체가 발생하지 않습니다. 또한 Tracking은 Standby 측에 Preempt가 있어야 절체로 완결되며, Priority만 떨어지고 Preempt가 없으면 역할은 그대로 유지됩니다.

---

### Q6. HSRP State는 어떤 순서로 전이하는가?

```properties
Disabled → Init → Listen → Speak → Standby → Active
```

| State | 의미 | 주요 확인 사항 |
|---|---|---|
| Init | 동작 시작 전 | Interface/SVI Up, 가상 IP Subnet 일치 |
| Listen | 선출 대기 | 동일 그룹에 이미 Active/Standby 존재 |
| Speak | 선출 참여 | Hello 송수신, 타이머 일치 |
| Standby | 대기 정상 | Priority와 Preempt 설정 |
| Active | 정상 서비스 | 가상 MAC, Tracking, 상위 경로 |

```cisco
show standby all
debug standby events
undebug all
```

> `debug` 명령은 운영망에서 CPU 부하와 Console 폭주를 유발할 수 있습니다. 사용 후 반드시 해제하십시오.

---

### Q7. VRRP는 HSRP와 무엇이 다른가?

👉 **핵심 답변:** VRRP는 RFC 표준이며 Master/Backup 용어를 쓰고, Preempt가 기본 활성이며 IP Protocol 112를 사용합니다. 가상 IP를 물리 Interface IP와 동일하게 설정하면 그 장비가 IP Owner가 되어 Priority 255를 갖습니다.

```properties
Priority 100 / IP Owner 255 고정 / Preempt 활성
Advertisement 1초 / Multicast 224.0.0.18 / IP Protocol 112
VRID 1~255 / 가상 MAC 0000.5e00.01XX

IP Owner : 가상 IP == 해당 Interface 실제 IP → Priority 255 → 항상 Master
```

구 문법과 신규 FHRP 문법이 공존합니다.

```cisco
! === 구 문법 ===
interface <INTERFACE>
 ip address <IP-ADDRESS> 255.255.255.0
 vrrp <GROUP> ip <IP-ADDRESS>
 vrrp <GROUP> priority 150
 vrrp <GROUP> preempt delay minimum 60
 vrrp <GROUP> track 10 decrement 100
!
! === 신규 FHRP 문법 ===
fhrp version vrrp v3
!
interface <INTERFACE>
 vrrp <GROUP> address-family ipv4
  address <IP-ADDRESS> primary
  priority 150
  preempt delay minimum 60
```

```cisco
show vrrp brief
show vrrp all
show vrrp interface <INTERFACE>
```

> VRRP 문법은 IOS 버전과 플랫폼에 따라 크게 다릅니다. 두 문법이 공존하지 않는 장비도 있어 플랫폼별 지원 여부 확인이 필요하며, Packet Tracer는 VRRP를 지원하지 않거나 옵션이 제한될 수 있습니다.

---

### Q8. HSRPv1과 HSRPv2는 무엇이 다른가?

| 항목 | HSRPv1 | HSRPv2 |
|---|---|---|
| Group 범위 | 0 ~ 255 | 0 ~ 4095 |
| 가상 MAC | `0000.0c07.acXX` | `0000.0c9f.fXXX` |
| 멀티캐스트 | 224.0.0.2 | 224.0.0.102 |
| UDP Port | 1985 | 1985 |
| Millisecond 타이머 | 제한적 | 지원 |
| IPv6 | 미지원 | 지원 |
| 상호 호환 | v2와 호환 불가 | v1과 호환 불가 |

```properties
R1 = v1, R2 = v2 (같은 Segment)
        ↓
서로 다른 멀티캐스트 주소와 가상 MAC 사용
        ↓
상대 Hello 미인식 → 양쪽 모두 Active → 가상 IP 중복 및 통신 불안정
```

```cisco
interface <INTERFACE>
 standby version 2
 standby <GROUP> ip <IP-ADDRESS>
```

```cisco
show standby all
show running-config interface <INTERFACE>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `standby version 2`는 그룹 단위가 아니라 Interface 단위 설정입니다. Version을 바꾸면 기존 그룹이 재초기화되어 순간 절체가 발생하며, VLAN ID를 Group 번호로 쓰면서 VLAN 256 이상을 다루는 설계에서는 v2가 필수입니다.

---

### Q9. MHSRP로 부하 분산은 어떻게 구성하는가?

👉 **핵심 답변:** HSRP는 그룹당 한 대만 Forwarding하므로, 그룹을 2개 이상 만들고 VLAN 또는 단말 그룹별로 서로 다른 가상 IP를 Gateway로 지정해 두 장비를 모두 활용합니다.

```properties
Group 10 (VLAN 10) : R1 Active, R2 Standby
Group 20 (VLAN 20) : R2 Active, R1 Standby
        ↓
VLAN 단위로 Uplink 부하 분산
        ↓
STP Root Bridge도 동일한 VLAN 기준으로 정합성 유지
```

```cisco
! === R1 : VLAN 10에서 우세, VLAN 20에서 열세 ===
interface vlan 10
 ip address 192.168.10.252 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 track 10 decrement 60
!
interface vlan 20
 ip address 192.168.20.252 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 priority 100
 standby 20 preempt
!
! === R2 : IP 주소만 .253으로 바꾸고 Priority 150/100을 반대로 적용 ===
```

```cisco
show standby brief
show spanning-tree vlan 10
show spanning-tree vlan 20
show ip interface brief
```

> DHCP로 Gateway를 배포하는 환경에서는 각 VLAN Pool의 `default-router`가 해당 VLAN의 가상 IP를 가리켜야 합니다. 자세한 내용은 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

### Q10. STP Root Bridge와 FHRP Active를 왜 맞춰야 하는가?

👉 **핵심 답변:** L2 경로는 STP가 결정하고 L3 Gateway는 FHRP가 결정하므로, 두 위치가 다르면 Traffic이 한 번 우회한 뒤 Inter-Switch Link를 거쳐 반대편으로 건너가는 비효율 경로를 탑니다.

```properties
[불일치] STP Root = SW-A, FHRP Active = SW-B
단말 Frame은 STP 기준으로 SW-A 방향 Uplink 사용
        ↓
SW-A가 가상 MAC 목적지를 SW-B로 전달 (Inter-Switch Link 경유)
        ↓
Uplink 대역 낭비, Latency 증가, ISL 장애 시 영향 확대

[정합] VLAN 10 : Root=SW-A, Active=SW-A / VLAN 20 : Root=SW-B, Active=SW-B
        ↓
L2 최단 경로와 L3 Gateway 위치 일치 → 예측 가능한 부하 분산
```

STP 판정 순서는 Root Bridge(Bridge ID 최소) → Root Port(Root Path Cost 최소) → Designated Port(Segment별) → Alternate/Blocking입니다.

```cisco
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
!
interface vlan 10
 standby 10 priority 150
 standby 10 preempt
```

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree root
show standby brief
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

관련 개념은 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence와 3-4. 🚀 RSTP & MSTP (초대형망 STP)에서 확인하십시오.

---

### Q11. L3 Switch SVI에서 FHRP를 구성할 때 주의점은?

👉 **핵심 답변:** SVI는 해당 VLAN이 존재하고 그 VLAN에 Up 상태 Port가 있어야 Up이 되므로, VLAN Database와 Trunk Allowed VLAN이 정합해야 FHRP도 정상 동작합니다.

```properties
SVI Up 조건
[1] VLAN이 VLAN Database에 존재
[2] 해당 VLAN에 Up/Up Access Port 또는 Trunk 존재
[3] SVI가 no shutdown
[4] Trunk Allowed VLAN에 해당 VLAN 포함
        ↓
미충족 → SVI Down → HSRP State가 Init에서 벗어나지 못함
```

```cisco
ip routing
!
vlan 10
 name USERS
!
interface vlan 10
 ip address 192.168.10.252 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 150
 standby 10 preempt delay minimum 60
!
interface <INTERFACE>
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

```cisco
show ip interface brief
show vlan brief
show interfaces trunk
show standby brief
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

SVI와 Inter-VLAN Routing 개념은 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing, Uplink 이중화는 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3)을 참고하십시오.

---

### Q12. FHRP와 경로 선택은 어떻게 연결되는가?

👉 **핵심 답변:** FHRP는 단말이 어느 Gateway로 Frame을 보낼지만 결정하며, 그 Packet을 어디로 보낼지는 Gateway의 Routing Table 판정이 결정합니다.

```properties
경로 선택 순서
[1] Longest Prefix Match (가장 구체적인 Prefix 우선)
[2] 동일 Prefix 내 Administrative Distance 비교
[3] 동일 AD 내 Metric 비교
[4] 동일 Metric이면 Load Balancing

잘못된 판단 : "AD가 낮으니 이 경로로 간다"
올바른 판단 : "LPM으로 후보를 좁히고, AD를 비교하고, 같으면 Metric을 본다"
```

```cisco
show ip route <IP-ADDRESS>
show ip ospf neighbor
show ip protocols
```

> Active 절체가 성공했어도 새 Active의 Routing Table이 비어 있으면 통신은 실패합니다. 절체 검증은 State 확인과 경로 확인을 항상 함께 수행하십시오.

---

## 2. 표준 설정 템플릿

### 2-1. HSRP Active 및 Standby

```cisco
! === Active : Priority 높음 + Preempt Delay + Tracking ===
interface <INTERFACE>
 ip address 192.168.10.252 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 timers 3 10
 standby 10 track 10 decrement 100
!
! === Standby : Priority 기본 + Preempt 필수 ===
interface <INTERFACE>
 ip address 192.168.10.253 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 100
 standby 10 preempt
 standby 10 timers 3 10
```

```cisco
show standby brief
show track brief
show ip route
```

> 기본 타이머는 Hello 3초, Hold 10초입니다. 타이머는 같은 그룹의 모든 장비에서 동일해야 하며, Priority가 낮은 쪽에도 Preempt가 없으면 Tracking이 동작해도 절체되지 않습니다.

---

### 2-2. 인증 및 그룹 이름

```cisco
key chain HSRP_KEY
 key 1
  key-string cisco123
!
! 아래 세 방식 중 하나만 선택합니다.
interface <INTERFACE>
 standby <GROUP> authentication cisco123
 standby <GROUP> authentication md5 key-string cisco123
 standby <GROUP> authentication md5 key-chain HSRP_KEY
 standby <GROUP> name IPSEC_R
```

```cisco
show standby all
show key chain
show crypto isakmp peer
```

> 인증이 한쪽만 설정되면 상대 Hello가 거부되어 Active-Active가 되고 `%HSRP-4-BADAUTH` 계열 Log가 발생합니다. 인증 옵션 지원 범위는 플랫폼별 확인이 필요하며 Packet Tracer는 Key Chain을 지원하지 않을 수 있습니다. 그룹 이름은 `crypto map ... redundancy <NAME>` 연동에 사용되므로 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동)을 함께 확인하십시오.

---

### 2-3. VRRP Master 및 Backup

```cisco
! === Master ===
interface <INTERFACE>
 ip address 192.168.10.252 255.255.255.0
 vrrp 10 ip 192.168.10.254
 vrrp 10 priority 150
 vrrp 10 preempt delay minimum 60
 vrrp 10 timers advertise 1
 vrrp 10 track 10 decrement 100
!
! === Backup : Preempt 기본 활성이므로 별도 설정 불필요 ===
interface <INTERFACE>
 ip address 192.168.10.253 255.255.255.0
 vrrp 10 ip 192.168.10.254
 vrrp 10 priority 100
```

```cisco
show vrrp brief
show vrrp interface <INTERFACE>
```

> 신규 FHRP 문법만 지원하는 플랫폼에서는 `fhrp version vrrp v3`와 `address-family ipv4` 구문을 사용합니다. 지원 여부는 플랫폼별 확인이 필요합니다.

---

## 3. 안전한 적용 절차

```properties
[1] 현재 확인 : State, Priority, Tracking, Routing, STP 위치 기록
[2] 백업      : running-config 및 대상 Interface 설정 저장
[3] 경로 확인 : Console 또는 OOB 접근 경로 확보
[4] 적용      : Standby 측 먼저 설정한 뒤 Active 측 적용
[5] 검증      : State, 가상 MAC, Tracking, Routing, 실제 서비스
[6] 저장      : copy running-config startup-config
```

```cisco
! === 변경 전 수집 ===
show running-config interface <INTERFACE>
show standby brief
show vrrp brief
show ip route
show spanning-tree vlan <VLAN-ID>
!
! === 변경 후 검증 ===
show standby all
show track brief
show ip route
show arp
show mac address-table vlan <VLAN-ID>
```

Rollback:

```cisco
interface <INTERFACE>
 no standby <GROUP> priority
 no standby <GROUP> preempt
 no standby <GROUP> track 10 decrement 100
 no standby version 2
 no standby <GROUP> ip <IP-ADDRESS>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 4. 실무 체크리스트

- [ ] 같은 그룹의 모든 장비에 동일한 가상 IP와 Group 번호를 설정했는가?
- [ ] HSRP Version이 양쪽에서 동일한가?
- [ ] HSRP Standby 측에 Preempt를 설정했는가?
- [ ] Preempt Delay가 Routing 수렴 시간보다 긴가?
- [ ] Tracking Decrement가 Priority 차이보다 큰가?
- [ ] 인증 방식과 Key, 타이머 값이 양쪽에서 일치하는가?
- [ ] 단말 Default Gateway와 DHCP `default-router`가 가상 IP인가?
- [ ] STP Root Bridge와 FHRP Active 위치가 VLAN별로 정합한가?
- [ ] SVI가 Up이며 Trunk Allowed VLAN에 해당 VLAN이 포함되는가?
- [ ] 절체 후 새 Active의 Routing Table을 확인했는가?
- [ ] 변경 전 백업, Rollback 절차, Console/OOB 접근을 준비했는가?

---

> 📌 **핵심 요약**
> - FHRP는 가상 IP와 가상 MAC으로 Gateway 단일 장애점을 제거한다.
> - 가상 MAC이 승계되므로 단말은 ARP 재학습 없이 통신을 유지한다.
> - HSRP는 Cisco 전용 Active/Standby, VRRP는 RFC 표준 Master/Backup이다.
> - Priority 기본값은 양쪽 모두 100이며 Preempt는 HSRP만 기본 비활성이다.
> - VRRP에서 가상 IP를 소유한 장비는 Priority 255로 고정된다.
> - HSRP 기본 타이머는 Hello 3초, Hold 10초이며 UDP 1985를 사용한다.
> - HSRPv1은 224.0.0.2, HSRPv2는 224.0.0.102, VRRP는 224.0.0.18을 사용한다.
> - HSRPv1 Group은 0~255, HSRPv2 Group은 0~4095이며 v1과 v2는 호환되지 않는다.
> - VRRP는 IP Protocol 112를 사용하고 가상 MAC은 `0000.5e00.01XX`이다.
> - Tracking 없이는 WAN 장애를 감지하지 못해 Black Hole이 발생한다.
> - Decrement는 Priority 차이보다 크게 설계해야 실제 절체가 일어난다.
> - MHSRP와 VLAN별 그룹 분할로 두 장비를 모두 활용한다.
> - STP Root Bridge와 FHRP Active 위치를 VLAN별로 맞춰야 경로가 최적화된다.
> - 경로 선택은 Longest Prefix Match → AD → Metric 순서로 판단한다.
> - 관련: 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy) · 7-2. 🧩 HSRP & VRRP 통합 정리 · 7-3. 🚑 HSRP & VRRP 트러블슈팅 치트시트 · 7-4. ⚡ HSRP & VRRP 명령어 퀵 레퍼런스 · 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동)
