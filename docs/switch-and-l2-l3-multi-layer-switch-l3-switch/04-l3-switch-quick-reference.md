# L3 Switch & Inter-VLAN Routing 명령어 퀵 레퍼런스

> **Tag:** #Network #Switch #L3Switch #InterVLAN #SVI #Cisco  
> **핵심 요약:** `ip routing`, SVI, Routed Port, Router-on-a-Stick, L3 EtherChannel 설정과 검증 명령을 빠르게 확인하기 위한 암기 카드. 모든 설정 뒤에는 검증 `show` 명령을 붙이고, SVI는 VLAN DB·Up 상태 L2 Port·`no shutdown` 세 조건을 항상 함께 점검한다.

---

## 1. 요구사항별 선택

| 요구사항 | 사용 기능 |
|---|---|
| L3 Switch 라우팅 활성화 | `ip routing` |
| VLAN Default Gateway | SVI (`interface vlan <VLAN-ID>`) |
| Switch 간 P2P 라우팅 링크 | Routed Port (`no switchport`) |
| Router만으로 VLAN 간 라우팅 | Router-on-a-Stick (Sub-interface) |
| 업링크 대역폭·이중화 | L3 EtherChannel |
| Gateway 이중화 | SVI + HSRP 또는 VRRP |
| VLAN 간 DHCP 할당 | `ip helper-address` |
| VLAN 간 트래픽 제어 | ACL |
| 라우팅용 TCAM 자원 확보 | SDM Template |

---

## 2. 방식 비교

| 구분 | Router-on-a-Stick | SVI | Routed Port |
|---|---|---|---|
| 위치 | Router Sub-interface | L3 Switch 논리 IF | L3 Switch 물리 포트 |
| 핵심 명령 | `encapsulation dot1q <VLAN-ID>` | `interface vlan <VLAN-ID>` | `no switchport` |
| VLAN 소속 | Sub-interface가 VLAN 대응 | 특정 VLAN에 종속 | 없음 |
| 처리 주체 | 주로 CPU | ASIC | ASIC |
| Up 조건 | 물리 IF Up + Trunk 정상 | VLAN DB + Up 상태 L2 Port + `no shutdown` | 링크 Up + `no shutdown` |
| STP | Switch 측 Trunk가 참여 | VLAN STP 영향 받음 | 참여하지 않음 |
| 용도 | 소규모 VLAN 라우팅 | VLAN Gateway | P2P 라우팅 업링크 |
| 병목 | 단일 Trunk 링크 | 사실상 없음 | 사실상 없음 |

---

## 3. 라우팅 활성화

```cisco
configure terminal
 ip routing
end
!
show ip route
show running-config | include ip routing
```

```properties
ip routing 없음 = SVI IP는 존재하나 VLAN 간 전달 없음
                 show ip route가 비거나 L2 Switch 형태 출력
ip routing 있음 = Connected 경로 등재, 정적·동적 라우팅 동작
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `ip routing` 활성화는 `ip default-gateway` 기반 관리 경로를 무효화할 수 있으므로 Default Route를 함께 준비합니다.

---

## 4. SVI 생성과 Access Port 할당

```cisco
configure terminal
 vlan <VLAN-ID>
  name USERS
 exit
 interface vlan <VLAN-ID>
  description GW-VLAN
  ip address <IP-ADDRESS> 255.255.255.0
  no shutdown
 interface range <INTERFACE-RANGE>
  switchport mode access
  switchport access vlan <VLAN-ID>
  no shutdown
end
!
show vlan brief
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
show interfaces status
show ip route connected
```

SVI 삭제:

```cisco
configure terminal
 no interface vlan <VLAN-ID>
end
!
show ip interface brief | include Vlan
```

```properties
[SVI Up/Up 조건]
VLAN이 VLAN DB에 존재
+ VLAN이 shutdown 상태가 아님
+ 해당 VLAN에 Up 상태 Access Port 또는 Allowed Trunk 존재
+ SVI 자체가 no shutdown

[단말 전제]
Access Port VLAN = SVI VLAN
단말 IP/Mask     = SVI Subnet과 일치
단말 Gateway     = SVI IP 또는 Virtual IP
```

> SVI를 만들어도 VLAN Database에 VLAN이 자동 생성되지는 않습니다.

---

## 5. Routed Interface 전환과 복귀

```cisco
configure terminal
 interface <INTERFACE>
  description P2P-UPLINK
  no switchport
  ip address <IP-ADDRESS> 255.255.255.252
  no shutdown
end
!
show ip interface brief <INTERFACE>
show interfaces <INTERFACE> switchport
show ip route connected
ping <PEER-IP>
```

L2로 복귀:

```cisco
configure terminal
 interface <INTERFACE>
  no ip address
  switchport
  switchport mode access
  switchport access vlan <VLAN-ID>
  no shutdown
end
!
show interfaces <INTERFACE> switchport
show vlan brief
```

```properties
no switchport 효과
VLAN 소속 해제 / switchport 설정 삭제 / STP 대상 제외 / 포트가 IP 보유
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `switchport`로 되돌려도 이전 L2 설정은 복원되지 않으므로 변경 전 `show running-config interface <INTERFACE>`를 보관합니다.

---

## 6. SVI Autostate

```cisco
configure terminal
 interface <INTERFACE>
  switchport autostate exclude
end
!
show running-config interface <INTERFACE>
show interfaces vlan <VLAN-ID>
```

```properties
Autostate 기본 동작
VLAN 내 Forwarding 가능한 L2 Port = 0 → SVI Line Protocol Down
→ Connected 경로 제거 및 광고 중단

switchport autostate exclude = 이 포트를 SVI Up 판단에서 제외
```

> Autostate 예외 및 비활성화 문법은 플랫폼별 지원 여부 확인이 필요합니다.

> Packet Tracer는 Autostate를 실장비와 동일하게 재현하지 않아 Access Port가 없어도 SVI가 Up으로 표시될 수 있습니다.

---

## 7. Router-on-a-Stick

Switch 측 Trunk:

```cisco
configure terminal
 interface <INTERFACE>
  description TRUNK-TO-ROUTER
  switchport trunk encapsulation dot1q
  switchport mode trunk
  switchport trunk allowed vlan <VLAN-LIST>
  switchport trunk native vlan <VLAN-ID>
end
!
show interfaces trunk
show interfaces <INTERFACE> switchport
```

Router 측 Sub-interface:

```cisco
configure terminal
 interface <INTERFACE>
  no ip address
  no shutdown
 interface <INTERFACE>.<VLAN-ID>
  encapsulation dot1q <VLAN-ID>
  ip address <IP-ADDRESS> 255.255.255.0
end
!
show vlans
show ip interface brief
show ip route connected
```

Native VLAN 수용:

```cisco
configure terminal
 interface <INTERFACE>.<VLAN-ID>
  encapsulation dot1q <VLAN-ID> native
  ip address <IP-ADDRESS> 255.255.255.0
end
!
show vlans
show interfaces trunk
```

```properties
802.1Q Trunk
Native VLAN = Untagged / 그 외 = Tagged
양단 Native VLAN 불일치 = VLAN Leaking (Broadcast Domain 병합)
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 8. L3 EtherChannel

```cisco
configure terminal
 interface port-channel <CHANNEL-ID>
  description L3-UPLINK
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
show ip interface brief | include Port-channel
show ip route connected
```

| 프로토콜 | 모드 조합 | Bundle |
|---|---|---|
| LACP | `active` ↔ `active` / `active` ↔ `passive` | O |
| LACP | `passive` ↔ `passive` | X |
| PAgP | `desirable` ↔ `desirable` / `desirable` ↔ `auto` | O |
| PAgP | `auto` ↔ `auto` | X |
| Static | `on` ↔ `on` | O (협상 없음) |

```properties
IP는 Port-channel에만 부여, 멤버는 no switchport + no ip address
Speed/Duplex/MTU 등 속성 일치 필수
한쪽만 on 모드면 Loop 또는 Blackhole 위험
```

상세 내용은 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3)을 참고하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Packet Tracer는 L3 EtherChannel 지원 범위가 모델에 따라 제한적입니다.

---

## 9. 정적 라우팅

```cisco
configure terminal
 ip route <DEST-NETWORK> <MASK> <NEXT-HOP>
 ip route 0.0.0.0 0.0.0.0 <NEXT-HOP>
 ip route <DEST-NETWORK> <MASK> <NEXT-HOP> 200
end
!
show ip route static
show ip route <IP-ADDRESS>
```

```properties
마지막 인자 200 = Administrative Distance 지정 (Floating Static Route)
주 경로 소실 시에만 라우팅 테이블에 등재
```

삭제:

```cisco
configure terminal
 no ip route <DEST-NETWORK> <MASK> <NEXT-HOP>
end
!
show ip route static
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 10. 동적 라우팅

```cisco
configure terminal
 router ospf 1
  router-id <ROUTER-ID>
  passive-interface default
  no passive-interface <INTERFACE>
  network <NETWORK> <WILDCARD> area 0
end
!
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
```

```cisco
configure terminal
 router eigrp 100
  no auto-summary
  network <NETWORK> <WILDCARD>
end
!
show ip eigrp neighbors
show ip eigrp interfaces
show ip route eigrp
```

```properties
passive-interface default        = 모든 IF에서 Hello 차단
no passive-interface <INTERFACE> = 인접 맺을 IF만 개방
효과 = 단말 전용 VLAN에 Hello 미전송, 공격 표면·CPU 절감
```

---

## 11. 경로 선택 순서

```properties
[1] Longest Prefix Match : 더 구체적인 Prefix 우선 (/24 > /16)
[2] Administrative Distance : 같은 Prefix면 AD 낮은 출처
[3] Metric : 같은 프로토콜·같은 Prefix면 Metric 낮은 경로
[4] Metric 동일 → Equal-Cost Multi-Path
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

```properties
10.1.0.0/16 via Static (AD 1)
10.1.1.0/24 via OSPF   (AD 110)
목적지 10.1.1.10 → /24 경로 선택 (AD 단독 비교 금지)
```

```cisco
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
traceroute <IP-ADDRESS>
```

---

## 12. SVI + HSRP / VRRP

```cisco
configure terminal
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
show standby vlan <VLAN-ID>
```

```cisco
configure terminal
 interface vlan <VLAN-ID>
  vrrp <GROUP-ID> ip <VIRTUAL-IP>
  vrrp <GROUP-ID> priority 110
end
!
show vrrp brief
```

```properties
단말 Gateway = Virtual IP / SVI 실주소는 장비별로 다름
권장 정렬 = HSRP Active = STP Root = 주 업링크 보유 장비
```

상세 내용은 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)를 참고하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 13. DHCP Relay

```cisco
configure terminal
 interface vlan <VLAN-ID>
  ip address <IP-ADDRESS> 255.255.255.0
  ip helper-address <DHCP-SERVER-IP>
  ip helper-address <DHCP-SERVER-IP-2>
  no shutdown
end
!
show ip interface vlan <VLAN-ID>
show running-config interface vlan <VLAN-ID>
show ip route <DHCP-SERVER-IP>
```

```properties
Discover = Broadcast (VLAN 경계 통과 불가)
→ SVI가 수신 후 Unicast 중계
→ giaddr = SVI IP (서버가 올바른 Pool 선택)
→ 왕복 경로가 없으면 Offer/Ack 실패
```

상세 내용은 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

## 14. SDM Template과 CEF 확인

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
show ip route
```

```cisco
show ip cef <IP-ADDRESS>
show adjacency
show arp
show processes cpu sorted
```

```properties
[Control Plane] Routing Protocol/Static → RIB → FIB → Adjacency
[Data Plane]    TCAM 프로그래밍 → ASIC 하드웨어 전달

CPU로 넘어가는 상황
ARP 미해결(Adjacency Incomplete) / TTL 만료·IP Option
제어 트래픽(라우팅, ARP, CDP) / TCAM 자원 초과
```

> Template 이름, 지원 범위, TCAM 확인 명령은 플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer는 `sdm prefer`, `show ip cef`, `show adjacency`를 지원하지 않거나 축약 형태로만 제공합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 15. 검증 명령어

```cisco
show ip route
show ip route <IP-ADDRESS>
show ip route connected
show ip protocols
```

```cisco
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
show ip interface vlan <VLAN-ID>
show interfaces <INTERFACE> switchport
show interfaces status
```

```cisco
show vlan brief
show vlan id <VLAN-ID>
show interfaces trunk
show mac address-table vlan <VLAN-ID>
show cdp neighbors detail
```

```cisco
show standby brief
show vrrp brief
show etherchannel summary
show spanning-tree vlan <VLAN-ID>
show access-lists
show sdm prefer
show version
show logging
```

---

## 16. 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| SVI에 IP만 설정 | `ip routing` 없으면 VLAN 간 라우팅 안 됨 |
| `interface vlan <VLAN-ID>` 생성 | VLAN Database에 VLAN이 자동 생성되지 않음 |
| SVI `no shutdown` | Up 상태 L2 Port가 없으면 `up/down` |
| SVI `up/up` | 라우팅·ACL·단말 설정 정상을 의미하지 않음 |
| Autostate Down | 장애가 아니라 정상 동작일 수 있음 |
| `no switchport` | 기존 L2 설정이 삭제되며 복원되지 않음 |
| `switchport` 복귀 | 이전 Access·Trunk 설정이 되돌아오지 않음 |
| Routed Port | STP에 참여하지 않음 |
| Native VLAN | Untagged이며 양단 불일치는 VLAN Leaking |
| `switchport trunk allowed vlan <LIST>` | 기존 목록을 덮어씀 (`add`/`remove` 사용) |
| Trunk Up | Allowed 목록에 없으면 해당 VLAN은 통과 못 함 |
| AD가 낮은 경로 | Prefix가 짧으면 선택되지 않을 수 있음 |
| Longest Prefix Match | AD·Metric보다 먼저 적용되는 1순위 기준 |
| VLAN 분리 | Broadcast Domain 분리이며 보안 경계가 아님 |
| Gateway Ping 성공 | 원격 대역 라우팅 정상을 의미하지 않음 |
| 단방향만 실패 | Return Path 또는 ACL 방향 문제 가능성 |
| `ip helper-address` | SVI가 Down이면 Relay도 동작하지 않음 |
| HSRP 구성 | 단말 Gateway가 Virtual IP여야 의미 있음 |
| L3 EtherChannel | 멤버 포트에 IP를 주면 Bundle 실패 |
| `sdm prefer` 입력 | 재부팅 전까지 적용되지 않음 |
| CPU 사용률 급증 | 하드웨어 전달 실패로 CPU 처리 중일 가능성 |
| Packet Tracer 정상 동작 | 실장비 동작을 보장하지 않음 |

---

## 17. 빠른 장애 진단 순서

```properties
단말 IP / Mask / Gateway 확인
    ↓
Access Port VLAN 할당 확인
    ↓
VLAN Database 존재 및 Active 확인
    ↓
Trunk Up 및 Allowed VLAN 확인
    ↓
Native VLAN 일치 확인
    ↓
SVI Up/Up 확인 (VLAN DB + Up 상태 L2 Port + no shutdown)
    ↓
ip routing 활성 확인
    ↓
라우팅 테이블 경로 확인 (Longest Prefix Match)
    ↓
Return Path 확인
    ↓
ACL 및 보안 정책 확인
    ↓
STP Forwarding 확인
    ↓
HSRP/VRRP Active 확인
    ↓
CPU 및 CEF 상태 확인
```

빠른 확인 3종:

```cisco
show ip interface brief | include Vlan
show ip route
show interfaces trunk
```

---

## 18. 안전한 변경 원칙

```properties
[1] 현재 확인 → [2] 백업 → [3] 경로 확인(관리 경로 포함 여부)
→ [4] 적용 → [5] 검증 → [6] 저장
```

변경 전:

```cisco
show running-config
show ip route
show ip interface brief
show vlan brief
show interfaces trunk
```

Rollback 준비 및 저장:

```cisco
reload in 10
!
reload cancel
!
write memory
```

변경 후:

```cisco
show ip route
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
show interfaces trunk
show standby brief
ping <IP-ADDRESS>
traceroute <IP-ADDRESS>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

> 📌 **핵심 요약**
> - 라우팅 활성화: `ip routing`
> - SVI 생성: `interface vlan <VLAN-ID>` + `ip address` + `no shutdown`
> - SVI Up 조건: VLAN DB 존재 + Up 상태 L2 Port + `no shutdown`
> - Routed Port: `no switchport` + `ip address`
> - Autostate 예외: `switchport autostate exclude`
> - Router-on-a-Stick: `encapsulation dot1q <VLAN-ID>`
> - Native VLAN 수용: `encapsulation dot1q <VLAN-ID> native`
> - L3 EtherChannel: Port-channel에 `no switchport` + IP, 멤버는 `channel-group`
> - DHCP Relay: `ip helper-address <DHCP-SERVER-IP>`
> - Gateway 이중화: `standby <GROUP-ID> ip <VIRTUAL-IP>` 또는 `vrrp <GROUP-ID> ip <VIRTUAL-IP>`
> - 경로 선택: Longest Prefix Match → AD → Metric
> - 상태 확인: `show ip route`, `show ip interface brief | include Vlan`
> - Trunk 확인: `show interfaces trunk`
> - 플랫폼 확인: `show sdm prefer`, `show version`
> - 관련: 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing · 4-2. 🧩 L3 Switch & Inter-VLAN Routing 통합 정리 · 4-3. 🚑 L3 Switch & Inter-VLAN Routing 트러블슈팅 치트시트 · 4-4. ⚡ L3 Switch & Inter-VLAN Routing 명령어 퀵 레퍼런스 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)
