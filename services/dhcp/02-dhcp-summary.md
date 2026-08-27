# 🧩 DHCP 통합 정리

> **Tag:** #Network #DHCP #IPServices #Relay #Cisco  
> **핵심 요약:** DHCP는 UDP 67(Server)/68(Client)를 사용해 IP·Subnet Mask·Gateway·DNS를 단말에 자동 할당하는 응용 계층 프로토콜이다. Client는 IP가 없는 상태에서 시작하므로 Discover와 Request를 Broadcast로 보내며, Server가 다른 Subnet에 있으면 Gateway 장비의 `ip helper-address`가 Broadcast를 Unicast로 변환하고 giaddr에 수신 Interface IP를 기록해 Server가 어느 Subnet의 요청인지 판별하게 한다. 운영 설계의 핵심은 Gateway·Server·고정 IP 장비를 `ip dhcp excluded-address`로 제외하고 Pool의 `network`와 `default-router`를 실제 Segment와 일치시키는 것이다.

---

## 1. 🎯 핵심 기술 개념

### Q1. DHCP는 무엇인가?

👉 **핵심 답변:** DHCP는 단말이 부팅 시 IP Address·Subnet Mask·Default Gateway·DNS Server 등의 IP 파라미터를 Server로부터 자동으로 임대(Lease)받는 Client/Server 방식의 프로토콜입니다.

```properties
DHCP가 배포하는 대표 정보
= IP Address / Subnet Mask
= Default Gateway (default-router)
= DNS Server / Domain Name
= Lease Time
= 옵션 정보 (TFTP, NTP, Option 82 등)
```

| 구분 | Port | 설명 |
|---|:---:|---|
| DHCP Server | UDP 67 | Client 요청 수신, Offer/ACK 송신 |
| DHCP Client | UDP 68 | Discover/Request 송신, Offer/ACK 수신 |

> DHCP는 응용 계층 프로토콜이지만 초기 동작이 Broadcast 기반이므로 L2 Broadcast Domain 범위와 L3 Gateway 설정에 직접 영향을 받습니다. 계층을 혼동하면 진단 순서를 잘못 잡게 됩니다.

---

### Q2. DORA 4단계는 어떻게 동작하는가?

👉 **핵심 답변:** Discover(Server 탐색) → Offer(IP 제안) → Request(선택 공표) → ACK(최종 확정) 순서로 진행되며, Client는 자기 IP가 없으므로 Discover와 Request를 Broadcast로 전송합니다.

```properties
[1] Discover : Client → Broadcast (Src 0.0.0.0 / Dst 255.255.255.255)
        ↓      "이 Segment에 DHCP Server가 있습니까?"
[2] Offer    : Server → Client
        ↓      "이 IP를 사용하시겠습니까?" (중복 검사 후 제안)
[3] Request  : Client → Broadcast
        ↓      "제안 중 특정 Server의 IP를 선택합니다."
[4] ACK      : Server → Client
               "해당 IP를 Lease Time 동안 확정합니다."
```

| 단계 | 방향 | Src IP | 전송 방식 | 역할 |
|---|---|---|---|---|
| **D**iscover | Client → Server | 0.0.0.0 | Broadcast | Server 탐색 |
| **O**ffer | Server → Client | Server IP | Broadcast 또는 Unicast | IP 제안 |
| **R**equest | Client → Server | 0.0.0.0 | Broadcast | 선택 Server 공표 |
| **A**CK | Server → Client | Server IP | Broadcast 또는 Unicast | 최종 확정 |

> Offer와 ACK가 Broadcast인지 Unicast인지는 Client가 설정한 Broadcast Flag와 구현에 따라 달라집니다. Client는 아직 IP가 확정되지 않아 ARP 응답을 못 할 수 있으므로 Broadcast로 회신하는 경우가 흔합니다.

---

### Q3. Discover와 Request는 왜 둘 다 Broadcast인가?

👉 **핵심 답변:** Discover는 Server 위치를 모르기 때문이고, Request는 여러 Server가 Offer를 보낸 상황에서 어떤 제안을 선택했는지 나머지 Server에게도 알려 미선택 Offer를 회수시키기 위함입니다.

```properties
Server A Offer + Server B Offer 수신
        ↓
Client가 하나를 선택
        ↓
Request를 Broadcast로 전송
        ↓
선택되지 않은 Server가 예약 IP를 Pool로 반환
```

Server는 Offer 전에 해당 IP가 이미 사용 중인지 ICMP Ping 또는 ARP로 확인할 수 있으며, 응답이 오면 Conflict로 기록합니다.

---

### Q4. Server가 다른 Subnet에 있으면 어떻게 동작하는가?

👉 **핵심 답변:** DHCP Discover는 Broadcast이므로 Router를 넘지 못합니다. 단말 쪽 Gateway 장비에 `ip helper-address`를 설정해 DHCP Relay Agent로 동작시키면, Relay Agent가 Broadcast를 원격 Server로 Unicast 변환해 전달합니다.

```properties
Client Broadcast (Dst 255.255.255.255)
        ↓
Gateway Interface 수신 (ip helper-address 설정됨)
        ↓
Relay Agent가 Unicast로 재작성
   Src    = 수신 Interface IP
   Dst    = helper-address로 지정한 Server IP
   giaddr = 수신 Interface IP 기록
        ↓
Routing을 통해 원격 DHCP Server 도달
        ↓
Server는 giaddr로 어느 Subnet 요청인지 판별
        ↓
해당 Subnet Pool에서 IP 선택
        ↓
응답을 giaddr(Relay Agent)로 Unicast 회신
        ↓
Relay Agent가 Client Segment로 전달
```

```properties
Client → Relay  = Broadcast
Relay  → Server = Unicast
Server → Relay  = Unicast (Dst = giaddr)
Relay  → Client = Broadcast 또는 Unicast
```

> giaddr가 비어 있으면 Server는 요청이 자신과 같은 Segment에서 온 것으로 판단합니다. giaddr가 채워져 있으면 그 값이 속한 Subnet의 Pool을 선택하므로, Relay Interface IP와 Pool의 `network` 대역이 반드시 일치해야 합니다.

Gateway 구조는 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing와 함께 이해하는 것이 좋습니다.

---

### Q5. `ip helper-address`는 DHCP만 전달하는가?

👉 **핵심 답변:** 아닙니다. `ip helper-address`는 기본적으로 여러 UDP Broadcast 서비스를 함께 전달하며, 필요한 Port만 남기려면 `ip forward-protocol udp`로 조정해야 합니다.

| Port | 서비스 | Port | 서비스 |
|:---:|---|:---:|---|
| 37 | Time | 68 | BOOTP/DHCP Client |
| 49 | TACACS | 69 | TFTP |
| 53 | DNS | 137 | NetBIOS Name Service |
| 67 | BOOTP/DHCP Server | 138 | NetBIOS Datagram Service |

```cisco
configure terminal
!
no ip forward-protocol udp 37
no ip forward-protocol udp 49
no ip forward-protocol udp 53
no ip forward-protocol udp 69
no ip forward-protocol udp 137
no ip forward-protocol udp 138
!
ip forward-protocol udp 67
end
!
show running-config | include forward-protocol
show ip interface <INTERFACE> | include Helper
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> 기본 Forwarding 대상 목록과 `no ip forward-protocol udp` 동작은 IOS/IOS XE 버전에 따라 차이가 있으므로 플랫폼별 지원 여부 확인이 필요합니다.

---

### Q6. Lease 갱신은 언제 일어나는가? (T1/T2)

👉 **핵심 답변:** Client는 Lease Time의 50%(T1)에 기존 Server로 Unicast 갱신을 시도하고, 실패하면 87.5%(T2)에 Broadcast로 임의 Server에게 갱신을 시도합니다. 그래도 실패해 Lease가 만료되면 IP를 반납하고 DORA를 처음부터 다시 수행합니다.

```properties
T1 = 50%   → Renewing  : 원 Server로 Unicast Request
        ↓ 실패
T2 = 87.5% → Rebinding : Broadcast Request (임의 Server 허용)
        ↓ 실패
100%       → Lease 만료 : IP 반납 후 Discover부터 재시작
```

| 환경 | 권장 Lease | 이유 |
|---|---|---|
| Guest·무선 회전율 높은 망 | 짧게 (수 시간) | IP 회수 속도 확보 |
| 일반 사무 사용자망 | 중간 (1~7일) | 안정성과 회수의 균형 |
| Printer·서버성 장비 | 길게 또는 Static Binding | IP 고정성 확보 |

> `lease infinite`는 IP를 사실상 고정하지만 Pool 주소가 회수되지 않습니다. 고정이 필요한 장비는 Static Binding 또는 장비 자체 Static IP가 관리 측면에서 더 명확합니다.

---

### Q7. `ip dhcp excluded-address`는 왜 반드시 필요한가?

👉 **핵심 답변:** Pool의 `network` 명령은 해당 Subnet 전체를 임대 후보로 삼기 때문에, Gateway IP·DHCP Server 자신의 IP·Static IP 장비 주소를 명시적으로 제외하지 않으면 이미 사용 중인 IP를 임대해 중복 주소 장애가 발생합니다.

```properties
network 192.168.10.0 255.255.255.0
= .1 ~ .254 전체가 임대 후보

excluded-address 미설정
→ Gateway(.254)나 Server(.252)도 임대 후보
→ 중복 주소 및 Segment 전체 통신 장애
```

제외해야 하는 대표 항목은 Default Gateway(Router Interface 또는 SVI IP), DHCP Server 자신의 IP, DNS·NTP·인증 Server, Printer·CCTV·AP 등 Static IP 장비, HSRP/VRRP Virtual IP와 각 Router의 실 IP, 향후 확장을 위한 예약 대역입니다.

> 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy) 환경에서는 Virtual IP와 각 Router의 실제 Interface IP를 모두 제외해야 합니다.

---

### Q8. Static Binding은 어떻게 구성하는가?

👉 **핵심 답변:** 특정 단말에 항상 같은 IP를 주려면 Pool 안에서 `host`로 IP를 지정하고 `hardware-address` 또는 `client-identifier`로 단말을 식별합니다.

```properties
hardware-address  = MAC 기반 식별
client-identifier = Client Identifier(Option 61) 기반 식별
                    Ethernet은 일반적으로 01 + MAC 형식
```

```cisco
configure terminal
!
ip dhcp pool PRINTER_01
 host 192.168.10.50 255.255.255.0
 hardware-address 00aa.bbcc.ddee
 default-router 192.168.10.254
 dns-server 192.168.10.252
 domain-name git.local
end
!
show ip dhcp binding
show ip dhcp pool
show running-config | section ip dhcp pool
```

> `hardware-address`와 `client-identifier`는 단말의 요청 방식에 따라 매칭 결과가 달라집니다. Windows·Linux·Embedded 장비의 Client Identifier 생성 방식이 다르므로, 매칭이 되지 않으면 `debug ip dhcp server events`로 실제 수신 값을 확인하는 방법을 검토합니다.

---

### Q9. L3 Switch에서 DHCP는 어떻게 구성하는가?

👉 **핵심 답변:** L3 Switch는 SVI를 Gateway로 사용하므로 SVI가 Up이어야 하며, DHCP Server 역할을 직접 수행하거나 SVI에 `ip helper-address`를 설정해 Relay Agent로 동작시킵니다.

```properties
[Server 역할] ip routing + SVI Up + ip dhcp pool
              → Switch 자체가 해당 VLAN에 IP 임대

[Relay 역할]  ip routing + SVI Up + ip helper-address
              → 중앙 DHCP Server로 요청 전달
```

SVI가 Up이려면 해당 VLAN이 VLAN Database에 Active 상태로 존재하고, 그 VLAN에 Up 상태의 Access Port 또는 Trunk 허용 VLAN이 있으며, SVI가 `no shutdown` 상태여야 합니다.

```cisco
show ip interface brief
show interfaces vlan <VLAN-ID>
show vlan brief
show ip route
```

> VLAN에 활성 Port가 없어 SVI가 Down이면 해당 Subnet의 Pool도 동작하지 않습니다. VLAN·Trunk 관계는 1-2. 🚚 Switchport & Trunk & Inter-VLAN에서 함께 확인하십시오.

---

### Q10. 다중 VLAN 환경은 어떻게 설계하는가?

👉 **핵심 답변:** VLAN별로 별도 Pool을 만들고 각 Pool의 `network`와 `default-router`를 해당 VLAN의 SVI 대역과 일치시키며, 중앙 Server 방식이면 각 SVI에 `ip helper-address`를 설정합니다.

| 항목 | 분산형 (L3 Switch가 Server) | 중앙 집중형 (Relay + 중앙 Server) |
|---|---|---|
| 설정 위치 | 각 L3 장비 | 중앙 Server + 각 SVI Relay |
| 관리 일관성 | 낮음 | 높음 |
| Server 장애 영향 | 국소적 | 광범위 |
| IP 현황 파악 | 장비별 분산 | 중앙 통합 |
| Routing 의존성 | 낮음 | 높음 |
| 적합 규모 | 소규모 | 중·대규모 |

```properties
VLAN 10 → 192.168.10.0/24 → SVI 192.168.10.254 → Pool VLAN10
VLAN 20 → 192.168.20.0/24 → SVI 192.168.20.254 → Pool VLAN20
각 Pool의 default-router = 같은 VLAN의 SVI IP
각 SVI IP = excluded-address에 포함
```

> VLAN은 Broadcast Domain 분리 수단이며 그 자체로 완전한 보안 경계가 아닙니다. VLAN 간 통신 제어는 Inter-VLAN Routing 지점의 ACL 또는 별도 보안 정책으로 구현해야 합니다.

---

### Q11. 중복 주소는 어떻게 감지하고 처리하는가?

👉 **핵심 답변:** Cisco IOS DHCP Server는 Offer 전 Ping/ARP 검사에서 응답을 받거나 Client가 DHCP Decline을 보내면 해당 IP를 Conflict로 기록하고 임대 대상에서 제외합니다. `show ip dhcp conflict`로 확인하고 원인을 제거한 뒤 `clear ip dhcp conflict`로 초기화합니다.

```properties
Conflict 기록 사유
= Ping 검사 응답 수신 (해당 IP가 이미 사용 중)
= Client의 DHCP Decline 수신 (Gratuitous ARP로 중복 탐지)

Conflict 상태 IP = 임대 후보에서 제외
원인 제거 없이 clear만 하면 반드시 재발
```

```cisco
show ip dhcp conflict
show ip dhcp binding
show ip dhcp pool
show running-config | include ip dhcp ping
!
clear ip dhcp conflict *
```

Conflict가 반복되면 해당 IP를 Static으로 쓰는 장비, `excluded-address` 누락 대역, 같은 Segment의 다른 DHCP Server, 다른 Segment와 중복 설계된 Pool 대역을 순서대로 점검합니다.

> Ping 검사 명령(`ip dhcp ping packets`, `ip dhcp ping timeout`)의 지원 범위와 기본값은 플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer에서는 Conflict 기록과 Ping 검사 동작이 실장비와 다르게 나타날 수 있습니다.

---

### Q12. 방어 관점에서 DHCP Snooping은 무엇인가?

👉 **핵심 답변:** DHCP Snooping은 Switch가 Port를 신뢰(Trusted)와 비신뢰(Untrusted)로 구분해, 비신뢰 Port에서 들어오는 DHCP Server 응답(Offer/ACK)을 차단하고 정상 Client 요청만 통과시키는 L2 방어 기능입니다.

```properties
Trusted Port
= 정상 DHCP Server 또는 상위 Uplink 방향
= Server 응답 메시지 허용

Untrusted Port (기본값)
= 일반 사용자 단말 방향
= Server 응답 메시지 차단
= Client 요청은 허용
= Binding Table 기반 검증 대상
```

```cisco
configure terminal
!
ip dhcp snooping
ip dhcp snooping vlan <VLAN-ID>
!
interface <INTERFACE>
 description UPLINK-TO-DHCP-SERVER
 ip dhcp snooping trust
end
!
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
```

Option 82는 Relay Agent 또는 Snooping 장비가 요청에 수신 장비·Port·VLAN 정보를 삽입하는 옵션입니다.

```properties
삽입 주체 = DHCP Relay Agent 또는 DHCP Snooping 장비
활용      = Server 측에서 요청 Port/VLAN 식별, 정책 기반 할당과 추적
주의점    = Relay Agent가 Option 82 포함 요청을 받았을 때의 처리 정책에 따라
            Trusted 여부로 폐기될 수 있음 (관련 명령 예: no ip dhcp snooping
            information option, ip dhcp relay information trust-all)
```

> Option 82 처리 정책(`ip dhcp snooping information option`, `ip dhcp relay information trusted`, `ip dhcp relay information policy`)의 문법과 기본값은 플랫폼과 IOS 버전 차이가 크므로 플랫폼별 지원 여부 확인이 필요합니다.

> Packet Tracer는 DHCP Snooping과 Option 82를 제한적으로만 구현하거나 미지원인 경우가 있습니다. 실습에서 동작하지 않아도 실장비 동작과 다를 수 있음을 전제해야 합니다.

Port 단위 방어는 6-1. 🔐 Port-security & Storm-control와 함께 설계하면 효과가 큽니다.

---

## 2. 🛠️ Cisco IOS DHCP Server 표준 설정

### 2-1. 기본 Pool 구성

```cisco
configure terminal
!
interface <INTERFACE>
 description GATEWAY-FOR-USER-SEGMENT
 ip address 192.168.10.254 255.255.255.0
 no shutdown
!
ip dhcp excluded-address 192.168.10.250 192.168.10.254
!
ip dhcp pool USERS_VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 192.168.10.252 192.168.10.253
 domain-name git.local
 lease 3 0 0
end
!
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp server statistics
show running-config | section ip dhcp
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 2-2. excluded-address와 Lease 설계

```cisco
configure terminal
!
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.252 192.168.10.254
ip dhcp excluded-address 192.168.10.100 192.168.10.120
!
ip dhcp pool USERS_VLAN10
 lease 0 12 0
end
!
show running-config | include excluded-address
show ip dhcp pool
```

```properties
lease 3 = 3일 / lease 0 12 0 = 12시간 / lease 0 0 30 = 30분 / lease infinite = 무제한

excluded-address는 이미 임대된 IP를 자동 회수하지 않는다
→ 기존 잘못된 임대는 clear ip dhcp binding으로 정리 검토
```

---

### 2-3. DHCP Client 구성

```cisco
configure terminal
!
interface <INTERFACE>
 ip address dhcp
 no shutdown
end
!
show ip interface brief
show dhcp lease
show ip route
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```properties
ip address dhcp 사용 시 주의
= 기존 Static IP가 제거되어 관리 세션이 끊길 수 있음
= 받은 IP가 예측 불가하므로 관리 대상 Interface에는 신중
= 기본 경로를 DHCP로 받을 수 있어 기존 Routing과 충돌 가능
```

---

### 2-4. DHCP Relay 구성 (Router Interface)

```cisco
configure terminal
!
interface <INTERFACE>
 description GATEWAY-FOR-VLAN10
 ip address 192.168.10.254 255.255.255.0
 ip helper-address 10.10.10.1
 no shutdown
end
!
show ip interface <INTERFACE> | include Helper
show running-config interface <INTERFACE>
show ip route 10.10.10.1
show ip dhcp server statistics
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```properties
Relay 동작 필수 조건
[1] ip helper-address가 Client가 접속한 Segment의 Interface에 설정
[2] Relay Agent에서 Server IP로 Routing 경로 존재
[3] Server에서 Relay Interface 대역(giaddr)으로 되돌아오는 경로 존재
[4] Server에 giaddr 대역과 일치하는 Pool 존재
[5] 경로상 ACL이 UDP 67/68을 차단하지 않음
```

---

### 2-5. L3 Switch SVI Relay와 다중 VLAN

```cisco
configure terminal
!
ip routing
!
interface Vlan10
 description GATEWAY-VLAN10
 ip address 192.168.10.254 255.255.255.0
 ip helper-address 10.10.10.1
 no shutdown
!
interface Vlan20
 description GATEWAY-VLAN20
 ip address 192.168.20.254 255.255.255.0
 ip helper-address 10.10.10.1
 no shutdown
end
!
show ip interface brief
show ip interface vlan <VLAN-ID> | include Helper
show interfaces vlan <VLAN-ID>
show vlan brief
show ip route
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```properties
giaddr 판별 원리
Vlan10 요청 → giaddr 192.168.10.254 → Server가 192.168.10.0/24 Pool 선택
Vlan20 요청 → giaddr 192.168.20.254 → Server가 192.168.20.0/24 Pool 선택
```

---

## 3. 🔍 검증 명령어

```cisco
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp server statistics
show ip dhcp database
!
show ip interface brief
show dhcp lease
!
show ip interface <INTERFACE> | include Helper
show running-config | include helper-address
show running-config | include forward-protocol
show ip route
show access-lists
!
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
!
debug ip dhcp server events
debug ip dhcp server packet
!
undebug all
```

> `debug ip dhcp server events`와 `debug ip dhcp server packet`은 CPU와 Console 부하를 유발합니다. 운영망에서는 짧게 사용하고 반드시 `undebug all`로 종료하십시오. Packet Tracer의 debug 출력은 실장비와 형식·상세도가 다를 수 있습니다.

---

## 4. 🛡️ 안전한 변경 및 Rollback 절차

```properties
[1] 현재 확인 : show ip dhcp pool / binding / conflict, show ip interface brief
[2] 백업      : show running-config 저장, 기존 binding 목록 저장
[3] 경로 확인 : Server IP까지의 Routing과 ACL, giaddr 대역 확인
[4] 적용      : excluded-address → Pool → helper-address 순서
[5] 검증      : 테스트 단말 1대로 실제 임대와 통신 확인
[6] 저장      : copy running-config startup-config

[Rollback]
[1] 변경 전 running-config 원문 확보 여부 확인
[2] 추가한 helper-address 제거 : no ip helper-address <IP-ADDRESS>
[3] 추가한 Pool 제거           : no ip dhcp pool <NAME>
[4] 변경한 excluded-address 원복
[5] 잘못 배포된 임대 정리      : clear ip dhcp binding *
[6] 단말 재획득 유도 후 재검증
```

```cisco
show running-config | section ip dhcp
show ip dhcp binding
!
clear ip dhcp binding *
!
copy running-config startup-config
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `clear ip dhcp binding *`은 모든 임대 기록을 삭제합니다. 단말은 갱신 시점에 재요청하므로 즉시 IP를 잃지는 않을 수 있으나, 동일 IP 재할당이 보장되지 않으므로 서비스 영향을 검토하고 실행해야 합니다.

---

## 5. ⚠️ 실무 체크리스트

- [ ] Gateway IP를 `excluded-address`로 제외했는가?
- [ ] DHCP Server 자신의 IP와 Static IP 장비 대역을 제외했는가?
- [ ] HSRP/VRRP Virtual IP와 실 IP를 모두 제외했는가?
- [ ] Pool의 `network`가 실제 Segment와 일치하는가?
- [ ] Pool의 `default-router`가 실제 Gateway IP와 일치하는가?
- [ ] `dns-server` 주소가 실제 도달 가능한가?
- [ ] Lease Time이 환경에 맞게 설계됐는가?
- [ ] Relay 환경에서 `ip helper-address`를 Client Segment Interface에 설정했는가?
- [ ] Relay Agent에서 Server까지, Server에서 giaddr 대역까지 양방향 경로가 있는가?
- [ ] 경로상 ACL이 UDP 67/68을 차단하지 않는가?
- [ ] `ip forward-protocol udp`로 불필요한 서비스 전달을 정리했는가?
- [ ] SVI가 Up이며 해당 VLAN에 활성 Port가 있는가?
- [ ] `show ip dhcp conflict`가 비어 있는가?
- [ ] 비인가 Server 방어를 위해 DHCP Snooping 적용을 검토했는가?
- [ ] 변경 전 Configuration을 백업하고 Rollback을 준비했는가?

---

> 📌 **핵심 요약**
> - DHCP는 UDP 67(Server)/68(Client)을 사용하는 응용 계층 프로토콜이다.
> - DORA는 Discover → Offer → Request → ACK 순서로 동작한다.
> - Client는 IP가 없으므로 Discover와 Request를 Broadcast로 전송한다.
> - Request가 Broadcast인 이유는 미선택 Server의 Offer를 회수시키기 위함이다.
> - Broadcast는 Router를 넘지 못하므로 다른 Subnet에서는 Relay가 필요하다.
> - Relay Agent는 Broadcast를 Unicast로 변환하고 giaddr에 수신 Interface IP를 기록한다.
> - Server는 giaddr로 어느 Subnet 요청인지 판별해 Pool을 선택한다.
> - `ip helper-address`는 DHCP뿐 아니라 여러 UDP 서비스를 함께 전달하므로 `ip forward-protocol udp`로 조정한다.
> - Lease 갱신은 T1 50% Unicast, T2 87.5% Broadcast 순서로 진행된다.
> - Gateway·Server·Static 장비 주소는 반드시 `excluded-address`로 제외한다.
> - Static Binding은 `host` + `hardware-address` 또는 `client-identifier`로 구성한다.
> - L3 Switch에서는 SVI가 Up이어야 Pool과 Relay가 정상 동작한다.
> - Conflict는 원인을 제거한 뒤 `clear ip dhcp conflict`로 정리한다.
> - DHCP Snooping은 비신뢰 Port의 Server 응답을 차단하는 방어 기능이며, VLAN은 분리 수단일 뿐 완전한 보안 경계가 아니다.
> - 관련: 5-1. 💾 DHCP (동적 IP 할당) · 5-1-1. 🧩 DHCP 통합 정리 · 5-1-2. 🚑 DHCP 트러블슈팅 치트시트 · 5-1-3. ⚡ DHCP 명령어 퀵 레퍼런스 · 5-2. 🔀 NAT & PAT (주소 변환) · 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing
