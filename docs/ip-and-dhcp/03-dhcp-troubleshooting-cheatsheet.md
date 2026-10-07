# DHCP 트러블슈팅 치트시트

> **Tag:** #Network #DHCP #IPServices #Relay #Cisco  
> **핵심 요약:** DHCP 장애는 **Port·VLAN 상태 → SVI Up → Client와 Server가 같은 Segment인지 → Relay(helper-address)와 giaddr → 양방향 Routing → ACL(UDP 67/68) → Pool 대역·가용성 → excluded-address → Conflict → default-router/DNS** 순서로 확인한다. 단말이 169.254.x.x(APIPA)를 받았다면 Server 응답이 도달하지 못한 것이고, IP는 받았는데 통신이 안 된다면 DHCP가 아니라 `default-router`·DNS·Routing 문제일 가능성이 높다.

---

## 1. 공통 진단 순서

```properties
[1]  단말 Port가 Up이고 올바른 VLAN에 속하는가?
[2]  해당 VLAN이 존재하며 SVI가 Up인가?
[3]  Client와 DHCP Server가 같은 Segment인가?
[4]  다른 Segment면 ip helper-address가 Client 쪽 Interface에 있는가?
[5]  Relay Agent에서 Server IP로 Routing 경로가 있는가?
[6]  Server에서 giaddr 대역으로 되돌아오는 경로가 있는가?
[7]  경로상 ACL이 UDP 67/68을 차단하지 않는가?
[8]  Server에 giaddr 대역과 일치하는 Pool이 있는가?
[9]  Pool 주소가 고갈되지 않았는가?
[10] excluded-address가 Gateway·Server·Static을 제외했는가?
[11] show ip dhcp conflict에 기록이 있는가?
[12] Client가 받은 default-router와 DNS가 올바른가?
```

```properties
IP를 아예 못 받음        → [1]~[9] 구간 문제
IP는 받지만 통신 불가    → [12] 및 Routing/ACL 문제
간헐적으로만 실패        → Pool 고갈, 중복 Server, Lease 설계 문제
특정 단말만 실패         → Static Binding 매칭, Port 보안, MAC 관련 문제
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 단말이 169.254.x.x를 받음 | Server 응답 미도달 | `show ip dhcp pool` | 같은 Segment면 Pool 확인, 다른 Segment면 helper-address 설정 |
| Binding에 아무것도 없음 | 요청이 Server에 미도달 | `show ip dhcp server statistics` | Relay·Routing·ACL 경로 점검 |
| Binding은 있는데 통신 불가 | `default-router` 오류 | `show dhcp lease` | Gateway IP와 일치하도록 수정 |
| Gateway IP가 단말에 임대됨 | `excluded-address` 누락 | `show ip dhcp binding` | 제외 추가 후 `clear ip dhcp binding` |
| Conflict 기록 증가 | 중복 주소 또는 Static 미제외 | `show ip dhcp conflict` | 원인 제거 후 `clear ip dhcp conflict *` |
| 일부 단말만 IP 미수신 | Pool 고갈 | `show ip dhcp pool` | 대역 확장 또는 Lease 단축 |
| VLAN 전체가 실패 | SVI Down | `show ip interface brief`, `show vlan brief` | VLAN 활성 Port 및 `no shutdown` 확인 |
| 다른 Subnet의 IP를 받음 | giaddr와 Pool 대역 불일치 | `show running-config \| include helper` | Relay Interface IP와 Pool `network` 정렬 |
| 예상치 못한 대역 수신 | 비인가 DHCP Server | `show ip dhcp snooping binding` | Snooping 적용 및 비신뢰 Port 차단 |
| Static Binding 미적용 | 식별자 방식 불일치 | `debug ip dhcp server events` | `hardware-address`/`client-identifier` 재확인 |
| Relay 설정 후에도 실패 | 반환 경로 또는 ACL | `show ip route`, `show access-lists` | 양방향 경로와 UDP 67/68 허용 확인 |
| 갱신 시점에 끊김 | T1 Unicast 차단 또는 Server 재부팅 | `show dhcp lease`, `show ip dhcp database` | ACL 허용, Binding 저장 검토 |

---

## 3. 핵심 진단 명령어

```cisco
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp server statistics
show ip dhcp database
show running-config | section ip dhcp
!
show ip interface brief
show dhcp lease
!
show ip interface <INTERFACE> | include Helper
show running-config | include helper-address
show running-config | include forward-protocol
show ip route <IP-ADDRESS>
show access-lists
!
show interfaces status
show vlan brief
show interfaces vlan <VLAN-ID>
show mac address-table vlan <VLAN-ID>
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

> `debug ip dhcp server events`와 `debug ip dhcp server packet`은 CPU와 Console 부하를 유발합니다. 운영망에서는 필요한 구간만 짧게 활성화하고 즉시 `undebug all`로 종료하십시오. Packet Tracer의 debug 출력은 실장비와 형식·상세도가 다를 수 있습니다.

---

## 4. 대표 장애 시나리오

### 시나리오 1. 단말이 IP를 받지 못하고 169.254.x.x를 사용함

**증상:** Client가 APIPA 주소를 자동 설정합니다. Server 응답이 도달하지 않았다는 신호입니다.

**원인 후보:** 다른 Segment인데 `ip helper-address` 누락, SVI Down, Pool `network` 불일치, Pool 고갈, ACL의 UDP 67/68 차단, Client Port의 VLAN 오류, Server Down 또는 Pool 미설정.

```cisco
show interfaces status
show vlan brief
show ip interface brief
show ip dhcp pool
show ip dhcp binding
show ip interface <INTERFACE> | include Helper
```

**조치:** 같은 Segment면 Pool을, 다른 Segment면 Relay를 보완합니다.

```cisco
configure terminal
ip dhcp excluded-address 192.168.10.254
!
ip dhcp pool USERS_VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 192.168.10.252
 lease 3 0 0
!
interface <INTERFACE>
 ip helper-address <IP-ADDRESS>
end
!
show ip dhcp pool
show ip dhcp binding
show ip interface <INTERFACE> | include Helper
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 2. IP는 받았지만 외부 통신이 되지 않음

**핵심 판단:** DHCP 임대 성공은 Routing 성공을 의미하지 않습니다.

```properties
Binding 존재
≠ Gateway 정상  ≠ DNS 정상
≠ Routing 경로 존재  ≠ ACL 허용
```

**원인 후보:** Pool의 `default-router`가 실제 Gateway IP와 다름, Gateway Interface·SVI Down, 배포된 Subnet Mask 오류, 상위 Routing 또는 주소 변환 미설정, ACL이 사용자 대역 Outbound 차단.

```cisco
show ip dhcp binding
show running-config | section ip dhcp pool
show dhcp lease
show ip route
show access-lists
!
configure terminal
ip dhcp pool USERS_VLAN10
 default-router 192.168.10.254
 dns-server 192.168.10.252 192.168.10.253
end
!
clear ip dhcp binding *
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

기존 임대는 잘못된 Gateway를 그대로 사용하므로 재획득이 필요합니다. 외부 통신 자체가 주소 변환 문제라면 5-2. 🔀 NAT & PAT (주소 변환)를 함께 확인합니다.

---

### 시나리오 3. Gateway IP가 단말에 임대되어 Segment 전체 장애 발생

**원인:** Pool의 `network`가 Subnet 전체를 임대 후보로 삼는데 Gateway IP를 제외하지 않았습니다.

```properties
network 192.168.10.0 255.255.255.0
= .1 ~ .254 전체가 임대 후보

excluded-address 누락
→ Gateway(.254)까지 임대 가능
→ 중복 주소 및 Segment 전체 영향
```

```cisco
show ip dhcp binding
show ip dhcp conflict
show running-config | include excluded-address
!
configure terminal
ip dhcp excluded-address 192.168.10.250 192.168.10.254
end
!
clear ip dhcp binding 192.168.10.254
clear ip dhcp conflict *
!
show ip dhcp binding
show ip dhcp conflict
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 4. Relay를 설정했는데도 IP를 받지 못함

**원인 후보:** `ip helper-address`를 Server 쪽 Interface에 설정(Client 쪽이어야 함), Relay Agent에서 Server IP로 경로 없음, Server에서 giaddr 대역으로 되돌아오는 경로 없음, 경로상 ACL의 UDP 67/68 차단, giaddr 대역과 일치하는 Pool 없음, Relay Interface에 IP가 없어 giaddr을 채울 수 없음.

**판정 로직**

```properties
Server statistics의 요청 Counter 증가 없음
→ 요청이 Server에 미도달 → Relay 설정, Routing, ACL 점검

요청 Counter 증가 있으나 Binding 없음
→ Server가 Pool을 못 찾음 → giaddr 대역과 Pool network 불일치 점검

Binding 생성되나 Client 미수신
→ 응답 반환 경로 문제 → Server → giaddr 경로 및 ACL 점검
```

**정상 예시 (위: Relay Agent / 아래: Server)**

```cisco
configure terminal
interface Vlan10
 ip address 192.168.10.254 255.255.255.0
 ip helper-address 10.10.10.1
 no shutdown
!
ip dhcp excluded-address 192.168.10.254
!
ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 10.10.10.53
end
!
show ip interface Vlan10 | include Helper
show ip route 10.10.10.1
show ip dhcp server statistics
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 5. 단말이 엉뚱한 Subnet의 IP를 받음

**원인:** Server는 giaddr 기준으로 Pool을 선택하므로, Relay Interface IP와 Pool의 `network` 대역이 정렬되지 않으면 다른 Pool이 선택됩니다.

```properties
giaddr 192.168.20.254
Server에 192.168.20.0/24 Pool 정의 없음
→ 다른 Pool 또는 상위 Subnet Pool이 선택될 수 있음
→ 단말이 잘못된 대역 IP 수신
```

```cisco
show running-config | include helper-address
show ip interface brief
show running-config | section ip dhcp pool
!
configure terminal
ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.254
end
!
show ip dhcp pool
show ip dhcp binding
```

> 여러 Interface에 동일한 helper-address를 설정한 환경에서는 각 Interface IP 대역에 대응하는 Pool이 모두 존재해야 합니다.

---

### 시나리오 6. Pool 주소가 고갈되어 일부 단말만 실패함

**원인 후보:** 단말 수가 Pool 크기 초과, Lease Time이 길어 회수 지연, Conflict 상태 IP 누적, excluded 범위 과다, 사라진 단말의 임대 잔존.

```properties
show ip dhcp pool 확인 항목
= Total addresses / Leased addresses
= Excluded addresses / Pending event
```

```cisco
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
!
configure terminal
ip dhcp pool USERS_VLAN10
 lease 0 8 0
end
!
clear ip dhcp conflict *
show ip dhcp pool
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

대역 자체가 부족하면 Lease 단축이 아니라 Subnet 재설계를 검토해야 합니다.

---

### 시나리오 7. Conflict 기록이 계속 증가함

**원인 후보:** Static 장비 주소가 Pool 범위에 포함되고 excluded 누락, 같은 Segment에 다른 DHCP Server 존재, 동일 대역을 두 Server가 중복 관리, 단말의 수동 IP 설정, 이전 임대 미정리.

```properties
[1] show ip dhcp conflict로 충돌 IP 목록 확보 → 실제 사용 장비 식별
[2] Static 장비면 excluded-address 추가, 비인가 Server면 Port 차단과 Snooping 검토
[3] clear ip dhcp conflict *로 기록 초기화 후 재발 여부 재확인
```

```cisco
show ip dhcp conflict
show running-config | include excluded-address
show running-config | include ip dhcp ping
!
configure terminal
ip dhcp excluded-address <IP-ADDRESS>
end
!
clear ip dhcp conflict *
show ip dhcp conflict
show ip dhcp binding
```

> Conflict 기록은 원인을 제거하지 않고 `clear`만 하면 반드시 재발합니다. Packet Tracer에서는 Conflict 기록과 Ping 검사 동작이 실장비와 다르게 나타날 수 있습니다.

---

### 시나리오 8. VLAN 전체가 IP를 받지 못함

**원인 후보:** VLAN 미존재, VLAN에 Up 상태 Access Port가 없어 SVI Down, SVI가 `shutdown`, Trunk Allowed VLAN 누락, Pool `network`가 SVI 대역과 불일치, `ip routing` 미설정.

```cisco
show vlan brief
show interfaces status
show ip interface brief
show interfaces vlan <VLAN-ID>
show ip dhcp pool
!
configure terminal
ip routing
!
vlan <VLAN-ID>
 name USERS
!
interface Vlan<VLAN-ID>
 ip address 192.168.30.254 255.255.255.0
 no shutdown
!
interface <INTERFACE>
 switchport mode access
 switchport access vlan <VLAN-ID>
end
!
show interfaces vlan <VLAN-ID>
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

VLAN·Trunk 조건은 1-2. 🚚 Switchport & Trunk & Inter-VLAN에서 확인합니다.

---

### 시나리오 9. Static Binding이 적용되지 않고 일반 IP를 받음

**원인 후보:** `hardware-address` 표기 형식 오류, 단말이 `client-identifier` 기반으로 요청, MAC 오입력, `host` IP가 다른 임대에 사용 중, `host` IP가 excluded 범위 포함, Static Pool Mask 불일치.

```properties
debug 출력에 Client Identifier가 01 + MAC 형식으로 보임
→ client-identifier 사용

debug 출력에 MAC만 보이거나 매칭 실패
→ hardware-address 표기 및 값 재확인
```

```cisco
show running-config | section ip dhcp pool
show ip dhcp binding
debug ip dhcp server events
!
configure terminal
ip dhcp pool PRINTER_01
 host 192.168.10.50 255.255.255.0
 client-identifier 0100.aabb.ccdd.ee
 default-router 192.168.10.254
end
!
show ip dhcp binding
undebug all
```

> `client-identifier` 표기 규칙과 매칭 우선순위는 IOS 버전에 따라 차이가 있으므로 플랫폼별 지원 여부 확인이 필요합니다.

---

### 시나리오 10. ACL 적용 후 DHCP가 동작하지 않음

**원인:** DHCP 초기 단계는 Src 0.0.0.0, Dst 255.255.255.255이며 UDP 67/68을 사용합니다. 사용자 대역 기준으로만 작성한 ACL에서는 이 트래픽이 걸러질 수 있습니다.

```properties
Discover / Request = Src 0.0.0.0:68 → Dst 255.255.255.255:67
Relay 이후 구간    = Src Relay Interface IP:67 → Dst Server IP:67
Offer / ACK        = Src Server IP:67 → Dst Client 또는 giaddr
```

```cisco
show access-lists
show ip interface <INTERFACE> | include access list
!
configure terminal
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
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 11. 예상하지 못한 대역의 IP가 배포됨 (비인가 Server 의심)

```properties
정상 Server 대역과 다른 IP 배포
+ Server 측 Binding에 해당 임대 없음
→ Segment 내 다른 DHCP 응답원이 존재할 가능성

Trusted Port   = Server 응답 허용 (Uplink, Server 연결 Port)
Untrusted Port = Server 응답 차단 (사용자 단말 Port)
```

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
show mac address-table vlan <VLAN-ID>
show interfaces status
!
configure terminal
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

> DHCP Snooping을 활성화하면서 Trusted Port를 지정하지 않으면 정상 Server 응답까지 차단됩니다. 세부 명령과 Rate Limit·Option 82 처리 방식은 플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer는 DHCP Snooping을 제한적으로만 구현하거나 미지원인 경우가 있습니다.

Port 단위 통제는 6-1. 🔐 Port-security & Storm-control와 함께 적용합니다.

---

### 시나리오 12. Lease 갱신 시점에 통신이 끊김

**원인 후보:** T1 Unicast 갱신 경로가 ACL로 차단됨, 원 Server Down으로 T2 Broadcast 응답 없음, Server 재부팅으로 Binding 소실, Binding Database 저장 미설정.

```properties
T1(50%) 실패 후 T2(87.5%) 성공 → 원 Server 도달 문제 (Unicast 경로 또는 ACL)
T2에서도 실패하고 만료 후 재획득 → Server 자체 또는 Segment 전반 문제
갱신 후 IP가 바뀜 → Server Binding 소실 또는 Pool 재설계 영향
```

```cisco
show dhcp lease
show ip dhcp binding
show ip dhcp database
show ip dhcp server statistics
!
configure terminal
ip dhcp database ftp://<IP-ADDRESS>/dhcp-binding
end
!
show ip dhcp database
show ip dhcp binding
```

> `ip dhcp database`의 지원 URL 형식과 저장 주기 옵션은 플랫폼별 지원 여부 확인이 필요합니다.

---

### 시나리오 13. helper-address 부작용과 `ip address dhcp` 관리 단절

`ip helper-address`는 DHCP 외에도 UDP 37, 49, 53, 67, 68, 69, 137, 138 등을 함께 Forwarding합니다. `ip address dhcp`는 기존 Static IP를 대체하므로 관리 세션이 즉시 끊길 수 있고 새로 받은 IP를 사전에 알 수 없습니다.

```cisco
show running-config | include forward-protocol
show ip interface <INTERFACE> | include Helper
!
configure terminal
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
!
configure terminal
interface <INTERFACE>
 no ip address dhcp
 ip address <IP-ADDRESS> 255.255.255.0
 no shutdown
end
!
show ip interface brief
show running-config interface <INTERFACE>
```

`ip address dhcp` Rollback은 Console에서 수행합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> 레거시 애플리케이션이 NetBIOS나 TFTP Broadcast에 의존할 수 있으므로 Forwarding 대상 변경 전에 실제 사용 여부를 확인하십시오.

---

## 5. 안전한 변경 및 복구 절차

```properties
[변경 절차]
[1] 현재 확인 : show ip dhcp pool / binding / conflict, show ip interface brief
[2] 백업      : show running-config 저장, 기존 binding 목록 저장
[3] 경로 확인 : Server IP까지의 Routing과 ACL, giaddr 대역 확인
[4] 적용      : excluded-address → Pool → helper-address 순서
[5] 검증      : 테스트 단말 1대로 실제 임대 및 통신 확인
[6] 저장      : copy running-config startup-config
```

```cisco
show running-config
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp server statistics
show ip interface brief
show ip route
!
copy running-config startup-config
```

```properties
[Rollback 절차]
[1] 변경 전 running-config 원문 확보 여부 확인
[2] helper-address 원복 : no ip helper-address <IP-ADDRESS>
[3] Pool 원복 : no ip dhcp pool <NAME> / excluded-address 원복
[4] ACL 원복 : no ip access-group <NAME> in
[5] 잘못 배포된 임대 정리 : clear ip dhcp binding *
[6] 테스트 단말로 재검증 후 저장
```

```cisco
configure terminal
interface <INTERFACE>
 no ip helper-address <IP-ADDRESS>
!
no ip dhcp pool USERS_VLAN10
end
!
clear ip dhcp binding <IP-ADDRESS>
show running-config | section ip dhcp
show ip dhcp binding
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `clear ip dhcp binding *`은 모든 임대 기록을 삭제하며 동일 IP 재할당을 보장하지 않습니다. 서비스 영향이 큰 시간대에는 특정 IP만 지정해 정리하는 방식을 우선 검토하십시오.

```properties
[긴급 격리 판단]
비인가 DHCP 응답 의심 → Port 식별 후 격리 → Snooping 적용 → 정상 대역 재배포
Pool 오설계로 대량 오배포 → 잘못된 Pool 즉시 제거 → excluded 보완 → 임대 정리 후 재배포
```

---

> 📌 **핵심 요약**
> - DHCP 장애는 Port·VLAN·SVI 상태부터 확인한다.
> - 169.254.x.x는 Server 응답이 도달하지 않았다는 신호다.
> - Client와 Server가 다른 Segment면 `ip helper-address`가 필수다.
> - `ip helper-address`는 Server 쪽이 아니라 Client 쪽 Interface에 설정한다.
> - Server는 giaddr로 Pool을 선택하므로 Relay Interface IP와 Pool 대역이 정렬돼야 한다.
> - Relay 환경에서는 반환 경로까지 양방향 Routing을 확인한다.
> - ACL은 UDP 67/68을 허용해야 한다.
> - Gateway·Server·Static 주소를 `excluded-address`로 제외하지 않으면 중복 주소 장애가 발생한다.
> - IP는 받았는데 통신이 안 되면 `default-router`와 DNS를 먼저 본다.
> - Pool 고갈은 `show ip dhcp pool`의 사용률로 판단한다.
> - Conflict는 원인 제거 후 `clear ip dhcp conflict *`로 정리한다.
> - `debug ip dhcp server events`는 운영 부하가 있으므로 짧게 사용하고 `undebug all`로 종료한다.
> - 비인가 Server 대응은 DHCP Snooping의 Trusted/Untrusted 구분으로 시작한다.
> - 변경은 현재 확인 → 백업 → 경로 확인 → 적용 → 검증 → 저장 순서를 지키고 Rollback을 준비한다.
> - 관련: 5-1. 💾 DHCP (동적 IP 할당) · 5-1-1. 🧩 DHCP 통합 정리 · 5-1-2. 🚑 DHCP 트러블슈팅 치트시트 · 5-1-3. ⚡ DHCP 명령어 퀵 레퍼런스 · 5-2. 🔀 NAT & PAT (주소 변환) · 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing
