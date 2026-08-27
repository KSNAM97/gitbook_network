# 🧩 L3 Switch & Inter-VLAN Routing 통합 정리

> **Tag:** #Network #Switch #L3Switch #InterVLAN #SVI #Cisco  
> **핵심 요약:** Inter-VLAN Routing은 Router-on-a-Stick(Sub-interface), SVI, Routed Port로 구현한다. L3 Switch에서 라우팅이 동작하려면 전역 `ip routing`이 필수이며, SVI는 VLAN Database 존재·해당 VLAN의 Up 상태 L2 Port·`no shutdown`을 모두 만족해야 Up/Up이 된다. Multi-Layer Switch는 제어평면이 만든 경로를 CEF 기반 하드웨어 테이블로 내려 와이어스피드로 전달한다.

---

## 1. 🎯 핵심 기술 개념

### Q1. Inter-VLAN Routing은 왜 필요한가?

👉 **핵심 답변:** VLAN은 Broadcast Domain을 분리하므로 서로 다른 VLAN은 L2 Forwarding만으로 통신할 수 없고, 반드시 L3 기능이 IP 헤더를 보고 라우팅해야 하기 때문입니다.

| 계층 | 판단 기준 | 관련 요소 |
|---|---|---|
| L1 | 전기·광 신호 | Cable, SFP, Link |
| L2 | Destination MAC | VLAN, MAC Table, STP |
| L3 | Destination IP | Routing Table, SVI, Routed Port |

```properties
VLAN 10 (192.168.10.0/24) ↔ VLAN 20 (192.168.20.0/24)
같은 Switch에 있어도 Broadcast Domain이 다름
        ↓
L3 Gateway(SVI 또는 Router)가 라우팅해야 통신 가능
```

> VLAN은 Broadcast Domain 분리 수단이며 완전한 보안 경계가 아닙니다. VLAN 간 차단이 목적이라면 ACL, VRF, 방화벽 같은 명시적 정책이 필요합니다.

---

### Q2. Inter-VLAN Routing 구현 방식은 무엇이 있는가?

👉 **핵심 답변:** Legacy Inter-VLAN(VLAN마다 물리 링크), Router-on-a-Stick(Sub-interface), SVI, Routed Port로 나뉘며 현대 캠퍼스망 표준은 SVI + Routed Port 조합입니다.

| 방식 | 구성 위치 | 특징 | 한계 |
|---|---|---|---|
| Legacy Inter-VLAN | Router 물리 IF ×N | 구성 단순 | VLAN 수만큼 포트 소모 |
| Router-on-a-Stick | Router Sub-interface | 링크 1개로 다수 VLAN | 단일 Trunk 병목 |
| SVI | `interface vlan <VLAN-ID>` | VLAN Gateway, 하드웨어 라우팅 | VLAN·L2 상태에 종속 |
| Routed Port | `no switchport` | P2P L3 링크, STP 미참여 | 단일 세그먼트만 수용 |

```properties
단말 다수의 사용자 VLAN Gateway → SVI
Switch 간 P2P 라우팅 업링크     → Routed Port
Switch가 L2뿐이고 Router만 있음 → Router-on-a-Stick
```

---

### Q3. Router-on-a-Stick은 어떻게 동작하는가?

👉 **핵심 답변:** Switch와 Router를 Trunk로 연결하고 Router 물리 인터페이스를 VLAN 단위 Sub-interface로 분할하여, 각 Sub-interface가 해당 VLAN의 Gateway IP를 갖게 하는 방식입니다.

```properties
PC(VLAN 10) → Access → Trunk(Tag 10)
        ↓
Sub-interface .10 이 Tag 해제 후 라우팅
        ↓
Sub-interface .20 이 Tag 20 부착 → Switch → PC(VLAN 20)
```

- Switch 측 연결 포트는 반드시 Trunk이며 Access로 남으면 통신되지 않습니다.
- Router 물리 인터페이스는 IP 없이 `no shutdown`만 수행합니다.
- Sub-interface IP가 해당 VLAN 단말의 Default Gateway입니다.
- Sub-interface 번호를 VLAN ID와 맞추는 것은 관례이며, 실제 VLAN 판별은 `encapsulation dot1q <VLAN-ID>` 값이 결정합니다.

---

### Q4. Router-on-a-Stick에서 Native VLAN은 어떻게 처리하는가?

👉 **핵심 답변:** Native VLAN 프레임은 Trunk에서 Untagged로 전달되므로 Router 측에서도 Untagged 처리를 명시해야 하며, 양단 Native VLAN이 불일치하면 VLAN Leaking이 발생합니다.

```properties
802.1Q Trunk : Native VLAN → Untagged / 그 외 → Tagged

SW1 Native VLAN 1 ↔ SW2 Native VLAN 99
        ↓
VLAN 1 Untagged 프레임이 VLAN 99로 유입
        ↓
VLAN Leaking (Broadcast Domain 병합, STP 오판 가능)
```

```cisco
interface <INTERFACE>.<VLAN-ID>
 encapsulation dot1q <VLAN-ID> native
 ip address <IP-ADDRESS> 255.255.255.0
end
!
show interfaces trunk
show vlans
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

Trunk와 Native VLAN 상세는 1-2. 🚚 Switchport & Trunk & Inter-VLAN을 참고하십시오.

---

### Q5. L3 Switch가 Router-on-a-Stick보다 나은 이유는?

👉 **핵심 답변:** L3 Switch는 라우팅 결정을 ASIC 하드웨어에서 처리하고 Inter-VLAN 트래픽이 내부 백플레인을 통과하므로, 모든 VLAN 트래픽이 단일 Trunk를 왕복하는 병목이 사라집니다.

| 비교 항목 | Router-on-a-Stick | L3 Switch (SVI) |
|---|---|---|
| 처리 주체 | 주로 CPU 소프트웨어 | ASIC 하드웨어 |
| 병목 | Router 연결 단일 Trunk | 사실상 없음 |
| 확장 | VLAN 증가 시 링크 포화 | SVI 추가로 확장 |
| WAN 기능 | 풍부(NAT, VPN 등) | 제한적 |

L3 Switch는 캠퍼스 내부 고속 Inter-VLAN Routing을 담당하고, Router는 WAN 연동과 고급 L3 서비스를 담당하는 역할 분리가 일반적입니다.

---

### Q6. `ip routing`은 왜 필수인가?

👉 **핵심 답변:** `ip routing`이 비활성이면 L3 Switch는 SVI IP를 관리용으로만 사용하고 IPv4 패킷을 다른 인터페이스로 전달하지 않습니다.

```cisco
configure terminal
 ip routing
end
!
show ip route
show running-config | include ip routing
```

```properties
ip routing 비활성
→ SVI 자신에 대한 Ping은 응답할 수 있음
→ VLAN 간 패킷 전달 없음, 정적·동적 라우팅도 미동작
→ show ip route가 비어 있거나 L2 Switch 형태 출력

ip routing 활성
→ SVI·Routed Port의 Connected 경로 등재 후 Inter-VLAN Routing 수행
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `ip routing` 활성화는 `ip default-gateway` 기반 관리 경로를 무효화할 수 있습니다. 필요한 Default Route(`ip route 0.0.0.0 0.0.0.0 <NEXT-HOP>`)를 함께 준비하십시오.

---

### Q7. SVI는 무엇인가?

👉 **핵심 답변:** SVI(Switched Virtual Interface)는 특정 VLAN에 대응하는 논리 L3 인터페이스로, 해당 VLAN의 Default Gateway이자 라우팅 진입점입니다.

```cisco
configure terminal
 vlan <VLAN-ID>
  name USERS
 exit
 interface vlan <VLAN-ID>
  ip address <IP-ADDRESS> 255.255.255.0
  no shutdown
end
!
show vlan brief
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
```

VLAN당 최대 1개이며 물리 포트와 1:1 대응하지 않습니다. L2 Switch는 관리용 SVI 1개를 쓰지만 L3 Switch는 서비스 VLAN마다 SVI를 Gateway로 운용합니다.

> SVI를 만들어도 VLAN Database에 VLAN이 자동 생성되지는 않습니다.

---

### Q8. SVI가 Up/Up이 되는 조건은 무엇인가?

👉 **핵심 답변:** ① VLAN이 VLAN Database에 존재하고 Shutdown 상태가 아니며, ② 그 VLAN에 Up 상태 Access Port가 있거나 해당 VLAN을 허용하는 Trunk가 Up이고, ③ SVI 자체가 `no shutdown`이어야 합니다.

```properties
[1] VLAN이 VLAN DB에 존재            → show vlan brief
[2] VLAN이 shutdown 상태가 아님       → show vlan id <VLAN-ID>
[3] 해당 VLAN에 Up 상태 L2 Port 존재  → show interfaces status
    (Access Port Up 또는 Allowed Trunk Up)
[4] SVI 자체가 no shutdown            → show ip interface brief
        ↓
모두 만족 → Up/Up / 하나라도 불충족 → down
```

| SVI 상태 | 대표 원인 | 확인 |
|---|---|---|
| `administratively down` | SVI에 `shutdown` | `show running-config interface vlan <VLAN-ID>` |
| `down/down` | VLAN이 VLAN DB에 없음 | `show vlan brief` |
| `up/down` | VLAN에 Up 상태 L2 Port 없음 | `show interfaces status` |
| `up/up`인데 통신 불가 | 라우팅·ACL·단말 설정 문제 | `show ip route`, `show access-lists` |

> Access Port가 없어도 해당 VLAN을 허용하는 Trunk가 Up이면 SVI는 Up될 수 있습니다. 반대로 Trunk Allowed 목록에서 VLAN이 빠지면 Trunk가 Up이어도 SVI는 Up되지 않을 수 있습니다.

---

### Q9. SVI Autostate는 무엇인가?

👉 **핵심 답변:** Autostate는 해당 VLAN에 Forwarding 가능한 L2 Port가 하나도 없으면 SVI의 Line Protocol을 자동으로 Down시키는 기본 동작으로, 사용되지 않는 VLAN 경로가 라우팅 테이블에 남는 것을 막습니다.

```cisco
interface <INTERFACE>
 switchport autostate exclude
end
!
show running-config interface <INTERFACE>
show interfaces vlan <VLAN-ID>
```

```properties
VLAN 내 Up/Forwarding L2 Port = 0
        ↓
SVI Line Protocol Down → Connected 경로 제거 및 라우팅 광고 중단

switchport autostate exclude
= 이 포트를 SVI Up 판단 근거에서 제외
= 모니터링 전용 포트만 남은 VLAN에서 SVI 유지 방지
```

> Autostate 예외 및 비활성화 문법은 플랫폼과 IOS 버전에 따라 다르므로 플랫폼별 지원 여부 확인이 필요합니다.

> Packet Tracer는 Autostate를 실장비만큼 정밀하게 재현하지 않아 Access Port가 없어도 SVI가 Up으로 표시될 수 있습니다.

---

### Q10. Routed Interface는 무엇이고 언제 쓰는가?

👉 **핵심 답변:** `no switchport`로 물리 포트를 VLAN에 속하지 않는 L3 포트로 전환한 것이며, Switch 간 P2P 라우팅 링크나 Router·방화벽 연결에 사용합니다.

```cisco
configure terminal
 ip routing
 interface <INTERFACE>
  no switchport
  ip address <IP-ADDRESS> 255.255.255.252
  no shutdown
end
!
show ip interface brief <INTERFACE>
show interfaces <INTERFACE> switchport
show ip route connected
```

| 구분 | Routed Port | SVI |
|---|---|---|
| 설정 | 물리 포트 `no switchport` | `interface vlan <VLAN-ID>` |
| VLAN 소속 | 없음 | 특정 VLAN에 종속 |
| Up 조건 | 링크 Up + `no shutdown` | VLAN DB + Up 상태 L2 Port + `no shutdown` |
| STP | 참여하지 않음 | VLAN STP 영향 받음 |
| 용도 | P2P 라우팅 링크 | VLAN Default Gateway |

```properties
[업링크 선택 판정]
업링크로 VLAN을 확장해야 한다?
 ├─ 예    → Trunk + SVI (L2 확장, STP 설계 필요)
 └─ 아니오 → Routed Port (L3 경계 상향, STP 도메인 축소)
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `no switchport`는 해당 포트의 L2 설정(access vlan, trunk, port-security)을 초기화하며 `switchport`로 되돌려도 복원되지 않습니다. 변경 전 `show running-config interface <INTERFACE>` 출력을 보관하십시오.

---

### Q11. 경로는 어떤 순서로 선택되는가?

👉 **핵심 답변:** ① Longest Prefix Match로 후보를 좁히고, ② 동일 Prefix에 서로 다른 출처가 있으면 Administrative Distance를 비교하고, ③ 같은 프로토콜 내에서는 Metric을 비교합니다.

```properties
[1] Longest Prefix Match : 더 구체적인 경로 우선 (/24 > /16)
[2] Administrative Distance : 같은 Prefix면 AD 낮은 출처
[3] Metric : 같은 프로토콜·같은 Prefix면 Metric 낮은 경로
[4] Metric까지 동일 → Equal-Cost Multi-Path

AD만 비교해서 경로를 예측하면 오판한다

예) 10.1.0.0/16 via Static(AD 1) + 10.1.1.0/24 via OSPF(AD 110)
    목적지 10.1.1.10 → /24 경로 선택 (Prefix가 더 길기 때문)
```

| 경로 출처 | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| EBGP | 20 |
| EIGRP (Internal) | 90 |
| OSPF | 110 |
| RIP | 120 |
| EIGRP (External) | 170 |
| IBGP | 200 |

```cisco
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
traceroute <IP-ADDRESS>
```

---

### Q12. MLS의 CEF 기반 하드웨어 스위칭은 무엇인가?

👉 **핵심 답변:** 제어평면이 계산한 RIB를 FIB로 정리해 하드웨어 테이블에 미리 내려두고, 이후 패킷은 CPU를 거치지 않고 ASIC이 FIB와 Adjacency Table을 조회해 전달하는 구조입니다.

```properties
[Control Plane]
Routing Protocol / Static → RIB(show ip route) → FIB(show ip cef)
→ Adjacency Table(Next-hop L2 정보)
        ↓
[Data Plane]
TCAM 프로그래밍 → 패킷 도착 시 ASIC 조회 후 즉시 전달 (CPU 미개입)
```

CPU 처리(Punt)로 넘어가는 대표 상황은 ARP 미해결로 Adjacency가 Incomplete인 목적지, TTL 만료·IP Option 같은 예외 처리, 라우팅 프로토콜·ARP·CDP 등 제어 트래픽, TCAM 자원 초과로 프로그래밍되지 못한 항목입니다.

```cisco
show ip cef <IP-ADDRESS>
show adjacency
show processes cpu sorted
```

> CPU 사용률이 지속적으로 높다면 하드웨어 전달이 실패해 소프트웨어 처리로 넘어가는 항목이 있는지 확인해야 합니다. TCAM 사용률 확인 명령은 플랫폼별 지원 여부 확인이 필요합니다.

> Packet Tracer는 `show ip cef`와 `show adjacency`를 지원하지 않거나 축약 형태로만 제공합니다.

---

### Q13. SDM Template과 플랫폼 의존성은 왜 확인해야 하는가?

👉 **핵심 답변:** 일부 Catalyst 플랫폼은 TCAM 자원 배분을 SDM Template으로 결정하므로, 라우팅 기능을 쓰려면 라우팅용 Template으로 변경하고 재부팅해야 합니다.

```cisco
show sdm prefer
!
configure terminal
 sdm prefer lanbase-routing
end
!
write memory
reload
!
show sdm prefer
```

| 구분 | 일반적인 특성 |
|---|---|
| L2 전용 Switch | `ip routing`, `no switchport` 미지원 또는 제한 |
| LAN Base 계열 | Template 변경 후 정적 라우팅만 지원되는 경우 있음 |
| IP Base / IP Services | SVI, Routed Port, 동적 라우팅 지원 |

> Template 이름과 모델별 지원 범위는 플랫폼과 IOS/IOS XE 버전에 따라 다르므로 플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer는 `sdm prefer` 옵션을 실장비보다 제한적으로 제공합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q14. L3 EtherChannel은 Inter-VLAN Routing과 어떤 관계인가?

👉 **핵심 답변:** 여러 물리 링크를 하나의 Routed Interface로 묶어 대역폭과 이중화를 확보하는 구성이며, Distribution 계층 사이 라우팅 업링크에 사용해 SVI Gateway와 조합합니다.

```cisco
configure terminal
 interface port-channel <CHANNEL-ID>
  no switchport
  ip address <IP-ADDRESS> 255.255.255.252
  no shutdown
 interface range <INTERFACE-RANGE>
  no switchport
  no ip address
  channel-group <CHANNEL-ID> mode active
  no shutdown
end
!
show etherchannel summary
show interfaces port-channel <CHANNEL-ID>
show ip route connected
```

```properties
L2 EtherChannel = Port-channel이 Trunk 또는 Access로 동작
L3 EtherChannel = Port-channel에 no switchport + IP 주소
```

- IP는 Port-channel에만 부여하고 멤버 포트에는 부여하지 않습니다.
- 멤버 포트의 Speed, Duplex, L2/L3 속성이 일치해야 Bundle됩니다.
- LACP `passive-passive`, PAgP `auto-auto`는 Bundle되지 않으며 한쪽만 `on`이면 Loop 또는 Blackhole 위험이 있습니다.

상세 내용은 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3)을 참고하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Packet Tracer는 L3 EtherChannel 지원 범위가 모델에 따라 제한적이므로 Bundle 실패가 시뮬레이터 제약일 수 있습니다.

---

### Q15. SVI와 HSRP/VRRP는 어떻게 연계하는가?

👉 **핵심 답변:** 단말은 SVI 실주소가 아니라 Virtual IP를 Gateway로 사용하고, 두 대의 L3 Switch가 같은 VLAN의 SVI에서 동일 Group을 구성해 Active·Standby 역할을 나눕니다.

```cisco
interface vlan <VLAN-ID>
 ip address <IP-ADDRESS> 255.255.255.0
 standby version 2
 standby <GROUP-ID> ip <VIRTUAL-IP>
 standby <GROUP-ID> priority 110
 standby <GROUP-ID> preempt
 no shutdown
end
!
show standby brief
show ip interface brief | include Vlan
```

VRRP를 사용하는 경우:

```cisco
interface vlan <VLAN-ID>
 vrrp <GROUP-ID> ip <VIRTUAL-IP>
 vrrp <GROUP-ID> priority 110
end
!
show vrrp brief
```

```properties
PC Gateway = Virtual IP / SVI 실주소는 장비별로 서로 다름
Active 장애 시 Standby가 Virtual IP·Virtual MAC 인수
권장 정렬 = HSRP Active = STP Root = 주 업링크 보유 장비
```

양단 SVI의 동일 Subnet·Mask, Group ID·인증 일치, 업링크 장애 대비 Tracking을 함께 확인합니다. 상세 내용은 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)를 참고하십시오.

---

### Q16. SVI 환경에서 VLAN 간 DHCP는 어떻게 동작하는가?

👉 **핵심 답변:** DHCP Discover는 Broadcast여서 VLAN 경계를 넘지 못하므로, SVI에 `ip helper-address`를 설정해 L3 Switch가 Unicast로 중계(DHCP Relay)해야 합니다.

```cisco
interface vlan <VLAN-ID>
 ip address <IP-ADDRESS> 255.255.255.0
 ip helper-address <DHCP-SERVER-IP>
 no shutdown
end
!
show ip interface vlan <VLAN-ID>
show ip route <DHCP-SERVER-IP>
```

```properties
PC → Discover(Broadcast, VLAN 내부)
→ SVI 수신 후 helper-address 서버로 Unicast 전달
→ giaddr = SVI IP (서버가 올바른 Pool 선택)
→ Offer/Ack 역방향 중계
```

- SVI가 Down이면 Relay도 동작하지 않습니다.
- 서버까지 왕복 경로가 라우팅 테이블에 존재해야 합니다.
- 서버 Pool Subnet이 SVI Subnet과 일치해야 giaddr 기반 할당이 성공합니다.
- 중계 대상 UDP 포트는 `ip forward-protocol udp` 설정에 영향을 받습니다.

상세 내용은 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

## 2. 🛠️ 표준 설정 템플릿

### 2-1. 기본 준비와 SVI Gateway

```cisco
configure terminal
 ip routing
 vlan 10
  name USERS
 exit
 vlan 20
  name SERVERS
 exit
 interface vlan 10
  description GW-VLAN10
  ip address 192.168.10.1 255.255.255.0
  ip helper-address 10.10.10.10
  no shutdown
 interface vlan 20
  description GW-VLAN20
  ip address 192.168.20.1 255.255.255.0
  no shutdown
 interface range GigabitEthernet1/0/1 - 12
  switchport mode access
  switchport access vlan 10
  no shutdown
end
!
show vlan brief
show ip interface brief | include Vlan
show interfaces vlan 10
show interfaces status
show ip route connected
```

---

### 2-2. Routed Port 업링크와 정적 경로

```cisco
configure terminal
 interface GigabitEthernet1/0/24
  description P2P-TO-CORE
  no switchport
  ip address 10.0.0.1 255.255.255.252
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.0.0.2
end
!
show ip interface brief GigabitEthernet1/0/24
show cdp neighbors detail
show ip route static
show ip route connected
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 2-3. Router-on-a-Stick

```cisco
configure terminal
 interface GigabitEthernet0/24
  switchport trunk encapsulation dot1q
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  switchport trunk native vlan 999
end
!
show interfaces trunk
```

```cisco
configure terminal
 interface GigabitEthernet0/0
  no ip address
  no shutdown
 interface GigabitEthernet0/0.10
  encapsulation dot1q 10
  ip address 192.168.10.254 255.255.255.0
 interface GigabitEthernet0/0.20
  encapsulation dot1q 20
  ip address 192.168.20.254 255.255.255.0
end
!
show vlans
show ip route connected
```

---

### 2-4. 동적 라우팅 연동

```cisco
configure terminal
 router ospf 1
  router-id 1.1.1.1
  passive-interface default
  no passive-interface GigabitEthernet1/0/24
  network 10.0.0.0 0.0.0.3 area 0
  network 192.168.10.0 0.0.0.255 area 0
end
!
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
```

```properties
passive-interface default          = 모든 IF에서 Hello 차단
no passive-interface <INTERFACE>   = 인접 맺을 IF만 개방
효과 = 단말 전용 VLAN에 불필요한 Hello 미전송, 공격 표면·CPU 절감
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 3. 🛡️ 안전한 변경 절차

```properties
[1] 현재 확인 : show running-config / show ip route / show ip interface brief
[2] 백업     : 설정 파일 외부 보관
[3] 경로 확인 : 변경 대상이 관리 경로에 포함되는지 확인
[4] 적용     : Maintenance Window에서 단계적 적용
[5] 검증     : show 명령과 ping/traceroute로 확인
[6] 저장     : write memory
```

```cisco
show running-config
show ip route
show ip interface brief
show vlan brief
show interfaces trunk
show standby brief
```

Rollback 준비 및 검증:

```cisco
copy running-config startup-config
reload in 10
!
reload cancel
write memory
!
show ip route
show ip interface brief | include Vlan
ping <IP-ADDRESS>
traceroute <IP-ADDRESS>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `copy startup-config running-config`는 병합 동작이므로 삭제된 항목이 복원되지 않습니다. 완전한 복구는 변경 전 설정을 기준으로 개별 항목을 되돌려야 합니다.

---

## 4. 🔍 검증 명령어

```cisco
show ip route
show ip route <IP-ADDRESS>
show ip route connected
show ip protocols
show ip cef <IP-ADDRESS>
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
show interfaces <INTERFACE> switchport
```

```cisco
show vlan brief
show vlan id <VLAN-ID>
show interfaces status
show interfaces trunk
show mac address-table vlan <VLAN-ID>
show spanning-tree vlan <VLAN-ID>
```

```cisco
show standby brief
show vrrp brief
show etherchannel summary
show access-lists
show sdm prefer
show processes cpu sorted
```

---

## 5. ⚠️ 실무 체크리스트

- [ ] `ip routing`이 활성화되어 있는가?
- [ ] 필요한 VLAN이 VLAN Database에 존재하는가?
- [ ] 각 VLAN에 Up 상태 L2 Port 또는 Allowed Trunk가 있는가?
- [ ] SVI가 `no shutdown` 상태이며 Up/Up인가?
- [ ] 단말 IP·Mask·Gateway가 SVI 또는 Virtual IP와 일치하는가?
- [ ] Trunk Allowed VLAN에 서비스 VLAN이 모두 포함되었는가?
- [ ] 양단 Native VLAN이 일치하는가?
- [ ] `no switchport` 대상 포트의 기존 L2 설정을 백업했는가?
- [ ] 원격 네트워크가 정적 또는 동적 라우팅으로 학습되었는가?
- [ ] Longest Prefix Match 기준으로 경로를 검증했는가?
- [ ] ACL이 Inter-VLAN 트래픽을 차단하고 있지 않은가?
- [ ] HSRP/VRRP Group과 Virtual IP가 양단에서 일치하는가?
- [ ] `ip helper-address`와 DHCP 서버 왕복 경로가 확보되었는가?
- [ ] SDM Template이 라우팅을 지원하는 값인가?

---

> 📌 **핵심 요약**
> - Inter-VLAN Routing은 Router-on-a-Stick, SVI, Routed Port로 구현한다.
> - L3 Switch 라우팅의 전제 조건은 전역 `ip routing` 활성화다.
> - SVI Up/Up 조건은 VLAN DB 존재 + Up 상태 L2 Port + `no shutdown`이다.
> - Autostate는 VLAN에 활성 Port가 없으면 SVI를 Down시킨다.
> - `no switchport`는 물리 포트를 VLAN에 속하지 않는 L3 포트로 만든다.
> - Routed Port는 P2P 라우팅, SVI는 VLAN Gateway에 사용한다.
> - Native VLAN은 Untagged이며 양단 불일치는 VLAN Leaking을 유발한다.
> - 경로 선택은 Longest Prefix Match → AD → Metric 순서다.
> - MLS는 RIB를 FIB로 내려 CEF 기반 하드웨어 전달을 수행한다.
> - CPU 사용률 급증은 하드웨어 전달 실패 신호일 수 있다.
> - SVI에 HSRP/VRRP를 구성해 Gateway 이중화를 확보한다.
> - VLAN 간 DHCP는 SVI의 `ip helper-address` 중계가 필요하다.
> - Routed Port 지원 여부와 SDM Template은 플랫폼마다 다르다.
> - VLAN은 Broadcast Domain 분리이며 완전한 보안 경계가 아니다.
> - 관련: 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing · 4-2. 🧩 L3 Switch & Inter-VLAN Routing 통합 정리 · 4-3. 🚑 L3 Switch & Inter-VLAN Routing 트러블슈팅 치트시트 · 4-4. ⚡ L3 Switch & Inter-VLAN Routing 명령어 퀵 레퍼런스 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)
