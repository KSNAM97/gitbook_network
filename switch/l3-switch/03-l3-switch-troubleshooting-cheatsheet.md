# L3 Switch & Inter-VLAN Routing 트러블슈팅 치트시트

> **Tag:** #Network #Switch #L3Switch #InterVLAN #SVI #Cisco  
> **핵심 요약:** Inter-VLAN 통신 실패는 **단말 Gateway/Mask → VLAN 할당 → Trunk Allowed VLAN → SVI 상태 → `ip routing` → 라우팅 테이블 → ACL** 순서로 확인한다. SVI에 IP를 줬는데도 통신이 안 되면 대부분 `ip routing` 미활성이거나 SVI가 Up/Up이 아닌 경우다. 절반 이상은 L3가 아니라 L2 또는 단말 설정 문제다.

---

## 1. 공통 진단 순서

```properties
[1]  단말 IP / Subnet Mask / Default Gateway가 올바른가?
[2]  단말이 Gateway(SVI 또는 Virtual IP)에 Ping이 되는가?
[3]  단말 포트가 의도한 VLAN에 Access 할당되어 있는가?
[4]  해당 VLAN이 VLAN Database에 존재하고 Active인가?
[5]  Switch 간 Trunk가 Up이고 해당 VLAN이 Allowed 목록에 있는가?
[6]  양단 Native VLAN이 일치하는가?
[7]  SVI가 Up/Up인가? (VLAN DB + Up 상태 L2 Port + no shutdown)
[8]  전역 ip routing이 활성화되어 있는가?
[9]  라우팅 테이블에 목적지 경로가 있는가? (Longest Prefix Match 기준)
[10] Return Path도 존재하는가?
[11] ACL 또는 보안 정책이 차단하고 있지 않은가?
[12] STP가 해당 VLAN 경로를 Blocking하고 있지 않은가?
[13] HSRP/VRRP Active가 의도한 장비인가?
[14] CPU 사용률이 비정상적으로 높지 않은가?
```

```properties
Gateway Ping 실패            → L2 또는 SVI 문제 우선
Gateway 성공 + 원격 실패      → L3 라우팅 또는 ACL 문제 우선
단방향만 실패                → Return Path 또는 ACL 방향 문제 우선
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| SVI에 IP를 줬는데 VLAN 간 통신 불가 | `ip routing` 미활성 | `show ip route` | `ip routing` 설정 |
| SVI `up/down` | VLAN에 Up 상태 L2 Port 없음 | `show interfaces status` | Access Port 확보 또는 Trunk 허용 |
| SVI `down/down` | VLAN이 VLAN DB에 없음 | `show vlan brief` | `vlan <VLAN-ID>` 생성 |
| SVI `administratively down` | SVI에 `shutdown` | `show running-config interface vlan <VLAN-ID>` | `no shutdown` |
| 로컬 VLAN은 되고 원격 VLAN 실패 | 원격 경로 미학습 | `show ip route` | 정적·동적 라우팅 구성 |
| 특정 VLAN만 통신 불가 | Trunk Allowed VLAN 누락 | `show interfaces trunk` | `switchport trunk allowed vlan add <VLAN-ID>` |
| 단말이 Gateway Ping 실패 | Access VLAN 오할당, Mask 오류 | `show vlan brief` | Port VLAN·단말 설정 수정 |
| 의도하지 않은 VLAN 간 통신 | Native VLAN 불일치 | `show interfaces trunk` | 양단 Native VLAN 일치 |
| `no switchport` 후 링크 단절 | 상대측이 L2로 남음 | `show interfaces <INTERFACE> switchport` | 양단 L3 전환 또는 Rollback |
| Router-on-a-Stick 통신 불가 | Switch 포트가 Access, dot1q 불일치 | `show interfaces trunk`, `show vlans` | Trunk 전환, VLAN ID 일치 |
| DHCP 주소 미할당 | `ip helper-address` 누락, 경로 없음 | `show ip interface vlan <VLAN-ID>` | Relay 설정 및 경로 확보 |
| Ping은 되고 서비스만 차단 | ACL Deny | `show access-lists` | ACL 규칙·방향 수정 |
| 이중화 후 간헐 단절 | HSRP Active·STP Root 불일치 | `show standby brief` | Priority·Root 정렬 |
| L3 EtherChannel Bundle 실패 | L2/L3 속성·모드 불일치 | `show etherchannel summary` | 멤버 속성·프로토콜 일치 |
| 부하 증가 시 지연·손실 | 하드웨어 전달 실패로 CPU 처리 | `show processes cpu sorted` | CEF·TCAM·예외 항목 점검 |
| 경로가 예상과 다름 | Longest Prefix Match 오판 | `show ip route <IP-ADDRESS>` | Prefix → AD → Metric 재검증 |
| `ip routing`·`no switchport` 거부 | 모델·SDM Template·License 제한 | `show version`, `show sdm prefer` | Template 변경 후 재부팅 |

---

## 3. 핵심 진단 명령어

### 라우팅 활성 및 경로

```cisco
show ip route
show ip route <IP-ADDRESS>
show running-config | include ip routing
show ip protocols
```

### SVI

```cisco
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
show ip interface vlan <VLAN-ID>
show running-config interface vlan <VLAN-ID>
```

### VLAN 및 Port

```cisco
show vlan brief
show vlan id <VLAN-ID>
show interfaces status
show interfaces <INTERFACE> switchport
show mac address-table vlan <VLAN-ID>
```

### Trunk 및 인접

```cisco
show interfaces trunk
show cdp neighbors detail
show running-config interface <INTERFACE>
```

### 하드웨어 및 정책

```cisco
show ip cef <IP-ADDRESS>
show adjacency
show arp
show processes cpu sorted
show access-lists
show logging
```

### 이중화 및 STP

```cisco
show standby brief
show vrrp brief
show etherchannel summary
show spanning-tree vlan <VLAN-ID>
```

---

## 4. 대표 장애 시나리오

### 시나리오 1. SVI에 IP를 설정했는데 VLAN 간 통신 불가

- **증상:** 단말이 자신의 Gateway에는 Ping이 되지만 다른 VLAN으로는 실패합니다.
- **원인 후보:** ① 전역 `ip routing` 미활성 ② 상대 VLAN SVI가 Down ③ 단말 Gateway 오설정 ④ ACL 차단 ⑤ 단말 방화벽 ICMP 차단

#### 확인

```cisco
show ip route
show running-config | include ip routing
show ip interface brief | include Vlan
show access-lists
```

```properties
show ip route에 Connected(C) 경로가 전혀 없다 → ip routing 미활성 가능성 높음
Connected는 있으나 상대 VLAN이 없다        → 상대 SVI Down
```

#### 조치

```cisco
configure terminal
 ip routing
end
!
show ip route
ping <IP-ADDRESS>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `ip routing` 활성화 후 기존 `ip default-gateway` 관리 경로가 무효화될 수 있으므로 Default Route를 함께 준비하십시오.

---

### 시나리오 2. SVI가 계속 `up/down` 상태

- **증상:** `show ip interface brief`에서 Status는 `up`, Protocol은 `down`입니다.
- **원인 후보:** 해당 VLAN에 Up 상태 Access Port가 없음, 어떤 Trunk의 Allowed 목록에도 없음, Allowed에는 있으나 Trunk가 Down, Autostate에 의한 정상 Down, VLAN이 `shutdown` 상태

#### 확인

```cisco
show interfaces vlan <VLAN-ID>
show vlan id <VLAN-ID>
show interfaces status
show interfaces trunk
```

```properties
VLAN 내 Up 상태 L2 Port = 0
        ↓
Autostate가 SVI Line Protocol Down
        ↓
Connected 경로 제거 → Inter-VLAN 통신 불가
```

#### 조치

```cisco
configure terminal
 interface <INTERFACE>
  switchport mode access
  switchport access vlan <VLAN-ID>
  no shutdown
end
!
show interfaces status
show ip interface brief | include Vlan
```

Trunk로 VLAN을 수용하는 경우:

```cisco
configure terminal
 interface <INTERFACE>
  switchport trunk allowed vlan add <VLAN-ID>
end
!
show interfaces trunk
show interfaces vlan <VLAN-ID>
```

> Packet Tracer는 Autostate를 실장비와 동일하게 재현하지 않아 Access Port가 없어도 SVI가 Up으로 표시될 수 있습니다.

---

### 시나리오 3. SVI가 `down/down`이거나 VLAN 생성이 거부됨

- **증상:** SVI를 만들었으나 계속 Down이거나 VLAN 생성 명령이 거부됩니다.
- **원인:** SVI는 논리 인터페이스이므로 대응 VLAN이 VLAN Database에 있어야 하며, VTP Client Mode에서는 로컬 VLAN 생성이 불가합니다.

#### 확인

```cisco
show vlan brief
show vlan id <VLAN-ID>
show vtp status
```

#### 조치

```cisco
configure terminal
 vlan <VLAN-ID>
  name USERS
  no shutdown
 exit
 interface vlan <VLAN-ID>
  ip address <IP-ADDRESS> 255.255.255.0
  no shutdown
end
!
show vlan brief
show ip interface brief | include Vlan
```

> VTP Mode 관련 제약은 2-1. 🔄 VTP 개념 & Mode & 동기화를 참고하십시오.

---

### 시나리오 4. 로컬 VLAN끼리는 되는데 원격 Switch VLAN과 통신 불가

- **증상:** 로컬 VLAN 10 ↔ 20은 정상이지만 원격 Switch 대역으로는 실패합니다.
- **원인 후보:** ① Switch 간 라우팅 프로토콜 미구성 ② `network` 문 누락 ③ `passive-interface default` 후 인접 IF 미개방 ④ Area·AS 번호 불일치 ⑤ 업링크 Subnet·Mask 불일치 ⑥ 원격 장비에 Return Path 없음

#### 확인

```cisco
show ip route
show ip route <REMOTE-NETWORK>
show ip ospf neighbor
show ip eigrp neighbors
show ip interface brief
```

#### 조치

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

정적 경로만 필요한 경우:

```cisco
configure terminal
 ip route <REMOTE-NETWORK> <MASK> <NEXT-HOP>
end
!
show ip route static
show ip route <REMOTE-NETWORK>
```

```properties
정상 판정
show ip ospf neighbor State = FULL
show ip route에 O 또는 D로 원격 대역 등재
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 5. 특정 VLAN만 Switch 간 통신 불가

- **증상:** VLAN 10은 정상인데 VLAN 20만 원격 Switch와 통신하지 못합니다.
- **원인 후보:** Trunk Allowed 목록 누락, `allowed vlan` 설정으로 기존 목록 덮어씀, 한쪽에만 VLAN 존재, STP Blocking, VTP Pruning 억제

#### 확인

```cisco
show interfaces trunk
show vlan brief
show spanning-tree vlan <VLAN-ID>
show mac address-table vlan <VLAN-ID>
```

```properties
show interfaces trunk 판독
Vlans allowed on trunk            = 설정된 허용 목록
Vlans allowed and active          = VLAN DB에도 존재
Vlans in spanning tree forwarding = 실제 전달 가능
        ↓
세 번째 목록에 없으면 통신 불가
```

#### 조치

```cisco
configure terminal
 interface <INTERFACE>
  switchport trunk allowed vlan add <VLAN-ID>
end
!
show interfaces trunk
show spanning-tree vlan <VLAN-ID>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `switchport trunk allowed vlan <LIST>`는 기존 목록을 덮어씁니다. 운영 중에는 `add` 또는 `remove`를 사용하십시오.

---

### 시나리오 6. 단말이 Gateway에 Ping조차 되지 않음

- **원인 후보:** ① 포트가 기본 VLAN 1에 남아 있음 ② 단말 Subnet Mask 오류로 Gateway가 다른 네트워크로 판단 ③ Gateway 값이 SVI·Virtual IP와 다름 ④ 포트 `err-disabled` ⑤ Port Security 차단 ⑥ SVI Down

#### 확인

```cisco
show interfaces status
show vlan brief
show interfaces <INTERFACE> switchport
show port-security interface <INTERFACE>
show ip interface brief | include Vlan
show arp
```

#### 조치

```cisco
configure terminal
 interface <INTERFACE>
  switchport mode access
  switchport access vlan <VLAN-ID>
  no shutdown
end
!
show interfaces status
show vlan brief
```

```properties
단말 확인 항목
IP      = SVI Subnet 내부 주소
Mask    = SVI와 동일
Gateway = SVI IP 또는 HSRP/VRRP Virtual IP
```

---

### 시나리오 7. `no switchport` 적용 후 링크가 완전히 끊김

- **원인 후보:** 상대측이 여전히 L2(Trunk/Access), 양단 IP가 다른 Subnet, 기존 Access·Trunk 설정 삭제로 VLAN 확장 중단, 해당 링크가 관리 경로였음, 플랫폼이 Routed Port 미지원

#### 확인

```cisco
show interfaces <INTERFACE> switchport
show ip interface brief <INTERFACE>
show cdp neighbors detail
show version
```

```properties
한쪽 L3 (Routed Port) ↔ 반대쪽 L2 (Trunk/Access)
        ↓
같은 링크에서 서로 다른 계층으로 동작 → IP 인접성 형성 불가
```

#### 조치

```cisco
configure terminal
 interface <INTERFACE>
  no switchport
  ip address <IP-ADDRESS> 255.255.255.252
  no shutdown
end
!
show ip interface brief <INTERFACE>
ping <PEER-IP>
```

Rollback:

```cisco
configure terminal
 interface <INTERFACE>
  no ip address
  switchport
  switchport mode trunk
  switchport trunk allowed vlan <VLAN-LIST>
  no shutdown
end
!
show interfaces trunk
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `switchport`로 되돌려도 이전 L2 설정은 복원되지 않으므로 변경 전 `show running-config interface <INTERFACE>` 출력을 보관해야 합니다.

---

### 시나리오 8. Native VLAN 불일치로 트래픽이 섞임

- **증상:** 의도하지 않은 VLAN 간 통신이 발생하거나 CDP에 Native VLAN Mismatch 경고가 기록됩니다.

```properties
SW1 Native VLAN 1 ↔ SW2 Native VLAN 99
        ↓
VLAN 1 Untagged 프레임이 VLAN 99로 수용
        ↓
VLAN Leaking → Broadcast Domain 병합, STP 오판 가능
```

#### 확인

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
show cdp neighbors detail
show logging
```

#### 조치

```cisco
configure terminal
 interface <INTERFACE>
  switchport trunk native vlan <VLAN-ID>
end
!
show interfaces trunk
```

양단 Native VLAN을 동일하게 맞추고, 사용하지 않는 전용 VLAN을 Native VLAN으로 지정하는 것이 안전합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 9. Router-on-a-Stick 구성 후 통신 불가

- **원인 후보:** ① Switch 연결 포트가 Access로 남음 ② `encapsulation dot1q <VLAN-ID>`와 실제 VLAN 불일치 ③ Router 물리 IF `shutdown` ④ Trunk Allowed 누락 ⑤ 단말 Gateway가 Sub-interface IP가 아님 ⑥ Native VLAN 처리 설정 없음

#### 확인

```cisco
show interfaces trunk
show vlan brief
show vlans
show ip route connected
show cdp neighbors detail
```

#### 조치

```cisco
configure terminal
 interface <INTERFACE>
  switchport trunk encapsulation dot1q
  switchport mode trunk
  switchport trunk allowed vlan <VLAN-LIST>
end
!
show interfaces trunk
```

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
show ip route connected
```

```properties
Router-on-a-Stick 병목
모든 VLAN 간 트래픽이 단일 Trunk를 왕복 → 링크와 CPU 동시 포화
대량 Inter-VLAN 환경은 L3 Switch 이전을 검토
```

---

### 시나리오 10. VLAN 간 DHCP 주소를 받지 못함

- **원인 후보:** `ip helper-address` 누락, SVI Down, 서버까지 왕복 경로 부재, 서버 Pool Subnet 불일치, ACL이 UDP 67/68 차단, 단말 포트 VLAN 오할당

#### 확인

```cisco
show ip interface vlan <VLAN-ID>
show running-config interface vlan <VLAN-ID>
show ip route <DHCP-SERVER-IP>
show access-lists
```

#### 조치

```cisco
configure terminal
 interface vlan <VLAN-ID>
  ip helper-address <DHCP-SERVER-IP>
  no shutdown
end
!
show ip interface vlan <VLAN-ID>
show ip route <DHCP-SERVER-IP>
```

```properties
확인 포인트
giaddr    = SVI IP
서버 Pool = SVI Subnet과 동일 네트워크
경로      = 양방향 모두 존재
```

상세 내용은 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

### 시나리오 11. Ping은 되는데 특정 서비스만 차단됨

- **원인 후보:** ① ACL이 특정 포트만 Deny ② ACL이 잘못된 인터페이스·방향에 적용 ③ 단말 방화벽 차단 ④ 반대 방향만 차단 ⑤ 서버 서비스 미동작

#### 확인

```cisco
show access-lists
show ip access-lists <ACL-NAME>
show ip interface vlan <VLAN-ID>
show logging
```

```properties
Deny 항목 Hit Counter 증가 → 해당 규칙이 차단 원인
Counter 변화 없음         → ACL 적용 위치 또는 방향이 트래픽 경로와 다름
```

#### 조치

```cisco
configure terminal
 ip access-list extended <ACL-NAME>
  permit tcp <SRC-NETWORK> <MASK> <DST-NETWORK> <MASK> eq <PORT>
end
!
show access-lists
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> VLAN 분리는 Broadcast Domain 분리이며 보안 경계가 아닙니다. VLAN 간 차단은 ACL 또는 방화벽으로 명시해야 합니다.

---

### 시나리오 12. 경로가 예상과 다른 방향으로 나감

- **원인 후보:** 더 구체적인 Prefix가 다른 Next-hop을 가리킴, Static이 동적 경로를 대체, Equal-Cost Multi-Path 분산, Summary·Default Route 과다 적용, Floating Static의 AD 오설정

#### 확인

```cisco
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
traceroute <IP-ADDRESS>
```

```properties
[1] 목적지에 매칭되는 가장 긴 Prefix를 먼저 찾는다
[2] 동일 Prefix가 여러 출처에 있으면 AD가 낮은 것
[3] 같은 프로토콜이면 Metric이 낮은 것
[4] Metric까지 같으면 Equal-Cost Multi-Path

AD만 비교하면 오판한다
```

```properties
10.1.0.0/16 via Static (AD 1)   → NEXT-HOP-A
10.1.1.0/24 via OSPF   (AD 110) → NEXT-HOP-B
목적지 10.1.1.10 → NEXT-HOP-B (Prefix가 더 길기 때문)
```

---

### 시나리오 13. Gateway 이중화 후 간헐적 단절

- **원인 후보:** ① HSRP Active와 STP Root가 다른 장비 ② Preempt 미설정 ③ Group ID·Virtual IP 불일치 ④ 양쪽 SVI Subnet·Mask 불일치 ⑤ 업링크 Tracking 없음 ⑥ 단말 Gateway가 SVI 실주소로 설정됨

#### 확인

```cisco
show standby brief
show standby vlan <VLAN-ID>
show spanning-tree vlan <VLAN-ID>
show ip interface brief | include Vlan
show logging
```

#### 조치

```cisco
configure terminal
 interface vlan <VLAN-ID>
  standby version 2
  standby <GROUP-ID> ip <VIRTUAL-IP>
  standby <GROUP-ID> priority 110
  standby <GROUP-ID> preempt
end
!
show standby brief
```

```properties
권장 정렬
HSRP Active = STP Root = 주 업링크 보유 장비
→ 트래픽 경로와 제어 경로 일치, 비대칭 경로 감소
```

상세 내용은 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)를 참고하십시오.

---

### 시나리오 14. L3 EtherChannel이 Bundle되지 않음

- **증상:** `show etherchannel summary`에서 멤버 포트가 `(D)` 또는 `(s)` 상태입니다.
- **원인 후보:** Port-channel은 L3인데 멤버가 L2로 남음, 멤버에 IP 개별 설정, Speed·Duplex·MTU 불일치, `passive-passive` 또는 `auto-auto`, 한쪽만 `on` 모드

#### 확인

```cisco
show etherchannel summary
show interfaces <INTERFACE> switchport
show running-config interface <INTERFACE>
show interfaces status
```

#### 조치

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
show ip interface brief | include Port-channel
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Packet Tracer는 L3 EtherChannel을 모델에 따라 제한적으로 지원하므로 Bundle 실패가 시뮬레이터 제약일 수 있습니다.

---

### 시나리오 15. 부하가 증가하면 지연과 손실이 발생

- **원인 후보:** ① 하드웨어 전달 실패로 패킷이 CPU로 유입 ② TCAM 초과로 경로 프로그래밍 실패 ③ ARP 미해결로 Adjacency Incomplete ④ Broadcast Storm 또는 L2 Loop ⑤ SDM Template 자원 부족 ⑥ ACL이 소프트웨어로 처리됨

#### 확인

```cisco
show processes cpu sorted
show ip cef <IP-ADDRESS>
show adjacency
show arp
show interfaces | include error|drop
show sdm prefer
```

```properties
Control Plane (CPU)   = 경로 계산, 프로토콜, 예외 처리
Data Plane (ASIC/FIB) = 정상 패킷 전달
        ↓
CPU 사용률이 지속적으로 높다
= Data Plane이 처리하지 못한 트래픽이 CPU로 유입
→ CEF → Adjacency → TCAM → SDM Template 순 확인
```

L2 Loop 의심 시:

```cisco
show spanning-tree vlan <VLAN-ID>
show interfaces status err-disabled
show mac address-table count
```

> TCAM 및 하드웨어 자원 확인 명령은 플랫폼별 지원 여부 확인이 필요합니다.

---

### 시나리오 16. `ip routing` 또는 `no switchport` 명령이 거부됨

- **원인 후보:** L2 전용 모델, SDM Template이 라우팅 미지원, License·Feature Set 제한, 플랫폼이 Routed Port 미지원

#### 확인

```cisco
show version
show sdm prefer
show license
show running-config | include ip routing
```

#### 조치

```cisco
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

```properties
SDM Template 변경 = TCAM 자원 재배분
재부팅 후 적용, 재부팅 전까지는 기존 Template로 동작
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> 모델별 Template 이름과 지원 범위는 플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer는 `sdm prefer` 옵션을 실장비보다 제한적으로 제공합니다.

---

### 시나리오 17. 한 방향만 통신되는 비대칭 장애

- **원인 후보:** ① 상대 장비에 Return 경로 없음 ② 한 방향만 ACL Deny ③ 단말 방화벽 인바운드 차단 ④ NAT·정책 라우팅 방향성 ⑤ 비대칭 경로로 상태 기반 장비가 세션 차단

#### 확인 순서

```properties
[1] 양쪽 장비에서 각각 상대 대역 경로 확인
[2] 양방향 ACL 규칙 및 적용 방향 확인
[3] traceroute를 양방향으로 각각 수행
[4] 경로 비대칭과 상태 기반 장비 통과 여부 확인
```

```cisco
show ip route <IP-ADDRESS>
show access-lists
traceroute <IP-ADDRESS>
```

---

## 5. 안전한 변경 및 복구 절차

```properties
[1] 현재 상태 확인 및 출력 저장
[2] 설정 백업 및 외부 보관
[3] 변경 대상이 관리 경로에 포함되는지 확인
[4] Console 또는 OOB 접근 확보
[5] Rollback 예약 (reload in <MINUTES>)
[6] Maintenance Window에서 단계적 적용
[7] 단계마다 show 명령으로 검증
[8] ping·traceroute로 서비스 도달성 확인
[9] 정상 확인 후 reload cancel
[10] write memory로 저장
```

변경 전 수집:

```cisco
show running-config
show ip route
show ip interface brief
show vlan brief
show interfaces trunk
show etherchannel summary
show standby brief
show access-lists
```

변경 후 검증:

```cisco
show ip route
show ip interface brief | include Vlan
show interfaces vlan <VLAN-ID>
show interfaces trunk
show standby brief
ping <IP-ADDRESS>
traceroute <IP-ADDRESS>
```

Rollback:

```cisco
reload in 10
!
reload cancel
!
write memory
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `copy startup-config running-config`는 병합 동작이라 삭제된 항목이 복원되지 않습니다. 완전한 복구는 변경 전 설정 파일 기준으로 개별 항목을 되돌려야 합니다.

---

> 📌 **핵심 요약**
> - Inter-VLAN 장애는 단말 설정과 VLAN 할당부터 확인한다.
> - SVI에 IP가 있어도 `ip routing`이 없으면 라우팅하지 않는다.
> - SVI `up/down`은 해당 VLAN에 Up 상태 L2 Port가 없다는 신호다.
> - SVI `down/down`은 VLAN이 VLAN Database에 없다는 신호다.
> - 특정 VLAN만 실패하면 Trunk Allowed VLAN을 먼저 확인한다.
> - Native VLAN은 Untagged이며 불일치는 VLAN Leaking을 유발한다.
> - `no switchport`는 L2 설정을 삭제하므로 사전 백업이 필수다.
> - 원격 대역 실패는 라우팅 미학습과 Return Path 부재를 확인한다.
> - 경로 판단은 Longest Prefix Match → AD → Metric 순서다.
> - Ping은 되고 서비스만 실패하면 ACL 규칙과 방향을 확인한다.
> - VLAN 간 DHCP는 `ip helper-address`와 왕복 경로가 필요하다.
> - HSRP Active와 STP Root를 동일 장비로 정렬한다.
> - CPU 사용률 급증은 하드웨어 전달 실패 신호일 수 있다.
> - 명령이 거부되면 모델, SDM Template, License를 확인한다.
> - 관련: 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing · 4-2. 🧩 L3 Switch & Inter-VLAN Routing 통합 정리 · 4-3. 🚑 L3 Switch & Inter-VLAN Routing 트러블슈팅 치트시트 · 4-4. ⚡ L3 Switch & Inter-VLAN Routing 명령어 퀵 레퍼런스 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)
