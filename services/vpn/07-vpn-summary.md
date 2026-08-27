# VPN 통합 정리 (GRE · IPsec · DMVPN)

> **Tag:** #Network #VPN #GRE #IPsec #DMVPN #Cisco  
> **핵심 요약:** GRE는 캡슐화(Encapsulation)만 제공하며 암호화를 제공하지 않고, IPsec은 암호화·무결성·인증을 제공하지만 Multicast 기반 동적 라우팅을 직접 보호하지 못합니다. 그래서 실무에서는 GRE로 라우팅과 Multicast를 실어 나르고 IPsec으로 암호화하는 GRE-over-IPsec, mGRE·NHRP·IPsec Profile을 결합해 확장성을 확보한 DMVPN을 사용합니다. 어떤 방식이든 하부 라우팅 도달성, Crypto ACL 대칭, NAT 예외, MTU가 공통 실패 지점입니다.

---

## 1. 핵심 기술 개념

### Q1. GRE는 암호화를 제공하는가?

👉 **핵심 답변:** 제공하지 않습니다. GRE(Generic Routing Encapsulation)는 원본 패킷을 New IP Header와 GRE Header로 한 번 더 감싸는 **캡슐화 전용 프로토콜**이며 기밀성·무결성·인증을 제공하지 않습니다.

```properties
GRE   = 캡슐화, 사설 대역을 공중망 위로 전달, 암호화 아님
IPsec = 기밀성(암호화) + 무결성 + 인증
```

- GRE는 IP Protocol 47을 사용합니다.
- GRE는 Multicast와 Non-IP Protocol도 Unicast로 감싸 전달할 수 있습니다.
- GRE 터널만 구성한 상태에서는 내부 사설 IP와 Payload가 그대로 노출될 수 있습니다.

> "터널"이라는 표현을 보안 터널로 오해하지 않아야 합니다. 보안이 필요하면 IPsec을 함께 적용해야 합니다.

---

### Q2. GRE Tunnel Interface가 Up/Up이 되는 조건은 무엇인가?

👉 **핵심 답변:** `tunnel source`가 Up이고 `tunnel destination` 주소가 **라우팅 테이블에서 도달 가능**해야 합니다. `tunnel source`만 설정하면 터널은 Down 상태로 남습니다.

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel destination <PEER-IP>
 tunnel mode gre ip
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip interface brief
show ip route <PEER-IP>
```

```properties
[1] tunnel source 인터페이스가 Up
[2] tunnel destination 주소가 라우팅 테이블에 존재
[3] 해당 경로가 Tunnel 자신을 다시 사용하지 않음
[4] 경로상 GRE(Protocol 47)가 차단되지 않음
```

> Tunnel Up/Up은 **Underlay 도달성만 확인된 상태**이며 내부 사설망 통신을 보장하지 않습니다.

---

### Q3. Recursive Routing(%TUN-5-RECURDOWN)은 왜 발생하는가?

👉 **핵심 답변:** `tunnel destination`으로 가는 최적 경로가 **그 터널 자신을 통과하도록 선택**되면 캡슐화가 자기 자신을 참조해 터널이 내려갑니다.

경로 선택 순서는 다음과 같으며, AD만 단독으로 비교해서는 안 됩니다.

```properties
[1] Longest Prefix Match (가장 긴 Prefix 우선)
[2] Administrative Distance
[3] Metric
```

```properties
Overlay 라우팅이 Tunnel Endpoint 대역을 광고
        ↓
Longest Prefix Match로 Tunnel 경로가 선택됨
        ↓
destination을 Tunnel로 캡슐화해야 하는 모순
        ↓
%TUN-5-RECURDOWN 후 Tunnel Down
```

방어 설정:

```cisco
interface Tunnel <TUNNEL-NUMBER>
 tunnel source Loopback0
!
ip prefix-list DENY_UNDERLAY seq 5 deny 121.160.0.0/16 le 32
ip prefix-list DENY_UNDERLAY seq 10 permit 0.0.0.0/0 le 32
!
router eigrp 100
 distribute-list prefix DENY_UNDERLAY in Tunnel <TUNNEL-NUMBER>
```

검증:

```cisco
show ip route <PEER-IP>
show interface tunnel <TUNNEL-NUMBER>
show logging
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q4. GRE 터널에서 MTU와 MSS를 왜 조정하는가?

👉 **핵심 답변:** 캡슐화 Header로 실제 전송 가능한 Payload가 줄기 때문입니다. 조정하지 않으면 Fragmentation이 발생하거나 **큰 패킷과 일부 TCP 세션만 실패**하는 증상이 나타납니다.

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip mtu 1400
 ip tcp adjust-mss 1360
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip interface tunnel <TUNNEL-NUMBER>
ping <IP-ADDRESS> size 1400 df-bit
```

```properties
Ping 성공 + 작은 패킷 성공
+ HTTPS·파일 전송·대용량 응답 실패
        ↓
MTU / MSS / Fragmentation 우선 의심
```

- `ip mtu`는 Tunnel Interface의 IP MTU를 지정합니다.
- `ip tcp adjust-mss`는 TCP Handshake의 MSS를 낮추며 **TCP에만 적용**됩니다.
- 정확한 오버헤드는 IPsec 알고리즘, Mode, NAT-T 사용 여부에 따라 달라지므로 실제 값은 플랫폼과 IOS 버전에서 확인해야 합니다.

---

### Q5. GRE Static과 GRE Dynamic은 무엇이 다른가?

👉 **핵심 답변:** 터널 구성은 동일하고, **원격 사설망 경로를 정적으로 관리하는지 터널 위 동적 라우팅 프로토콜로 교환하는지**가 다릅니다.

| 항목 | GRE Static | GRE Dynamic |
|---|---|---|
| 원격 사설망 경로 | 관리자가 정적 등록 | 라우팅 프로토콜이 자동 교환 |
| 확장성 | 지사 증가 시 부담 증가 | 상대적으로 유리 |
| 장애 자동 대응 | 제한적 | 재수렴으로 대응 |
| 주의점 | 경로 누락 | Recursive Routing, Split Horizon |

```cisco
! GRE Static
ip route 192.168.20.0 255.255.255.0 Tunnel <TUNNEL-NUMBER>
!
! GRE Dynamic
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface Tunnel <TUNNEL-NUMBER>
 network 172.16.100.0 0.0.0.255
```

검증:

```cisco
show ip route
show ip eigrp neighbors
show ip route eigrp
```

`passive-interface default` 환경에서 Tunnel을 `no passive-interface`로 열지 않아 인접이 맺히지 않는 경우가 가장 흔합니다. 라우팅 동작은 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol)과 3-1. 🌲 OSPF (Open Shortest Path First)를 함께 확인하십시오.

---

### Q6. Hub-Spoke 구조에서 Split Horizon 이슈는 왜 발생하는가?

👉 **핵심 답변:** Split Horizon은 **경로를 학습한 인터페이스로 그 경로를 다시 광고하지 않는 규칙**입니다. Hub의 단일 mGRE 터널에 여러 Spoke가 물려 있으면 한 Spoke에서 배운 경로를 같은 터널의 다른 Spoke로 재광고하지 못합니다.

```properties
Spoke A → Hub : 192.168.10.0/24 광고 (수신 IF = Tunnel)
Spoke B도 같은 Tunnel
        ↓
Split Horizon으로 재광고 차단
        ↓
Spoke B는 Spoke A 대역을 학습하지 못함
```

```cisco
interface Tunnel <TUNNEL-NUMBER>
 no ip split-horizon eigrp 100
 ip ospf network point-to-multipoint
```

검증:

```cisco
show ip eigrp neighbors
show ip ospf neighbor
show ip ospf interface tunnel <TUNNEL-NUMBER>
show ip route
```

OSPF는 Split Horizon이 아니라 **Network Type** 문제입니다. Tunnel의 기본 Type은 Point-to-Point이므로 다수 인접을 위해 변경이 필요합니다.

---

### Q7. mGRE와 NHRP는 어떤 역할을 하는가?

👉 **핵심 답변:** mGRE는 `tunnel mode gre multipoint`로 **하나의 터널에 다수 Peer를 수용**하고, NHRP는 **Tunnel IP와 실제 NBMA(공인) IP의 매핑을 등록·조회**해 어느 공인 IP로 캡슐화할지 결정합니다.

```properties
NHS (Next Hop Server) = Hub, 매핑 등록 수신 및 조회 응답
NHC (Next Hop Client) = Spoke, 자신의 매핑을 NHS에 등록

등록 : Spoke 부팅 → (Tunnel IP, NBMA IP) 등록 → Hub가 Dynamic Entry 학습
조회 : 목적지 Tunnel IP 질의 → NBMA IP 확인 → Spoke-to-Spoke 동적 터널 형성
```

Hub:

```cisco
interface Tunnel <TUNNEL-NUMBER>
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp map multicast dynamic
```

Spoke:

```cisco
interface Tunnel <TUNNEL-NUMBER>
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp nhs <IP-ADDRESS>
 ip nhrp map <IP-ADDRESS> <PEER-IP>
 ip nhrp map multicast <PEER-IP>
```

검증:

```cisco
show ip nhrp
show ip nhrp brief
show ip nhrp nhs detail
show interface tunnel <TUNNEL-NUMBER>
```

| 표시 | 의미 |
|---|---|
| static | 관리자가 수동 매핑한 항목 |
| dynamic | NHS가 NHC로부터 학습한 항목 |
| NBMA address | Tunnel IP에 대응하는 실제 공인 IP |

- mGRE에는 `tunnel destination`을 지정하지 않습니다.
- mGRE의 기본 성격은 NBMA이므로 Multicast 전달을 `ip nhrp map multicast`로 정의해야 합니다.
- Spoke에 Multicast 매핑이 없으면 라우팅 Hello가 Hub에 도달하지 않아 인접이 형성되지 않을 수 있습니다.

---

### Q8. IPsec은 무엇을 보장하며 알고리즘은 어떻게 선택하는가?

👉 **핵심 답변:** IPsec은 **기밀성·무결성·인증·재전송 방지**를 제공합니다. 알고리즘은 강도 관점에서 AES/SHA-2 계열과 DH Group 14 이상을 권장하고, DES/3DES/MD5와 DH Group 1/2는 레거시로 구분합니다.

| 구분 | 최신 권장 | 레거시 |
|---|---|---|
| 암호화 | AES-128/192/256, AES-GCM | DES, 3DES |
| 해시 | SHA-256 이상(SHA-2 계열) | MD5, SHA-1 |
| DH Group | Group 14 이상 | Group 1, Group 2 |

```cisco
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256
 hash sha256
 group 14
 lifetime 3600
```

검증:

```cisco
show crypto isakmp policy
show crypto isakmp sa
```

> 실제 사용 가능한 알고리즘과 DH Group은 IOS 버전·License·플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer 등 학습 도구는 IPsec/DMVPN 지원이 제한적일 수 있어 일부 명령이 동작하지 않을 수 있습니다.

---

### Q9. AH와 ESP는 무엇이 다른가?

👉 **핵심 답변:** AH(Protocol 51)는 **암호화를 제공하지 않고 인증·무결성만** 제공하며 IP Header까지 무결성 범위에 포함하므로 **NAT 환경에서 통과가 곤란**합니다. ESP(Protocol 50)는 암호화와 인증을 함께 제공하고 NAT-T와 조합해 NAT를 통과할 수 있어 실무 표준입니다.

| 항목 | AH (51) | ESP (50) |
|---|:---:|:---:|
| 암호화 | X | O |
| 무결성 | O | O |
| 인증 | O | O |
| IP Header 무결성 포함 | O | X |
| NAT 통과 | 곤란 | NAT-T로 가능 |
| 실무 사용 | 드묾 | 표준 |

```properties
AH  = 인증 전용, 암호화 없음, NAT가 IP Header를 변경하면 검증 실패
ESP = 암호화 + 인증, NAT-T(UDP 4500) 캡슐화로 NAT 통과
```

> "AH는 암호화를 제공한다"는 진술은 오답입니다. 기밀성이 필요하면 ESP를 사용해야 합니다.

---

### Q10. Transport Mode와 Tunnel Mode는 무엇이 다른가?

👉 **핵심 답변:** Transport Mode는 **원본 IP Header를 유지**하고 Payload부터 보호하며, Tunnel Mode는 **원본 패킷 전체를 감싸고 New IP Header를 추가**합니다.

```properties
Transport Mode : [Original IP] [ESP] [TCP/UDP + Data]
Tunnel Mode    : [New IP] [ESP] [Original IP] [TCP/UDP + Data]
```

| 항목 | Transport Mode | Tunnel Mode |
|---|---|---|
| New IP Header | 추가하지 않음 | 추가함 |
| 오버헤드 | 상대적으로 작음 | 상대적으로 큼 |
| 대표 용도 | GRE-over-IPsec | 순수 Site-to-Site |
| 원본 IP 은닉 | 안 됨 | 됨 |

```cisco
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode transport
```

검증:

```cisco
show crypto ipsec transform-set
show crypto ipsec sa
```

GRE-over-IPsec은 GRE가 이미 New IP Header를 만들었으므로 Transport Mode가 유리하고, GRE 없이 사설망을 연결하면 Tunnel Mode가 필요합니다.

---

### Q11. IKE Phase 1과 Phase 2는 무엇을 협상하는가?

👉 **핵심 답변:** Phase 1은 Peer를 인증하고 **안전한 관리 채널(ISAKMP SA)** 을 만들고, Phase 2는 그 채널 위에서 **실제 데이터를 보호할 IPsec SA**를 협상합니다.

| 구분 | 협상 대상 | Mode | 결과물 |
|---|---|---|---|
| Phase 1 | 암호화·해시·인증·DH Group·Lifetime | Main / Aggressive | ISAKMP SA |
| Phase 2 | Transform-set, Crypto ACL, PFS, Lifetime | Quick | IPsec SA |

```cisco
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
!
crypto map <NAME> 10 ipsec-isakmp
 set peer <PEER-IP>
 set transform-set <NAME>
 set pfs group14
 match address <ACL-NUMBER>
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show crypto map
show crypto session
```

| State | 의미 |
|---|---|
| MM_NO_STATE | Main Mode 초기 상태 또는 협상 실패 |
| MM_KEY_EXCH | 키 교환 진행 중, PSK 불일치 시 정지 |
| QM_IDLE | Phase 1 완료, Quick Mode 대기(정상) |

```properties
UDP 500  = IKE 협상
UDP 4500 = NAT-T 캡슐화
경로상 방화벽·ACL에서 위 포트와 ESP(50)를 허용해야 함
```

IKEv2는 협상 메시지 수가 줄고 EAP 인증과 비대칭 인증을 지원하며 DoS 완화 설계가 반영되어 있습니다. 명령 체계는 `crypto ikev2 proposal` / `policy` / `profile`로 분리되며 지원 여부는 IOS 버전·License·플랫폼별 확인이 필요합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q12. Crypto ACL은 왜 양단이 미러여야 하는가?

👉 **핵심 답변:** Crypto ACL은 보호 대상(관심 트래픽)을 정의하는 Proxy Identity이며, 양단에서 **정확히 반대 방향으로 일치**해야 IPsec SA 협상이 성립합니다.

```properties
본사 : permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
지사 : permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
        ↓
Mirror(대칭) 관계
```

비대칭이면 Phase 1은 QM_IDLE인데 Phase 2가 형성되지 않고, `show crypto ipsec sa`의 encaps만 증가하며 특정 대역만 통신되는 증상이 나타납니다.

검증:

```cisco
show crypto map
show access-lists <ACL-NUMBER>
show crypto ipsec sa
```

> Crypto ACL은 `permit`으로 보호 대상만 정의합니다. 일반 필터링은 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리의 Extended ACL로 분리하십시오.

---

### Q13. NAT 예외 처리와 NAT-T는 왜 필요한가?

👉 **핵심 답변:** VPN 보호 대상이 NAT로 변환되면 Crypto ACL과 매칭되지 않아 VPN이 성립하지 않으므로 NAT ACL에서 먼저 `deny`로 제외해야 합니다. NAT-T는 별개로 **경로상 NAT 장비가 있을 때 ESP를 UDP 4500으로 캡슐화**해 통과시키는 기능입니다.

```cisco
ip access-list extended NAT_EXCEPT
 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
!
ip nat inside source list NAT_EXCEPT interface <INTERFACE> overload
```

검증:

```cisco
show access-lists NAT_EXCEPT
show ip nat translations
show crypto ipsec sa
```

```properties
[1] VPN 대상 deny (NAT 제외)
[2] 나머지 permit (인터넷용 변환)
순서가 뒤바뀌면 VPN 트래픽이 변환되어 협상 실패 또는 간헐 단절

NAT-T = ESP를 UDP 4500으로 캡슐화, 약 8 Byte 추가 오버헤드
```

NAT 동작은 5-2. 🔀 NAT & PAT (주소 변환), 내부 주소 할당은 5-1. 💾 DHCP (동적 IP 할당)을 함께 확인하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q14. CBAC은 무엇이고 VPN과 어떻게 조합하는가?

👉 **핵심 답변:** CBAC은 세션 상태를 State Table에 기록해 **내부에서 시작한 세션의 응답만 동적으로 허용**하는 IOS Stateful Firewall입니다. VPN과 함께 사용할 때는 Inbound ACL에서 IKE·ESP·GRE를 허용해야 VPN이 성립합니다.

```cisco
ip inspect name <NAME> tcp
ip inspect name <NAME> udp
ip inspect name <NAME> icmp
!
ip access-list extended ACL_IN
 permit udp host <PEER-IP> eq 500 any eq 500
 permit udp host <PEER-IP> eq 4500 any eq 4500
 permit esp host <PEER-IP> any
 permit gre host <PEER-IP> any
 deny   ip any any log-input
!
interface <INTERFACE>
 ip access-group ACL_IN in
 ip inspect <NAME> out
```

검증:

```cisco
show ip inspect config
show ip inspect interfaces
show ip inspect session
show ip access-lists ACL_IN
```

```properties
ACL           = 상태를 모르는 정적 필터
Reflexive ACL = L3/L4 역방향 미러링
CBAC          = 세션 상태 추적 + Application 계층 검사 + 임계치 기반 방어
ZBF           = Zone 기반 정책 모델, CBAC의 후속 권장 방식
```

CBAC은 레거시 방식이며 신규 설계는 ZBF(`class-map type inspect` / `policy-map type inspect` / `zone-pair`)를 검토하십시오. 지원 여부는 IOS 버전·License·플랫폼별 확인이 필요합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q15. GRE-over-IPsec은 왜 필요한가?

👉 **핵심 답변:** 순수 IPsec은 Unicast만 보호하므로 **Multicast 기반 동적 라우팅을 직접 보호할 수 없습니다.** GRE가 Multicast를 Unicast로 캡슐화하고 IPsec이 그 GRE를 암호화하면 라우팅까지 보호됩니다.

```properties
Crypto ACL에 OSPF(224.0.0.5) 또는 EIGRP(224.0.0.10)를 직접 매칭
        ↓
IPsec이 Multicast를 처리하지 못함
        ↓
%CRYPTO-4-RECVD_PKT_NOT_IPSEC 및 인접 단절

해결 : Crypto ACL을 GRE(Protocol 47) 전체로 지정
```

```cisco
access-list <ACL-NUMBER> permit gre host <IP-ADDRESS> host <PEER-IP>
!
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto map <NAME> 10 ipsec-isakmp
 set peer <PEER-IP>
 set transform-set <NAME>
 match address <ACL-NUMBER>
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show crypto map
show ip eigrp neighbors
show ip route eigrp
```

| 적용 방식 | 캡슐화 구조 | New IP Header |
|---|---|:---:|
| Tunnel IF + Tunnel Mode | `[New IP][GRE][New IP][ESP][Original]` | 2개 |
| 물리 IF + Tunnel Mode | `[New IP][ESP][New IP][GRE][Original]` | 2개 |
| 물리 IF + Transport Mode | `[New IP][ESP][GRE][Original]` | 1개 |

물리 인터페이스 적용 + Transport Mode가 오버헤드 관점에서 유리하지만, GRE Header가 추가되는 것은 동일하므로 `ip mtu`와 `ip tcp adjust-mss` 조정은 여전히 필요합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q16. DMVPN Phase 1/2/3은 무엇이 다른가?

👉 **핵심 답변:** Phase 1은 Spoke 간 트래픽이 **항상 Hub를 경유**하고, Phase 2는 라우팅 Next-Hop이 상대 Spoke로 유지되어 **Spoke 간 직접 터널**이 형성되며, Phase 3은 라우팅 Next-Hop이 Hub여도 **NHRP Redirect/Shortcut**으로 직접 전달됩니다.

| 구분 | Routing Next-Hop | 데이터 경로 | 핵심 명령 | 경로 요약 |
|---|---|---|---|:---:|
| Phase 1 | Hub | Hub 경유 | `no ip split-horizon eigrp <AS>` | 가능 |
| Phase 2 | 상대 Spoke | Spoke 직접 | `no ip next-hop-self eigrp <AS>` 추가 | 어려움 |
| Phase 3 | Hub | Shortcut으로 직접 | Hub `ip nhrp redirect` / Spoke `ip nhrp shortcut` | 가능 |

Hub:

```cisco
crypto isakmp key 0 <PASSWORD> address 0.0.0.0 0.0.0.0
!
crypto ipsec profile <NAME>
 set transform-set <NAME>
!
interface Tunnel <TUNNEL-NUMBER>
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1234
 ip nhrp map multicast dynamic
 ip nhrp redirect
 no ip split-horizon eigrp 100
 tunnel protection ipsec profile <NAME>
```

Spoke:

```cisco
interface Tunnel <TUNNEL-NUMBER>
 tunnel mode gre multipoint
 ip nhrp network-id 1234
 ip nhrp nhs <IP-ADDRESS>
 ip nhrp map <IP-ADDRESS> <PEER-IP>
 ip nhrp map multicast <PEER-IP>
 ip nhrp shortcut
 tunnel protection ipsec profile <NAME>
```

검증:

```cisco
show ip nhrp
show dmvpn
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show ip route eigrp
```

| 항목 | Crypto Map | IPsec Profile |
|---|---|---|
| Peer 주소 | `set peer`로 사전 지정 | 사전 지정 불필요 |
| 적용 위치 | 물리 Interface | Tunnel Interface |
| 보호 대상 | Crypto ACL | 터널 전체 |
| 확장성 | Peer 증가 시 설정 증가 | Spoke 증가에 유리 |
| 대표 용도 | Site-to-Site, GRE-over-IPsec | DMVPN |

> Hub의 `address 0.0.0.0 0.0.0.0`은 주소가 유동적인 Spoke를 수용하기 위한 설정입니다. 충분히 복잡한 키를 사용하고 가능하면 인증서 기반 인증을 검토하십시오.

---

### Q17. IPsec 이중화는 어떻게 구성하는가?

👉 **핵심 답변:** 본사 라우터 2대가 공유하는 **HSRP Virtual IP를 VPN Peer로 사용**하고 `crypto map <NAME> redundancy <HSRP-GROUP>`으로 Crypto Map을 HSRP와 연동해 Active 장비가 IPsec SA를 대표 처리하도록 만듭니다.

```cisco
interface <INTERFACE>
 standby 2 ip <IP-ADDRESS>
 standby 2 priority 120
 standby 2 preempt delay minimum 60
 standby 2 track <INTERFACE> 100
 standby 2 name IPSEC_R
!
crypto isakmp keepalive 10 3
!
interface <INTERFACE>
 crypto map <NAME> redundancy IPSEC_R
```

검증:

```cisco
show standby brief
show standby all
show crypto isakmp sa
show crypto isakmp peer
show crypto ipsec sa
show crypto session
```

- HSRP Group에 `name`을 지정하지 않으면 `redundancy` 연동을 사용할 수 없습니다.
- Standby 장비에 SA가 보이지 않는 것은 정상입니다.
- HSRP는 자기 인터페이스 장애만 감지하므로 내부/외부를 **교차 Track**해야 합니다.
- 원격지의 `set peer`와 `crypto isakmp key ... address`는 모두 Virtual IP로 통일합니다.

```properties
재수렴 시간 구성 요소
[1] HSRP 절체 감지 (Hello/Hold Timer)
[2] Track 반영 및 Priority 재계산
[3] 기존 SA 정리 (DPD 또는 Lifetime)
[4] Phase 1 / Phase 2 재협상
[5] 라우팅 재수렴
```

Gateway 이중화 동작은 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)를 참고하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 2. 기술 비교표

### 2-1. VPN 방식 종합 비교

| 항목 | GRE 단독 | IPsec 단독 | GRE-over-IPsec | DMVPN |
|---|:---:|:---:|:---:|:---:|
| 암호화 | X | O | O | O |
| Multicast 전달 | O | X | O | O |
| 동적 라우팅 | O | 직접 불가 | O | O |
| Non-IP Protocol | O | X | O | O |
| 확장성 | 낮음(P2P) | 낮음(Peer별) | 중간 | 높음 |
| Spoke-to-Spoke 직접 | X | X | X | Phase 2/3에서 O |
| Peer 주소 사전 필요 | O | O | O | Spoke는 불필요 |
| 오버헤드 | 작음 | 중간 | 큼 | 큼 |
| 대표 적용 | 임시 경로 | 소수 Site | 라우팅 암호화 | 대규모 Hub-Spoke |

### 2-2. 캡슐화 순서와 보호 범위

```properties
Original IP Packet
        ↓
GRE Header 추가        (라우팅·Multicast 전달)
        ↓
IPsec ESP 처리         (암호화 및 무결성)
        ↓
New IP Header 추가     (공중망 라우팅)
        ↓
물리 Interface 전송

GRE 단독                  : [New IP][GRE][Original IP][Data]  → 보호 없음
IPsec Tunnel Mode 단독    : [New IP][ESP][Original IP][Data]  → Unicast만 보호
GRE-over-IPsec(Transport) : [New IP][ESP][GRE][Original IP]   → 라우팅까지 보호
DMVPN                     : [New IP][ESP][mGRE][Original IP]  → 터널 전체 보호
```

복호화는 역순이므로 어느 단계에서 실패했는지 판단할 때 이 순서를 기준으로 접근합니다.

### 2-3. 오버헤드 개념

| 구성 요소 | 대략적인 크기 | 비고 |
|---|---|---|
| New IP Header | 20 Byte | IPv4 기준 |
| GRE Header | 4 Byte 이상 | Key/Sequence 옵션 시 증가 |
| ESP Header + Trailer | 알고리즘 의존 | IV, Padding, ICV 포함 |
| NAT-T UDP Header | 8 Byte | NAT-T 사용 시 |

```properties
GRE 단독          ≈ 24 Byte 감소
GRE + ESP         ≈ 수십 Byte 추가 감소
GRE + ESP + NAT-T ≈ 8 Byte 더 감소
        ↓
ip mtu 1400 / ip tcp adjust-mss 1360은 여유를 둔 보수적 기준값
```

### 2-4. 프로토콜 및 포트

| 대상 | 프로토콜/포트 | 비고 |
|---|---|---|
| GRE | IP Protocol 47 | 경로상 허용 필요 |
| ESP | IP Protocol 50 | 암호화 데이터 |
| AH | IP Protocol 51 | NAT 통과 곤란 |
| IKE | UDP 500 | Phase 1/2 협상 |
| NAT-T | UDP 4500 | NAT 환경 캡슐화 |

---

## 3. 방식 선택 가이드

```properties
암호화 불필요, 단순 논리 경로만 필요   → GRE 단독 (공중망에서는 권장하지 않음)
소수 Site + Unicast만 보호            → IPsec 단독 (Tunnel Mode)
동적 라우팅 또는 Multicast 암호화      → GRE-over-IPsec (물리 IF + Transport)
지사 다수, Hub 설정 증가 회피          → DMVPN Phase 1
Spoke 간 트래픽 비중이 큼              → DMVPN Phase 2 또는 Phase 3
본사 VPN 단일 장애점 제거              → IPsec 이중화 (HSRP + redundancy)
```

---

## 4. 안전한 변경 절차

```properties
[1] 현재 확인 : 라우팅·Tunnel·Crypto·NAT·ACL 상태 출력 저장
[2] 백업      : show running-config 보관, 변경 항목 문서화
[3] 경로 확인 : Underlay 도달성과 OOB 관리 경로 확보
[4] 적용      : 최소 단위로 순차 적용
[5] 검증      : show 명령으로 실제 상태 확인
[6] 저장      : copy running-config startup-config
```

변경 전:

```cisco
show running-config
show ip route
show interface tunnel <TUNNEL-NUMBER>
show crypto isakmp sa
show crypto ipsec sa
show crypto map
show ip nat translations
show access-lists
```

변경 후:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip nhrp
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show ip route
show ip eigrp neighbors
```

Rollback 절차:

```properties
[1] 적용 전 running-config를 외부에 보관
[2] 인터페이스에서 no crypto map <NAME>으로 제거 후 원래 설정 복원
[3] Tunnel은 shutdown 후 파라미터 복원
[4] ACL / NAT는 이전 내용을 그대로 재구성
[5] 추가한 Static Route와 Distribute-List 제거
[6] show 명령으로 원상 복귀 확인
[7] 정상 확인 전에는 startup-config에 저장하지 않음
```

```cisco
reload in 10
! 변경 및 검증 수행
reload cancel
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 5. 검증 명령어

```cisco
! Tunnel
show interface tunnel <TUNNEL-NUMBER>
show ip interface brief
!
! NHRP / DMVPN
show ip nhrp
show ip nhrp nhs detail
show dmvpn
!
! IPsec
show crypto isakmp sa
show crypto isakmp policy
show crypto isakmp peer
show crypto ipsec sa
show crypto ipsec transform-set
show crypto map
show crypto session
show crypto engine connections active
!
! 라우팅 / NAT / 이중화
show ip route <PEER-IP>
show ip eigrp neighbors
show ip ospf neighbor
show ip nat translations
show access-lists <ACL-NUMBER>
show standby brief
!
! Debug
debug crypto isakmp
debug crypto ipsec
undebug all
```

> ⚠️ `debug` 계열은 운영 장비에서 CPU 부하와 Console 폭주를 유발할 수 있습니다. 대상 범위를 좁히고 확인 후 즉시 해제하십시오.

---

## 6. 실무 체크리스트

- [ ] GRE만 사용하는 구간에 암호화가 필요하지 않은지 검토했는가?
- [ ] Tunnel destination이 Underlay 라우팅으로 도달 가능한가?
- [ ] Overlay 라우팅이 Tunnel Endpoint 대역을 광고하지 않는가?
- [ ] `ip mtu`와 `ip tcp adjust-mss`를 설정했는가?
- [ ] Hub 터널에서 Split Horizon 또는 OSPF Network Type을 처리했는가?
- [ ] mGRE에서 `tunnel destination`을 잘못 남겨두지 않았는가?
- [ ] NHRP network-id와 authentication이 Hub/Spoke에서 일치하는가?
- [ ] Phase 1 파라미터가 양단에서 완전히 일치하는가?
- [ ] Transform-set과 Mode가 양단에서 일치하는가?
- [ ] Crypto ACL이 양단에서 미러 관계인가?
- [ ] Crypto Map을 의도한 인터페이스에 적용했는가?
- [ ] NAT ACL에서 VPN 대상을 먼저 `deny`로 제외했는가?
- [ ] 경로상 UDP 500, UDP 4500, ESP, GRE가 허용되는가?
- [ ] CBAC 또는 ZBF Inbound 정책에서 IKE/ESP/GRE를 허용했는가?
- [ ] DMVPN에서 Crypto Map 대신 IPsec Profile을 사용했는가?
- [ ] IPsec 이중화에서 HSRP `name`과 `redundancy` 연동을 적용했는가?
- [ ] 변경 전 설정과 상태 출력을 백업했는가?
- [ ] Rollback 절차와 OOB 접근 수단을 확보했는가?

---

> 📌 **핵심 요약**
> - GRE는 캡슐화 전용이며 암호화를 제공하지 않는다.
> - GRE Tunnel은 destination이 라우팅 테이블에서 도달 가능해야 Up/Up이 된다.
> - Recursive Routing은 Longest Prefix Match → AD → Metric 순서로 터널 경로가 선택될 때 발생한다.
> - 캡슐화 오버헤드 때문에 `ip mtu 1400`과 `ip tcp adjust-mss 1360` 조정이 필요하다.
> - AH는 암호화를 제공하지 않고 NAT 통과가 곤란하므로 실무는 ESP를 사용한다.
> - Transport Mode는 원본 IP Header를 유지하고 Tunnel Mode는 New IP Header를 추가한다.
> - Phase 1은 ISAKMP SA를, Phase 2는 IPsec SA를 협상한다.
> - Crypto ACL은 양단이 반드시 미러(대칭)여야 한다.
> - VPN 대상 트래픽은 NAT ACL에서 먼저 `deny`로 제외해야 한다.
> - NAT-T는 ESP를 UDP 4500으로 캡슐화해 NAT를 통과시킨다.
> - 순수 IPsec은 Multicast 라우팅을 직접 보호하지 못하므로 GRE-over-IPsec을 사용한다.
> - GRE-over-IPsec의 Crypto ACL은 GRE(Protocol 47)를 대상으로 지정한다.
> - DMVPN은 mGRE + NHRP + IPsec Profile + 동적 라우팅의 결합이다.
> - DMVPN Phase 1은 Hub 경유, Phase 2는 Spoke 직접, Phase 3은 Redirect/Shortcut으로 직접 전달한다.
> - IPsec 이중화는 HSRP Virtual IP를 Peer로 쓰고 `crypto map <NAME> redundancy <HSRP-GROUP>`으로 연동한다.
> - CBAC은 레거시 Stateful Firewall이며 신규 설계는 ZBF를 검토한다.
> - 알고리즘은 AES/SHA-2 계열을 권장하되 지원 여부는 IOS 버전·License·플랫폼별 확인이 필요하다.
> - 관련: 5-3-1-1. 🚇 GRE Static · 5-3-1-2. 🔄 GRE Dynamic · 5-3-1-3. 🌐 mGRE & NHRP · 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE) · 5-3-3. 🛡️ CBAC (Stateful Firewall) · 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화) · 5-3-5. 🌐 DMVPN (동적 멀티포인트 VPN) · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동) · 5-3-7. 🧩 VPN 통합 정리 (GRE · IPsec · DMVPN) · 5-3-8. 🚑 VPN 트러블슈팅 치트시트 · 5-3-9. ⚡ VPN 명령어 퀵 레퍼런스
