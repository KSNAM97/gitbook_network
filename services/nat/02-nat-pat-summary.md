# 🧩 NAT & PAT 통합 정리

> **Tag:** #Network #NAT #PAT #IPServices #Cisco  
> **핵심 요약:** NAT는 IP Header의 주소를 변환하는 L3 기능이며, 동작의 전제는 `ip nat inside`와 `ip nat outside` 방향 지정입니다. Static NAT는 1:1 고정 매핑으로 외부→내부 접속을 허용하고, Dynamic NAT는 Pool에서 임시 배정하며, PAT(Overload)는 Port 번호로 구분해 공인 IP 1개를 다수 단말이 공유합니다. NAT 대상 선별 ACL의 `permit`은 트래픽 허용이 아니라 "변환 대상 선정"을 의미합니다.

---

## 1. 🎯 핵심 기술 개념

### Q1. NAT는 무엇을 변환하는가?

👉 **핵심 답변:** NAT는 IP Header의 Source 또는 Destination IP Address를 교체하는 L3 기능이며, PAT는 여기에 TCP/UDP Port 또는 ICMP Query ID까지 함께 교체합니다.

```properties
L2  = MAC Address, VLAN, Trunk
L3  = IP Address, Routing, NAT
L4  = TCP/UDP Port (PAT가 추가로 사용하는 식별자)

NAT = IP Address 교체 (L3)
PAT = IP Address + Port/Query ID 교체 (L3 + L4 식별자)
```

NAT는 L3 기능이므로 L1/L2가 Up이고 IP Routing이 정상이어야 의미가 있습니다. NAT는 경로를 만들어 주지 않고 방화벽 정책을 대신하지도 않습니다.

---

### Q2. 사설 IP는 왜 그대로 인터넷을 사용할 수 없는가?

👉 **핵심 답변:** RFC 1918 사설 대역은 인터넷 Global Routing Table에 광고되지 않도록 예약된 대역이므로, ISP Router는 해당 트래픽을 폐기하거나 응답 경로를 만들 수 없습니다.

| 분류 | 대역 | Prefix |
|---|---|---|
| Class A | 10.0.0.0 ~ 10.255.255.255 | 10.0.0.0/8 |
| Class B | 172.16.0.0 ~ 172.31.255.255 | 172.16.0.0/12 |
| Class C | 192.168.0.0 ~ 192.168.255.255 | 192.168.0.0/16 |

부수 효과는 IPv4 주소 고갈 완화와 내부 주소 은닉이며, 은닉은 보안 효과의 일부일 뿐 ACL을 대체하지 않습니다.

---

### Q3. NAT 4대 용어는 어떻게 구분하는가?

👉 **핵심 답변:** "Inside/Outside"는 **어느 쪽 호스트의 주소인가**를, "Local/Global"은 **어느 쪽 네트워크에서 본 표현인가**를 나타냅니다.

| 용어 | 누구의 주소 | 어디에서 본 표현 | 예시 |
|---|---|---|---|
| **Inside Local** | 내부 호스트 | 내부망 | `192.168.10.10` (실제 사설 IP) |
| **Inside Global** | 내부 호스트 | 외부망 | `121.160.23.2` (변환된 공인 IP) |
| **Outside Local** | 외부 호스트 | 내부망 | `200.1.1.1` |
| **Outside Global** | 외부 호스트 | 외부망 | `200.1.1.1` |

```properties
내부 단말이 외부 웹서버에 접속하는 PAT 상황

Inside Local   192.168.10.10:1500
Inside Global  121.160.23.2:1500
Outside Local  200.1.1.1:80
Outside Global 200.1.1.1:80
```

- 일반적인 Inside NAT 환경에서는 Outside Local과 Outside Global이 동일하게 표시됩니다.
- 두 값이 달라지는 것은 `ip nat outside source`로 **외부 주소까지 변환**할 때입니다.
- "누구의 주소 / 누구의 시선" 두 축으로 나눠 기억하면 혼동하지 않습니다.

---

### Q4. Outside Local과 Outside Global이 달라지는 경우는?

👉 **핵심 답변:** 두 사설망 통합으로 대역이 겹치거나 외부 서버를 내부에서 다른 주소로 보이게 해야 할 때 Outside NAT를 사용하면 값이 달라집니다.

```cisco
ip nat outside source static <OUTSIDE-GLOBAL-IP> <OUTSIDE-LOCAL-IP>
```

검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
```

```properties
ip nat inside source  = 내부 호스트 주소 변환 (Inside Local ↔ Inside Global)
ip nat outside source = 외부 호스트 주소 변환 (Outside Global ↔ Outside Local)
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

방향별로 라우팅이 참조하는 주소가 달라지므로 적용 전 Outside Local 기준 경로를 확인해야 하며, 세부 동작은 플랫폼별 지원 여부 확인이 필요합니다.

---

### Q5. NAT 유형은 어떻게 비교하는가?

| 유형 | 매핑 방식 | 필요 공인 IP | 외부→내부 최초 접속 | Entry 생성 시점 |
|---|---|:---:|:---:|---|
| **Static NAT** | 사설 1 ↔ 공인 1 고정 | 호스트 수만큼 | 가능 | 설정 즉시(상시) |
| **Dynamic NAT** | 사설 다수 ↔ Pool 임시 배정 | Pool 크기 | 불가 | 내부 트래픽 발생 시 |
| **PAT (Overload)** | 사설 다수 → 공인 1 (Port 구분) | 1개 | 불가 | 내부 트래픽 발생 시 |
| **Static PAT (Port Forwarding)** | 사설 1:Port ↔ 공인 1:Port | 1개 공유 | 지정 Port만 가능 | 설정 즉시(상시) |

Static Entry는 트래픽이 없어도 표시되지만 Dynamic·PAT Entry는 트래픽이 없으면 표시되지 않으므로, `show ip nat translations` 출력이 비어 있다는 사실만으로 설정 오류라고 단정할 수 없습니다.

---

### Q6. `ip nat inside`와 `ip nat outside`는 왜 필수인가?

👉 **핵심 답변:** NAT는 패킷이 Inside 지정 Interface와 Outside 지정 Interface 사이를 통과할 때 동작하므로, 방향 지정이 없으면 변환 규칙이 있어도 아무 일도 일어나지 않습니다.

```cisco
interface <INTERFACE>
 description TO-LAN
 ip address <IP-ADDRESS> <SUBNET-MASK>
 ip nat inside
!
interface <INTERFACE>
 description TO-ISP
 ip address <IP-ADDRESS> <SUBNET-MASK>
 ip nat outside
```

검증:

```cisco
show ip nat statistics
show ip nat translations
show run | section ip nat
```

```properties
show ip nat statistics의 Inside/Outside interfaces 목록이 비어 있음
= 변환 규칙이 있어도 NAT 미동작
```

자주 발생하는 실수는 한쪽만 지정, Sub-interface 대신 물리 Interface에 지정, Tunnel과 물리 중 잘못된 쪽에 지정, 이중화 Backup Interface 누락, Interface 교체 후 재적용 누락입니다.

> NAT 장애의 최다 원인이 방향 지정 누락입니다. 다른 항목보다 먼저 확인하십시오.

---

### Q7. 네 가지 변환 방식은 각각 어떻게 설정하는가?

#### Static NAT (1:1 고정)

```cisco
ip nat inside source static <INSIDE-LOCAL-IP> <INSIDE-GLOBAL-IP>
!
ip nat inside source static 192.168.10.50 121.160.23.50
```

Inside Global 주소가 ISP에서 우리 쪽으로 실제 라우팅되어야 하며, 다른 규칙이나 Interface가 이미 쓰는 주소면 충돌합니다. Static NAT는 "변환"만 하므로 공개 범위는 ACL로 통제합니다.

#### Dynamic NAT (Pool 임시 배정)

```cisco
access-list <ACL-NUMBER> permit <NETWORK> <WILDCARD-MASK>
ip nat pool <POOL-NAME> <START-IP> <END-IP> netmask <SUBNET-MASK>
ip nat inside source list <ACL-NUMBER> pool <POOL-NAME>
!
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat pool PUBLIC_POOL 121.160.23.10 121.160.23.20 netmask 255.255.255.0
ip nat inside source list 1 pool PUBLIC_POOL
```

`netmask` 대신 `prefix-length <LENGTH>`도 사용할 수 있고, Pool 주소는 Interface 주소와 겹치지 않게 설계합니다.

#### PAT (Overload)

```cisco
ip nat inside source list <ACL-NUMBER> pool <POOL-NAME> overload
ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
```

#### Port Forwarding (Static PAT)

```cisco
ip nat inside source static tcp <INSIDE-IP> <PORT> <OUTSIDE-IP> <PORT>
ip nat inside source static tcp <INSIDE-IP> <PORT> interface <INTERFACE> <PORT>
!
ip nat inside source static tcp 192.168.10.10 80 121.160.23.2 80
ip nat inside source static tcp 192.168.10.11 443 121.160.23.2 443
ip nat inside source static udp 192.168.10.20 53 121.160.23.2 53
```

검증(네 방식 공통):

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
```

```properties
overload 있음 = PAT, Port로 세션 구분, 공인 IP 1개 공유
overload 없음 = Dynamic NAT, 주소 1:1 임시 배정, Pool 고갈 위험

Pool 주소 수 < 동시 통신 호스트 수 → 배정 실패 → NAT Miss 증가

121.160.23.2:80  → 192.168.10.10:80
121.160.23.2:443 → 192.168.10.11:443
121.160.23.2:53  → 192.168.10.20:53 (UDP)
```

Interface 기반 PAT는 Interface 주소가 DHCP·PPP로 바뀌어도 자동 반영되고 Pool 관리가 필요 없지만, 해당 Interface에 실제 공인 IP가 있고 `ip nat outside`여야 합니다. Port Forwarding은 외부 Port와 내부 Port를 다르게 지정할 수도 있습니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Router 관리 Port를 Forwarding하면 원격 접속이 불가능해질 수 있으므로 Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q8. NAT 대상 ACL의 `permit`은 무슨 의미인가?

👉 **핵심 답변:** NAT에서 쓰이는 ACL의 `permit`은 "트래픽 허용"이 아니라 "이 트래픽을 변환 대상으로 선정"이고, `deny`는 "변환하지 않음"입니다.

```properties
[Interface에 적용된 ACL]
permit = 통과 허용 / deny = 폐기

[NAT 대상 선별 ACL]
permit = NAT 수행(변환 대상) / deny = NAT 미수행(원본 주소 유지)
```

```cisco
ip access-list extended NAT_TARGET
 deny   ip 10.10.0.0 0.0.255.255 10.20.0.0 0.0.255.255
 permit ip 10.10.0.0 0.0.255.255 any
!
ip nat inside source list NAT_TARGET interface <INTERFACE> overload
```

검증:

```cisco
show access-lists NAT_TARGET
show ip nat translations
show run | section ip nat
```

위 `deny` 줄은 트래픽을 차단하는 것이 아니라 **변환에서 제외**하며, 본사·지사 VPN 트래픽을 사설 주소 그대로 보낼 때 쓰는 표준 방식입니다.

- `deny`만 있고 `permit`이 없으면 변환 대상이 없어 NAT가 전혀 동작하지 않습니다.
- 범위를 지나치게 넓게(`permit ip any any`) 쓰면 VPN 트래픽까지 변환되어 통신이 깨집니다.
- ACL 원칙은 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named)과 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리를 참고하십시오.

---

### Q9. NAT는 라우팅·ACL과 어떤 순서로 처리되는가?

👉 **핵심 답변:** Inside→Outside는 **라우팅 후 NAT**, Outside→Inside는 **NAT 후 라우팅**입니다. 방향에 따라 ACL이 보는 주소도 달라집니다.

```properties
[Inside → Outside]
Inside Inbound ACL (Inside Local 기준)
    ↓
Routing 결정
    ↓
NAT 변환 (Inside Local → Inside Global)
    ↓
Outside Outbound ACL (Inside Global 기준)

[Outside → Inside]
Outside Inbound ACL (Inside Global 기준)
    ↓
NAT 역변환 (Inside Global → Inside Local)
    ↓
Routing 결정
    ↓
Inside Outbound ACL (Inside Local 기준)
```

```properties
Outside Interface Inbound ACL = 변환 전이므로 공인 IP로 작성
Inside Interface Outbound ACL = 변환 후이므로 사설 IP로 작성
```

- 외부에서 내부 서버로 들어오는 트래픽 허용 ACL의 목적지는 **사설 IP가 아니라 공인 IP**입니다.
- Inside→Outside는 라우팅이 먼저이므로, 목적지 경로가 없으면 변환 Entry조차 생기지 않습니다.

> 세부 처리 단계는 정책 라우팅·암호화·Inspection 조합에 따라 달라질 수 있으므로 플랫폼별 지원 여부 확인이 필요합니다.

---

### Q10. NAT와 함께 확인해야 할 라우팅 조건은?

변환이 성공해도 내부→외부 경로와 Inside Global 주소로 오는 ISP 경로가 모두 있어야 통신이 완성됩니다.

```cisco
ip route 0.0.0.0 0.0.0.0 <NEXT-HOP-IP>
```

검증:

```cisco
show ip route
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
```

```properties
경로 선택 순서
[1] Longest Prefix Match  : 가장 구체적인 Prefix 우선
[2] Administrative Distance : 동일 Prefix일 때 비교
[3] Metric                : 동일 Prefix·동일 Protocol일 때 비교
```

`/32` Static Route가 있으면 AD가 더 낮은 다른 경로보다 우선될 수 있으므로, "AD가 낮으니 선택된다"는 판단은 Prefix 길이를 확인한 뒤에만 유효합니다. 정적 경로는 2. 📍 Static Route (정적 라우팅)을 참고하십시오.

---

### Q11. NAT와 VPN(GRE/IPsec)을 함께 쓸 때 무엇을 조심해야 하는가?

👉 **핵심 답변:** VPN으로 보낼 사설 대역 간 트래픽이 PAT로 변환되면 상대측 Crypto 정책과 주소가 맞지 않아 Tunnel을 통과하지 못하므로, NAT 예외(NAT Exemption)를 반드시 설정해야 합니다.

방식 1 — NAT ACL `deny`, 방식 2 — route-map:

```cisco
ip access-list extended NAT_TARGET
 deny   ip 10.10.0.0 0.0.255.255 10.20.0.0 0.0.255.255
 permit ip 10.10.0.0 0.0.255.255 any
!
ip nat inside source list NAT_TARGET interface <INTERFACE> overload
!
route-map NAT_EXEMPT permit 10
 match ip address NAT_TARGET
!
ip nat inside source route-map NAT_EXEMPT interface <INTERFACE> overload
```

검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
show crypto ipsec sa
```

```properties
VPN 대상 트래픽이 NAT됨 → Crypto ACL·Proxy ID 불일치 → Tunnel 미형성 또는 통신 불가
```

GRE 구성에서는 `ip nat inside`를 물리 Interface와 Tunnel Interface 중 어디에 둘지에 따라 결과가 달라집니다. 상세 설계는 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE)와 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화)를 참고하십시오.

> route-map 기반 NAT와 `overload` 조합의 세부 동작은 IOS 버전별 차이가 있어 플랫폼별 지원 여부 확인이 필요하고, Packet Tracer는 route-map 기반 NAT를 실장비와 동일하게 지원하지 않을 수 있습니다.

---

### Q12. NAT-T(NAT Traversal)는 무엇인가?

👉 **핵심 답변:** IPsec ESP는 TCP/UDP Port가 없어 PAT가 세션을 구분할 수 없으므로, ESP를 UDP 4500으로 캡슐화해 PAT 구간을 통과시키는 기능이 NAT-T입니다.

```properties
IKE 협상        : UDP 500
NAT 감지 후 전환 : UDP 4500
ESP 캡슐화      : UDP 4500 Payload로 전송

PAT + ESP 직접 = 세션 구분 불가로 실패 가능
PAT + NAT-T    = UDP 4500으로 통과 가능
NAT + AH       = IP Header 인증 충돌로 실패
```

IKE Phase 1에서 경로상 NAT 존재를 감지하면 UDP 4500으로 전환하므로 중간 방화벽에서 UDP 500과 4500이 모두 허용돼야 하고, ESP(IP Protocol 50)를 그대로 통과시키려면 PAT가 아닌 1:1 Static NAT가 필요합니다.

> NAT-T 지원 범위와 자동 감지 동작은 플랫폼·IOS 버전에 따라 다르므로 확인이 필요하며, Packet Tracer는 NAT-T를 재현하지 않을 수 있습니다.

---

### Q13. NAT 환경에서 ICMP와 Traceroute는 어떻게 동작하는가?

ICMP Echo는 Port가 없으므로 PAT는 ICMP Query ID로 세션을 구분합니다.

```properties
Pro   Inside global        Inside local        Outside local   Outside global
icmp  121.160.23.2:8       192.168.10.10:8     200.1.1.1:8     200.1.1.1:8
tcp   121.160.23.2:1500    192.168.10.10:1500  200.1.1.1:80    200.1.1.1:80
```

- ICMP Error(Time Exceeded, Unreachable)는 원본 Header를 Payload에 포함하므로, NAT가 Payload 내부 주소까지 수정해야 정상 표시됩니다.
- 중간 Hop이 사설 IP로 보이거나 `* * *`가 나오는 것은 ICMP 정책·ACL·Rate-limit이 섞인 결과일 수 있습니다.
- Ping 성공만으로 TCP 서비스 정상을 판단할 수 없습니다.

> Packet Tracer의 ICMP Query ID 표기와 Traceroute 결과는 실장비와 다르게 보일 수 있습니다.

---

### Q14. Translation Table과 Timeout은 어떻게 관리하는가?

```cisco
ip nat translation timeout <SECONDS>
ip nat translation tcp-timeout <SECONDS>
ip nat translation udp-timeout <SECONDS>
ip nat translation icmp-timeout <SECONDS>
ip nat translation finrst-timeout <SECONDS>
```

검증과 해제:

```cisco
show ip nat translations verbose
show ip nat statistics
show run | section ip nat
!
clear ip nat translation inside <INSIDE-GLOBAL-IP> <INSIDE-LOCAL-IP>
clear ip nat translation *
```

```properties
Timeout 과대 = Entry 누적, 메모리·Port 자원 부담
Timeout 과소 = 유지되어야 할 세션 조기 종료
```

> ⚠️ `clear ip nat translation *`은 유지 중인 모든 Dynamic 변환 Entry를 삭제하므로 진행 중인 사용자 세션과 원격 관리 접속이 끊길 수 있습니다. 선별 삭제를 우선하고, 전체 삭제는 영향 범위를 확인한 뒤 Maintenance Window에 실행하십시오.

Static Entry는 `clear`로 제거되지 않고 설정에서 삭제해야 사라집니다. 기본값과 설정 범위는 플랫폼별로 다르며, Packet Tracer는 Timeout 명령을 지원하지 않거나 다르게 동작할 수 있습니다.

---

### Q15. `debug ip nat`은 언제 사용하는가?

```cisco
debug ip nat
debug ip nat detailed
!
no debug all
```

> ⚠️ `debug ip nat`은 변환되는 모든 패킷에 로그를 생성하므로 트래픽이 많은 운영 장비에서 CPU 과부하와 Console 폭주로 장비가 응답하지 않을 수 있습니다. 범위를 좁혀 짧게만 사용하고 즉시 해제하십시오.

```properties
[1] Console 또는 Buffer 로깅 확인 경로 확보
[2] ACL로 대상 트래픽 범위 최소화
[3] 짧은 시간만 활성화
[4] 즉시 no debug all
[5] show processes cpu sorted로 부하 확인
```

`debug` 이전에 `show ip nat translations`, `show ip nat statistics`, `show run | section ip nat`를 먼저 확인하십시오.

---

## 2. 🛠️ 표준 설정 템플릿

### 2-1. Interface 기반 PAT + 공개 서버 (공인 IP 1개)

```cisco
configure terminal
!
interface <INTERFACE>
 description TO-LAN
 ip address <IP-ADDRESS> <SUBNET-MASK>
 ip nat inside
!
interface <INTERFACE>
 description TO-ISP
 ip address <IP-ADDRESS> <SUBNET-MASK>
 ip nat outside
!
access-list <ACL-NUMBER> permit 192.168.10.0 0.0.0.255
ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
!
ip nat inside source static tcp 192.168.10.10 80 121.160.23.2 80
!
ip route 0.0.0.0 0.0.0.0 <NEXT-HOP-IP>
end
```

검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
show ip route
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 2-2. VPN 예외 PAT와 ISP 방향 사설 대역 차단

```cisco
configure terminal
!
ip access-list extended NAT_TARGET
 deny   ip 10.10.0.0 0.0.255.255 10.20.0.0 0.0.255.255
 permit ip 10.10.0.0 0.0.255.255 any
!
ip nat inside source list NAT_TARGET interface <INTERFACE> overload
!
ip access-list standard ANTI_PRIVATE
 deny   10.0.0.0 0.255.255.255
 deny   172.16.0.0 0.15.255.255
 deny   192.168.0.0 0.0.255.255
 permit any
!
interface <INTERFACE>
 ip access-group ANTI_PRIVATE out
end
```

검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
show access-lists
```

> `ANTI_PRIVATE`의 `deny`는 실제 트래픽 차단이고, `NAT_TARGET`의 `deny`는 변환 제외입니다. 두 역할을 혼동하지 마십시오.

---

## 3. 🔧 안전한 변경 절차

```properties
[1] 현재 확인 : show run | section ip nat / show ip nat translations / show ip nat statistics
[2] 백업     : Configuration 및 현재 변환 Entry 출력 저장
[3] 경로 확인 : show ip route / show ip route <IP-ADDRESS>
[4] 적용     : 방향 지정 → ACL → Pool → 변환 규칙 순서
[5] 검증     : 변환 Entry 생성 및 실제 서비스 테스트
[6] 저장     : copy running-config startup-config
```

백업과 Rollback:

```cisco
copy running-config flash:pre-nat-change.cfg
!
configure terminal
no ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
no ip nat inside source static <INSIDE-LOCAL-IP> <INSIDE-GLOBAL-IP>
no ip nat pool <POOL-NAME>
end
```

Rollback 후 검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 4. 🔍 검증 명령어

```cisco
show ip nat translations
show ip nat translations verbose
show ip nat statistics
show run | section ip nat
show access-lists
```

Interface·라우팅·부하:

```cisco
show ip interface brief
show ip interface <INTERFACE>
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
show processes cpu sorted
show logging
```

---

## 5. ⚠️ 실무 체크리스트

- [ ] 내부 Interface에 `ip nat inside`를 지정했는가?
- [ ] 외부 Interface에 `ip nat outside`를 지정했는가?
- [ ] `show ip nat statistics`의 Interface 목록에 두 방향이 표시되는가?
- [ ] NAT 대상 ACL의 `permit`이 변환 대상 선정임을 이해했는가?
- [ ] NAT 대상 ACL 범위가 필요 이상으로 넓지 않은가?
- [ ] 공인 IP가 1개인데 `overload`를 빠뜨리지 않았는가?
- [ ] Pool 크기와 동시 세션 수를 비교했는가?
- [ ] 외부 공개 서버에 Static NAT 또는 Port Forwarding을 적용했는가?
- [ ] Outside Inbound ACL을 공인 IP 기준으로 작성했는가?
- [ ] Inside Outbound ACL을 사설 IP 기준으로 작성했는가?
- [ ] Default Route와 ISP 측 Return 경로가 존재하는가?
- [ ] 경로 판단을 Longest Prefix Match부터 확인했는가?
- [ ] VPN 대상 트래픽을 NAT에서 제외했는가?
- [ ] `clear ip nat translation *`의 세션 영향을 검토했는가?
- [ ] `debug ip nat` 사용 후 즉시 해제했는가?
- [ ] 변경 전 Configuration을 백업하고 Rollback 절차를 준비했는가?

---

> 📌 **핵심 요약**
> - NAT는 IP Header 주소를 교체하는 L3 기능이고, PAT는 Port 또는 ICMP Query ID까지 사용한다.
> - `ip nat inside`와 `ip nat outside` 방향 지정이 없으면 NAT는 동작하지 않는다.
> - Inside/Outside는 누구의 주소인지, Local/Global은 어느 쪽 시선인지를 나타낸다.
> - Outside Local과 Outside Global이 달라지는 것은 Outside NAT를 사용할 때다.
> - Static NAT만 외부→내부 최초 접속을 허용하고, Port Forwarding은 지정 Port에 한해 허용한다.
> - `overload`를 붙이면 PAT이고, 빼면 Dynamic NAT로 Pool 고갈 위험이 있다.
> - NAT 대상 ACL의 `permit`은 허용이 아니라 변환 대상 선정이며 `deny`는 변환 제외다.
> - Inside→Outside는 라우팅 후 NAT, Outside→Inside는 NAT 후 라우팅으로 순서가 다르다.
> - Outside Inbound ACL은 공인 IP, Inside Outbound ACL은 사설 IP 기준으로 작성한다.
> - 경로 선택은 Longest Prefix Match → AD → Metric 순서로 판단하며 AD 단독 비교는 금지한다.
> - VPN 대상 사설 대역 간 트래픽은 ACL `deny` 또는 route-map으로 NAT에서 제외한다.
> - IPsec ESP는 PAT를 통과하기 어려워 NAT-T가 UDP 4500으로 캡슐화한다.
> - `clear ip nat translation *`은 진행 중인 세션을 끊을 수 있어 영향 범위 확인이 필요하다.
> - `debug ip nat`은 CPU 부하가 크므로 범위를 좁혀 짧게만 사용하고 즉시 해제한다.
> - 관련: 5-2. 🔀 NAT & PAT (주소 변환) · 5-2-1. 🧩 NAT & PAT 통합 정리 · 5-2-2. 🚑 NAT & PAT 트러블슈팅 치트시트 · 5-2-3. ⚡ NAT & PAT 명령어 퀵 레퍼런스 · 5-1. 💾 DHCP (동적 IP 할당) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리
