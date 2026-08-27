# ⚡ DHCP 명령어 퀵 레퍼런스

> **Tag:** #Network #DHCP #IPServices #Relay #Cisco  
> **핵심 요약:** Cisco IOS DHCP Server·Relay·Client 설정과 검증 명령을 빠르게 확인하기 위한 암기 카드. Pool 정의 전에 `ip dhcp excluded-address`로 Gateway와 고정 IP를 제외하고, 다른 Subnet의 Server를 쓸 때는 Client 쪽 Interface에 `ip helper-address`를 설정한다는 두 가지가 실무의 핵심이다.

---

## 1. 🧠 요구사항별 명령 선택

| 요구사항 | 명령 또는 기능 |
|---|---|
| IP 임대 대역 정의 | `ip dhcp pool` + `network` |
| Gateway 배포 | `default-router <IP-ADDRESS>` |
| DNS 배포 | `dns-server <IP-ADDRESS>` |
| Domain Name 배포 | `domain-name <NAME>` |
| 임대 기간 지정 | `lease` |
| 임대 제외 주소 | `ip dhcp excluded-address` |
| 특정 단말 고정 IP | `host` + `hardware-address` 또는 `client-identifier` |
| 다른 Subnet Server 사용 | `ip helper-address <IP-ADDRESS>` |
| Relay 전달 UDP 조정 | `ip forward-protocol udp <PORT>` |
| Router를 Client로 동작 | `ip address dhcp` |
| 중복 주소 확인 | `show ip dhcp conflict` |
| 비인가 Server 방어 | `ip dhcp snooping` |

---

## 2. 📊 DORA 흐름과 Port

```properties
[D] Discover : Client → Broadcast (Src 0.0.0.0:68 / Dst 255.255.255.255:67)
[O] Offer    : Server → Client    (IP 제안, 중복 검사 후)
[R] Request  : Client → Broadcast (선택한 Server 공표)
[A] ACK      : Server → Client    (Lease 확정)
```

| 단계 | 방향 | Src IP | 방식 |
|---|---|---|---|
| Discover | Client → Server | 0.0.0.0 | Broadcast |
| Offer | Server → Client | Server IP | Broadcast 또는 Unicast |
| Request | Client → Server | 0.0.0.0 | Broadcast |
| ACK | Server → Client | Server IP | Broadcast 또는 Unicast |

| 구분 | Port |
|---|:---:|
| Server | UDP 67 |
| Client | UDP 68 |

---

## 3. 🔀 Relay 경로와 giaddr

```properties
Client → Relay  : Broadcast
Relay  → Server : Unicast (giaddr = 수신 Interface IP)
Server → Relay  : Unicast (Dst = giaddr)
Relay  → Client : Broadcast 또는 Unicast

giaddr 비어 있음 → Server는 자신과 같은 Segment 요청으로 판단
giaddr 채워짐    → 해당 값이 속한 Subnet의 Pool 선택
```

| 항목 | 값 |
|---|---|
| 설정 위치 | Client가 접속한 Segment의 Interface |
| giaddr 값 | 요청을 수신한 Relay Interface IP |
| Pool 선택 기준 | giaddr가 속한 Subnet |
| 필수 조건 | 양방향 Routing + UDP 67/68 허용 |

---

## 4. ⏱️ Lease 타이머

```properties
T1 = Lease의 50%   → Unicast 갱신 (Renewing, 원 Server 대상)
T2 = Lease의 87.5% → Broadcast 갱신 (Rebinding, 임의 Server 허용)
만료 = Lease 100%  → IP 반납 후 Discover부터 재시작
```

| 타이머 | 시점 | 전송 방식 |
|---|---|---|
| T1 | 50% | Unicast |
| T2 | 87.5% | Broadcast |
| Expire | 100% | Broadcast (DORA 재시작) |

---

## 5. 🔧 DHCP Server 기본 설정

```cisco
configure terminal
!
ip dhcp excluded-address 192.168.10.254
ip dhcp excluded-address 192.168.10.250 192.168.10.253
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
show running-config | section ip dhcp
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 6. 🚫 excluded-address

```cisco
configure terminal
!
ip dhcp excluded-address <IP-ADDRESS>
ip dhcp excluded-address 192.168.10.1 192.168.10.20
end
!
show running-config | include excluded-address
show ip dhcp pool
```

| 대상 | 이유 |
|---|---|
| Default Gateway | 중복 시 Segment 전체 영향 |
| DHCP Server 자신 | Server 자체 통신 불가 |
| DNS·NTP·인증 Server | 서비스 중단 |
| Printer·CCTV·AP | Static 장비 충돌 |
| HSRP/VRRP Virtual IP | Gateway 이중화 파손 |
| 예약 확장 대역 | 향후 설계 충돌 방지 |

```properties
excluded-address는 이미 임대된 IP를 자동 회수하지 않는다
→ 필요 시 clear ip dhcp binding으로 정리
```

---

## 7. ⏳ lease

```cisco
configure terminal
!
ip dhcp pool USERS_VLAN10
 lease 3 0 0
end
!
show ip dhcp pool
show running-config | section ip dhcp pool
```

| 명령 | 의미 |
|---|---|
| `lease 3` | 3일 |
| `lease 3 0 0` | 3일 0시간 0분 |
| `lease 0 12 0` | 12시간 |
| `lease 0 0 30` | 30분 |
| `lease infinite` | 무제한 |

> `lease infinite`는 Pool 주소가 회수되지 않습니다. 고정이 필요한 장비는 Static Binding 또는 장비 Static IP를 우선 검토하십시오.

---

## 8. 📌 Static Binding

```cisco
configure terminal
!
ip dhcp pool PRINTER_01
 host 192.168.10.50 255.255.255.0
 hardware-address 00aa.bbcc.ddee
 default-router 192.168.10.254
 dns-server 192.168.10.252
!
ip dhcp pool PRINTER_02
 host 192.168.10.51 255.255.255.0
 client-identifier 0100.aabb.ccdd.ef
 default-router 192.168.10.254
end
!
show ip dhcp binding
show running-config | section ip dhcp pool
```

```properties
hardware-address  = MAC 기반 매칭
client-identifier = Option 61 기반 매칭 (Ethernet은 통상 01 + MAC)
매칭 실패 시      = debug ip dhcp server events로 실제 수신 값 확인
```

> `client-identifier` 표기 규칙과 매칭 우선순위는 IOS 버전별 차이가 있어 플랫폼별 지원 여부 확인이 필요합니다.

---

## 9. 🔁 DHCP Relay

Router Interface:

```cisco
configure terminal
!
interface <INTERFACE>
 ip address 192.168.10.254 255.255.255.0
 ip helper-address <IP-ADDRESS>
 no shutdown
end
!
show ip interface <INTERFACE> | include Helper
show ip route <IP-ADDRESS>
show ip dhcp server statistics
```

L3 Switch SVI:

```cisco
configure terminal
!
ip routing
!
interface Vlan<VLAN-ID>
 ip address 192.168.20.254 255.255.255.0
 ip helper-address 10.10.10.1
 ip helper-address 10.10.10.2
 no shutdown
end
!
show ip interface vlan <VLAN-ID> | include Helper
show running-config | include helper-address
show interfaces vlan <VLAN-ID>
show ip dhcp binding
```

제거:

```cisco
configure terminal
!
interface <INTERFACE>
 no ip helper-address <IP-ADDRESS>
end
!
show running-config | include helper-address
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 10. 📡 helper-address 전달 UDP 조정

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
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> 기본 Forwarding 대상 목록과 동작은 IOS/IOS XE 버전에 따라 다를 수 있으므로 플랫폼별 지원 여부 확인이 필요합니다.

---

## 11. 💻 DHCP Client

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

Rollback:

```cisco
configure terminal
!
interface <INTERFACE>
 no ip address dhcp
 ip address <IP-ADDRESS> 255.255.255.0
end
!
show ip interface brief
show running-config interface <INTERFACE>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```properties
ip address dhcp
= 기존 Static IP 대체
= 받은 IP 예측 불가
= 관리 대상 Interface에는 신중 적용
```

---

## 12. 🏢 다중 VLAN 구성

L3 Switch가 Server 역할:

```cisco
configure terminal
!
ip routing
!
ip dhcp excluded-address 192.168.10.254
ip dhcp excluded-address 192.168.20.254
!
ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 168.126.63.1
 lease 3 0 0
!
ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.254
 dns-server 168.126.63.1
 lease 3 0 0
end
!
show ip dhcp pool
show ip dhcp binding
show ip interface brief
show vlan brief
```

L3 Switch가 Relay 역할:

```cisco
configure terminal
!
interface Vlan10
 ip helper-address 10.10.10.1
!
interface Vlan20
 ip helper-address 10.10.10.1
end
!
show running-config | include helper-address
show ip dhcp binding
```

```properties
Pool network        = 해당 VLAN Subnet과 동일
Pool default-router = 해당 VLAN SVI IP와 동일
SVI IP              = excluded-address 포함
SVI 상태            = Up 필수 (VLAN에 활성 Port 필요)
```

> VLAN은 Broadcast Domain 분리 수단이며 그 자체로 완전한 보안 경계가 아닙니다.

---

## 13. 🛡️ DHCP Snooping 기본

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

```properties
Trusted Port   = Server 응답(Offer/ACK) 허용
Untrusted Port = Server 응답 차단, Client 요청만 허용 (기본값)
Option 82      = 수신 장비/Port/VLAN 정보를 요청에 삽입
```

Option 82 관련 명령 예시:

```cisco
configure terminal
!
no ip dhcp snooping information option
!
ip dhcp relay information trust-all
end
!
show ip dhcp snooping
show running-config | include dhcp snooping
```

> DHCP Snooping과 Option 82 관련 명령의 문법·기본값·Rate Limit 옵션은 플랫폼과 IOS 버전 차이가 크므로 플랫폼별 지원 여부 확인이 필요합니다.

> Packet Tracer는 DHCP Snooping과 Option 82를 제한적으로만 구현하거나 미지원인 경우가 있습니다. 실습에서 동작하지 않아도 실장비와 다를 수 있음을 전제하십시오.

---

## 14. ⚔️ 중복 주소 관리

```cisco
show ip dhcp conflict
show ip dhcp binding
show ip dhcp pool
show running-config | include ip dhcp ping
!
clear ip dhcp conflict *
clear ip dhcp conflict <IP-ADDRESS>
!
clear ip dhcp binding *
clear ip dhcp binding <IP-ADDRESS>
!
show ip dhcp conflict
show ip dhcp binding
```

```properties
Conflict 발생 원인
= Ping/ARP 검사에서 응답 수신
= Client의 DHCP Decline 수신

원인 제거 없이 clear만 수행 → 반드시 재발
```

> `ip dhcp ping packets`와 `ip dhcp ping timeout`의 지원 여부와 기본값은 플랫폼별 지원 여부 확인이 필요합니다.

> `clear ip dhcp binding *`은 동일 IP 재할당을 보장하지 않으므로 서비스 영향을 검토한 뒤 실행하십시오.

---

## 15. 🔍 검증 명령어

Server 상태:

```cisco
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp server statistics
show ip dhcp database
```

Relay 및 경로:

```cisco
show ip interface <INTERFACE> | include Helper
show running-config | include helper-address
show running-config | include forward-protocol
show ip route
show access-lists
```

Client 및 L2:

```cisco
show ip interface brief
show dhcp lease
show vlan brief
show interfaces status
show interfaces vlan <VLAN-ID>
```

실시간 추적:

```cisco
debug ip dhcp server events
debug ip dhcp server packet
!
undebug all
```

> `debug ip dhcp server events`와 `debug ip dhcp server packet`은 CPU와 Console 부하를 유발합니다. 운영망에서는 짧게 사용하고 반드시 `undebug all`로 종료하십시오. Packet Tracer의 debug 출력 형식은 실장비와 다를 수 있습니다.

---

## 16. 🔒 ACL 환경에서의 DHCP 허용

```properties
Discover / Request = Src 0.0.0.0:68 → Dst 255.255.255.255:67
Relay 이후 구간    = Src Relay Interface IP:67 → Dst Server IP:67
Offer / ACK        = Src Server IP:67 → Dst Client 또는 giaddr
```

```cisco
configure terminal
!
ip access-list extended USER_IN
 permit udp any eq 68 any eq 67
 permit udp any eq 67 any eq 68
 permit ip 192.168.10.0 0.0.0.255 any
 deny ip any any
!
interface <INTERFACE>
 ip access-group USER_IN in
end
!
show access-lists USER_IN
show ip interface <INTERFACE> | include access list
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 17. 🔄 안전한 변경 원칙

```properties
[1] 현재 확인
[2] 백업
[3] 경로 확인 (Routing / ACL / giaddr 대역)
[4] 적용 (excluded-address → Pool → helper-address)
[5] 검증 (테스트 단말 1대)
[6] 저장
```

변경 전과 후:

```cisco
show running-config
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip interface brief
show ip route
!
copy running-config startup-config
```

Rollback:

```cisco
configure terminal
!
interface <INTERFACE>
 no ip helper-address <IP-ADDRESS>
!
no ip dhcp pool <NAME>
end
!
clear ip dhcp binding *
!
show running-config | section ip dhcp
show ip dhcp binding
```

---

## 18. ⚠️ 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| `network` 지정 | Subnet 전체가 임대 후보이며 Gateway도 포함된다 |
| `excluded-address` 미설정 | Gateway·Server IP가 단말에 임대될 수 있다 |
| `excluded-address` 추가 | 이미 임대된 IP를 자동 회수하지는 않는다 |
| `default-router` | Client에게 배포할 Gateway이며 실제 Gateway IP와 일치해야 한다 |
| Binding 존재 | 통신 성공을 의미하지 않는다 |
| 169.254.x.x | Server 응답 미도달 신호이며 Pool 문제로 단정할 수 없다 |
| `ip helper-address` 위치 | Server 쪽이 아니라 Client 쪽 Interface다 |
| `ip helper-address` 범위 | DHCP만이 아니라 여러 UDP 서비스를 함께 전달한다 |
| giaddr | Relay 수신 Interface IP이며 Pool 선택 기준이다 |
| Relay 설정 완료 | 반환 경로 Routing까지 보장하지는 않는다 |
| T1 갱신 | Unicast이므로 ACL에 막힐 수 있다 |
| T2 갱신 | Broadcast이므로 다른 Server가 응답할 수 있다 |
| `lease infinite` | IP 회수가 되지 않아 Pool 고갈 위험이 있다 |
| SVI 존재 | 활성 Port가 없으면 Down이며 Pool도 동작하지 않는다 |
| `clear ip dhcp conflict` | 원인을 제거하지 않으면 재발한다 |
| VLAN 분리 | Broadcast Domain 분리이며 완전한 보안 경계는 아니다 |
| DHCP Snooping 활성화 | Trusted Port 지정 없이는 정상 Server 응답도 차단된다 |
| Packet Tracer 동작 | Snooping·Option 82·debug 출력이 실장비와 다를 수 있다 |

---

## 19. 🚑 빠른 장애 진단 순서

```properties
Port Up 및 VLAN 확인
    ↓
VLAN 존재 및 SVI Up 확인
    ↓
Client와 Server가 같은 Segment인지 확인
    ↓
다른 Segment면 helper-address 확인
    ↓
Relay → Server Routing 확인
    ↓
Server → giaddr 반환 경로 확인
    ↓
ACL의 UDP 67/68 허용 확인
    ↓
giaddr 대역과 일치하는 Pool 확인
    ↓
Pool 가용 주소 확인
    ↓
excluded-address 확인
    ↓
show ip dhcp conflict 확인
    ↓
default-router 및 DNS 확인
```

```properties
IP 미수신        → Relay / Routing / ACL / Pool 구간
IP 수신 후 불통  → default-router / DNS / 상위 Routing 구간
간헐적 실패      → Pool 고갈 / 중복 Server / Lease 설계
특정 단말만 실패 → Static Binding 매칭 / Port 보안
```

---

> 📌 **핵심 요약**
> - Server Pool: `ip dhcp pool <NAME>` + `network <NETWORK> <MASK>`
> - Gateway 배포: `default-router <IP-ADDRESS>`
> - DNS 배포: `dns-server <IP-ADDRESS>` / Domain 배포: `domain-name <NAME>`
> - 임대 기간: `lease <DAY> <HOUR> <MINUTE>` 또는 `lease infinite`
> - 제외 주소: `ip dhcp excluded-address <IP-ADDRESS> [<IP-ADDRESS>]`
> - 고정 할당: `host` + `hardware-address` 또는 `client-identifier`
> - Relay: `ip helper-address <IP-ADDRESS>` (Client 쪽 Interface)
> - 전달 UDP 조정: `ip forward-protocol udp <PORT>`
> - Client 동작: `ip address dhcp`
> - 방어 기본: `ip dhcp snooping` + `ip dhcp snooping trust`
> - 임대 확인: `show ip dhcp binding` / 사용률 확인: `show ip dhcp pool`
> - 충돌 확인: `show ip dhcp conflict` / 정리는 `clear ip dhcp conflict *`
> - 통계 확인: `show ip dhcp server statistics`
> - Port는 Server UDP 67, Client UDP 68이며 Discover는 Broadcast, Relay 이후는 Unicast다.
> - 관련: 5-1. 💾 DHCP (동적 IP 할당) · 5-1-1. 🧩 DHCP 통합 정리 · 5-1-2. 🚑 DHCP 트러블슈팅 치트시트 · 5-1-3. ⚡ DHCP 명령어 퀵 레퍼런스 · 5-2. 🔀 NAT & PAT (주소 변환) · 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing
