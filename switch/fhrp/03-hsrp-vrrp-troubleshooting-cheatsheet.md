# 🚑 HSRP & VRRP 트러블슈팅 치트시트

> **Tag:** #Network #FHRP #HSRP #VRRP #Redundancy #Cisco  
> **핵심 요약:** FHRP 장애는 **Interface/SVI 상태 → 그룹 번호와 가상 IP → Version → 인증 → 타이머 → Priority/Preempt → Tracking → 상위 Routing → STP 정합성** 순서로 확인합니다. State가 Active인데도 통신하지 못한다면 FHRP가 아니라 상위 경로, Tracking 미설정, STP 위치 불일치 또는 단말 Gateway 설정 문제일 가능성이 높습니다.

---

## 1. 🎯 공통 진단 순서

```properties
[1] 해당 Interface 또는 SVI가 Up/Up인가?
[2] Group 번호가 양쪽에서 일치하는가?
[3] 가상 IP가 양쪽에서 완전히 동일하고 Interface Subnet 안에 있는가?
[4] HSRP Version이 양쪽에서 동일한가?
[5] 인증 방식과 Key String이 일치하는가?
[6] Hello/Hold 타이머가 양쪽에서 일치하는가?
[7] Priority와 Preempt가 설계대로 설정됐는가?
[8] Tracking 대상과 Decrement가 의도한 절체를 유발하는가?
[9] 새 Active의 Routing Table에 상위 경로가 있는가?
[10] STP Root Bridge 위치와 FHRP Active 위치가 정합한가?
[11] 단말 Default Gateway가 가상 IP인가?
[12] 두 장비 사이 L2 경로(Trunk)가 살아 있는가?
```

```properties
[판정 논리]
State 이상        → FHRP 자체 문제 (그룹/IP/Version/인증/타이머)
State 정상 + 불통 → 상위 경로 또는 Tracking 또는 단말 문제
경로 비효율       → STP Root와 FHRP Active 위치 불일치
```

---

## 2. 🛠️ 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 양쪽 모두 Active | 그룹·가상 IP·Version·인증 불일치, L2 단절 | `show standby all` | 불일치 항목 통일 후 Trunk 확인 |
| State가 Init에서 멈춤 | SVI Down, 가상 IP Subnet 불일치 | `show ip interface brief` | VLAN·SVI·Allowed VLAN 수정 |
| 복구 후 Active 미복귀 | HSRP Preempt 기본 비활성 | `show standby brief` | `standby <GROUP> preempt` 설정 |
| WAN 장애 시 절체 안 됨 | Tracking 미설정 | `show track brief` | Track Object와 Decrement 설정 |
| Priority는 떨어지나 절체 안 됨 | Decrement 부족, 상대 Preempt 없음 | `show standby all` | Decrement 확대, Standby에 Preempt |
| 절체는 됐는데 통신 불통 | 새 Active의 상위 경로 없음 | `show ip route` | Routing 대칭 구성, Delay 조정 |
| 절체 시 수십초 단절 | Preempt Delay 부족, 수렴 지연 | `show ip ospf neighbor` | Delay 확대, 타이머 재설계 |
| State가 계속 Flapping | 타이머 불일치, CPU 부하, 회선 불안정 | `show logging` | 타이머 통일, 물리 회선 점검 |
| 통신은 되나 경로가 우회 | STP Root와 FHRP Active 불일치 | `show spanning-tree vlan <VLAN-ID>` | VLAN별 Root와 Active 정렬 |
| 특정 단말만 통신 불가 | Gateway가 물리 IP로 설정됨 | 단말 IP 설정, `show ip dhcp pool` | Gateway를 가상 IP로 변경 |
| 예상과 다른 장비가 Active | Priority가 양쪽 기본값 100 | `show standby brief` | Priority 명시적 차등 설정 |
| VRRP 명령이 없음 | 플랫폼·IOS 미지원 또는 문법 상이 | `show version` | 구 문법/신규 문법 확인 |
| HSRPv2 전환 후 절체 | Version 불일치, 그룹 재초기화 | `show standby all` | 양쪽 Version 통일 |
| HSRP는 넘어갔으나 VPN 불통 | HSRP name 및 crypto 연동 누락 | `show crypto isakmp peer` | `standby <GROUP> name` 연동 |

---

## 3. 🔍 핵심 진단 명령어

```cisco
! --- HSRP 및 VRRP 상태 ---
show standby brief
show standby all
show standby <INTERFACE> <GROUP>
show vrrp brief
show vrrp all
show running-config interface <INTERFACE>
!
! --- Tracking ---
show track
show track brief
show track <NUMBER>
!
! --- Interface 및 L2 ---
show ip interface brief
show interfaces <INTERFACE> status
show vlan brief
show interfaces trunk
show mac address-table vlan <VLAN-ID>
!
! --- STP ---
show spanning-tree vlan <VLAN-ID>
show spanning-tree root
!
! --- L3 경로 ---
show ip route
show ip route <IP-ADDRESS>
show ip ospf neighbor
show arp
!
! --- Log 및 Debug ---
show logging
debug standby events
undebug all
```

> `debug` 명령은 운영망에서 CPU 부하와 Console 폭주를 유발할 수 있으므로 반드시 `undebug all`로 해제하십시오.

---

## 4. 🚨 대표 장애 시나리오

### 시나리오 1. 두 장비가 서로 Active (Active-Active 충돌)

#### 증상

```properties
R1 show standby brief → State Active, Standby unknown
R2 show standby brief → State Active, Standby unknown
단말 통신 간헐적 실패, MAC Flapping Log 발생
```

#### 원인 후보

1. Group 번호 불일치
2. 가상 IP 불일치
3. HSRP Version 불일치(v1 vs v2)
4. 인증 방식 또는 Key String 불일치
5. 두 장비 사이 Trunk 또는 L2 경로 단절
6. Trunk Allowed VLAN에서 해당 VLAN 누락
7. 타이머 불일치로 Hello를 놓침

#### 확인

```cisco
show standby all
show running-config interface <INTERFACE>
show interfaces trunk
show logging
```

#### 조치

```cisco
interface <INTERFACE>
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 timers 3 10
 standby 10 authentication md5 key-string cisco123
```

```cisco
show standby brief
show mac address-table vlan <VLAN-ID>
```

> Version 불일치는 서로 다른 멀티캐스트 주소(224.0.0.2 vs 224.0.0.102)와 가상 MAC을 사용하므로 상대 Hello를 전혀 인식하지 못합니다. 나머지 설정이 같아도 Version만 다르면 Active-Active가 됩니다.

---

### 시나리오 2. State가 Init에서 벗어나지 못함

#### 증상

```properties
Interface  Grp  Pri  P  State  Active   Standby   Virtual IP
Vl10       10   150  P  Init   unknown  unknown   192.168.10.254
```

#### 원인 후보

- SVI가 Down 또는 Administratively Down
- 해당 VLAN이 VLAN Database에 없음
- 해당 VLAN에 Up 상태 Port가 없음
- Trunk Allowed VLAN에서 VLAN 누락
- Interface에 IP Address가 없음
- 가상 IP가 Interface Subnet과 다름

#### 확인

```cisco
show ip interface brief
show interfaces vlan <VLAN-ID>
show vlan brief
show interfaces trunk
```

#### 조치

```cisco
vlan 10
 name USERS
!
interface vlan 10
 ip address 192.168.10.252 255.255.255.0
 no shutdown
!
interface <INTERFACE>
 switchport trunk allowed vlan add 10
```

```cisco
show ip interface brief
show standby brief
```

> 가상 IP는 반드시 해당 Interface IP와 동일한 Subnet에 있어야 합니다. Subnet이 다르면 그룹이 정상 동작하지 않습니다.

---

### 시나리오 3. 장애 복구 후 원래 Active로 돌아오지 않음

#### 증상 및 원인

```properties
R1 Priority 150, R2 Priority 100
R1 장애 → R2 Active (정상) → R1 복구 → R2가 여전히 Active

HSRP Preempt = 기본 비활성 → 이 증상은 정상 동작
VRRP Preempt = 기본 활성   → 이 증상이 발생하지 않음
```

Priority를 양쪽 모두 기본값 100으로 둔 경우에도 예상과 다른 장비가 Active가 될 수 있습니다.

```properties
HSRP Tie-Break : Priority(높은 값) → Interface IP Address(높은 값)
Preempt 비활성 시 기존 Active가 유지되어 Tie-Break 결과와 무관해질 수 있음
```

#### 확인

```cisco
show standby brief
show running-config interface <INTERFACE>
```

`show standby brief`의 `P` 열이 Preempt 활성 여부를 나타냅니다.

#### 조치

```cisco
interface <INTERFACE>
 standby 10 priority 150
 standby 10 preempt delay minimum 60
```

```cisco
show standby brief
show standby <INTERFACE> 10
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Preempt를 설정하면 즉시 절체가 발생할 수 있습니다. 운영 시간에 적용하면 순간 단절이 생기므로 Maintenance Window에서 수행하십시오.

---

### 시나리오 4. WAN 회선만 끊겼는데 절체가 되지 않음

#### 증상 및 원인

```properties
R1 LAN Interface = Up/Up, WAN Interface = Down
R1 State = Active 유지, 단말 Ping = U.U.U

FHRP 감지 범위   = 자신의 FHRP 설정 Interface 상태
FHRP 미감지 범위 = 상위 WAN Interface 장애, Routing Neighbor 소실
        ↓
Active가 그대로 유지되어 Traffic이 죽은 경로로 흘러감 → Black Hole
```

#### 확인

```cisco
show ip interface brief
show standby all
show track brief
show ip route
```

#### 조치

```cisco
! === Active 측 ===
track 10 interface <INTERFACE> line-protocol
!
interface <INTERFACE>
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 track 10 decrement 100
!
! === Standby 측 : Preempt 필수 ===
interface <INTERFACE>
 standby 10 priority 100
 standby 10 preempt
```

```cisco
show track
show standby all
show ip route
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 5. Tracking으로 Priority는 떨어지는데 절체가 안 됨

#### 증상

```properties
R1 Priority 설정값 150, 현재 120 / R2 Priority 100
R1 State = Active 유지
```

#### 원인 및 판정 로직

```properties
절체 성립 조건
[1] Track Down 감지
[2] Active Priority - Decrement < Standby Priority
[3] Standby 측 Preempt 활성

세 조건 중 하나라도 불충족 → 절체 실패
```

그 외에 Track 대상 Interface가 실제로는 Up 상태이거나, Track 번호와 그룹 연결이 잘못된 경우도 확인해야 합니다.

#### 확인

```cisco
show track
show track brief
show standby all
show ip interface brief
```

#### 조치

```cisco
interface <INTERFACE>
 no standby 10 track 10 decrement 30
 standby 10 track 10 decrement 100
!
! === Standby 측 ===
interface <INTERFACE>
 standby 10 preempt
```

```cisco
show standby all
show track brief
```

> Decrement 기본값은 10입니다. 값을 생략하거나 Priority 차이보다 작게 두면 Priority만 떨어지고 절체는 발생하지 않습니다.

---

### 시나리오 6. 절체는 됐지만 새 Active에서 통신이 안 됨

#### 증상

```properties
show standby brief → R2 State Active (정상)
단말 → 가상 IP Ping 성공
단말 → 외부 Ping 실패
```

#### 원인 후보

- 새 Active의 Routing Table에 상위 경로 없음
- Routing Protocol Neighbor가 아직 형성되지 않음
- 새 Active의 WAN Interface Down
- 새 Active에 NAT 또는 ACL 설정 누락
- Default Route가 한쪽 장비에만 존재

#### 확인

```cisco
show ip route
show ip route 0.0.0.0
show ip ospf neighbor
show access-lists
show ip nat translations
```

#### 판정 로직

```properties
경로 선택 순서
[1] Longest Prefix Match
[2] Administrative Distance
[3] Metric

AD만 비교해서 판단하면 안 됩니다. 먼저 목적지에 대해 가장 구체적인
Prefix를 찾고, 그 안에서 AD를 비교하고, 동일하면 Metric을 비교합니다.
```

#### 조치

두 장비의 상위 경로 설정을 대칭으로 맞춥니다.

```cisco
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 203.0.113.0 0.0.0.255 area 0
```

```cisco
show ip route
show ip ospf neighbor
show ip protocols
```

> FHRP의 State는 Gateway 역할만 나타냅니다. 절체 검증은 State 확인과 Routing Table 확인을 항상 함께 수행하십시오.

---

### 시나리오 7. 절체 단절 시간이 길거나 State가 Flapping 한다

#### 증상

```properties
설계 목표 3초 이내 → 실측 30~60초 단절
또는 %HSRP-5-STATECHANGE 계열 Log가 반복 발생
```

#### 원인 후보

- Hello/Hold 기본값(3초/10초) 사용
- Routing Protocol 수렴 시간이 지배적
- Preempt Delay 부족으로 미수렴 상태에서 탈환
- 양쪽 타이머 불일치
- 물리 회선 불안정 또는 Duplex Mismatch
- CPU 과부하로 Hello 처리 지연
- Track 대상 Interface Flapping
- STP 재수렴이 함께 발생

#### 확인

```cisco
show standby <INTERFACE> <GROUP>
show ip ospf neighbor
show spanning-tree vlan <VLAN-ID>
show interfaces <INTERFACE> counters errors
show processes cpu sorted
show logging
```

#### 조치

```cisco
interface <INTERFACE>
 standby 10 timers 1 3
 standby 10 preempt delay minimum 90
!
track 10 interface <INTERFACE> line-protocol
 delay up 30 down 10
```

```cisco
show standby <INTERFACE> 10
show track
show processes cpu sorted
```

> 타이머를 줄이면 감지는 빨라지지만 CPU 부하와 Flapping 위험이 커집니다. Millisecond 타이머와 Track Object의 `delay` 옵션은 플랫폼별 지원 여부 확인이 필요합니다.

---

### 시나리오 8. 통신은 되지만 경로가 비효율적이다

#### 증상 및 원인

```properties
VLAN 10 STP Root = SW-A, VLAN 10 FHRP Active = SW-B
        ↓
단말 Traffic이 SW-A로 올라가 Inter-Switch Link를 거쳐 SW-B로 이동
        ↓
Uplink 하나만 사용되고 ISL 대역 낭비, ISL 장애 시 영향 확대
```

L2 경로 결정 주체(STP)와 L3 Gateway 결정 주체(FHRP)가 서로 다른 장비를 가리키기 때문입니다.

```properties
STP 판정 순서
Root Bridge → Root Port → Designated Port → Alternate/Blocking
        ↓
이 결과와 FHRP Active가 같은 장비를 가리켜야 최단 경로가 됩니다.
```

#### 확인

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree root
show standby brief
show interfaces trunk
```

#### 조치

```cisco
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
!
interface vlan 10
 standby 10 priority 150
 standby 10 preempt
!
interface vlan 20
 standby 20 priority 100
 standby 20 preempt
```

```cisco
show spanning-tree vlan 10
show standby brief
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

관련 개념은 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence와 3-4. 🚀 RSTP & MSTP (초대형망 STP)를 참고하십시오.

---

### 시나리오 9. 특정 단말만 통신하지 못한다

#### 원인 후보

- 단말 Default Gateway가 물리 IP로 설정됨
- 단말 ARP Cache에 구 MAC이 남아 있음
- Subnet Mask 오류
- DHCP Pool의 `default-router`가 물리 IP를 배포
- Access Port VLAN 오배정

#### 확인

```cisco
show arp
show mac address-table vlan <VLAN-ID>
show interfaces <INTERFACE> switchport
show ip dhcp pool
show ip dhcp binding
```

단말 측에서는 `ipconfig /all`과 `arp -a`로 Gateway IP와 MAC을 확인합니다.

#### 조치

```cisco
ip dhcp pool VLAN10_POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
!
interface <INTERFACE>
 switchport mode access
 switchport access vlan 10
```

```cisco
show ip dhcp pool
show interfaces <INTERFACE> switchport
show arp
```

> 단말이 물리 IP를 Gateway로 쓰면 평시에는 통신되지만 장애 시 절체 효과를 전혀 받지 못합니다. 이 증상은 장애 후에야 드러나므로 사전 점검이 중요합니다. 상세 내용은 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

### 시나리오 10. HSRPv2로 전환한 뒤 문제가 발생했다

#### 증상 및 원인

```properties
standby version 2 적용 직후 Active-Active 또는 반복 절체 발생

원인 후보
- 한쪽만 Version 2로 변경됨
- Version 변경으로 기존 그룹이 재초기화됨
- 가상 MAC 변경으로 단말 ARP Cache 및 Switch MAC Table 불일치
```

```properties
HSRPv1 가상 MAC = 0000.0c07.acXX / 멀티캐스트 224.0.0.2
HSRPv2 가상 MAC = 0000.0c9f.fXXX / 멀티캐스트 224.0.0.102
        ↓
Version 변경 = 가상 MAC과 멀티캐스트 주소 변경
```

#### 확인

```cisco
show standby all
show running-config interface <INTERFACE>
show mac address-table vlan <VLAN-ID>
show arp
```

#### 조치

```cisco
interface <INTERFACE>
 standby version 2
 standby 10 ip 192.168.10.254
```

```cisco
show standby all
show mac address-table vlan <VLAN-ID>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Group 번호 256 이상을 사용하는 설계는 HSRPv2가 필수입니다. VLAN ID를 Group 번호로 쓰는 관행에서는 VLAN 256 이상을 다룰 때 반드시 v2로 통일하십시오.

---

### 시나리오 11. VRRP 명령이 동작하지 않거나 문법이 다르다

#### 증상

```properties
vrrp 10 ip 192.168.10.254
```

#### 원인 후보

- 플랫폼이 VRRP를 지원하지 않음
- IOS 버전이 구 문법 대신 신규 FHRP 문법만 지원
- L2 전용 Switch에서 시도
- Packet Tracer의 VRRP 지원 범위 제한
- Feature License 또는 Image 제약

#### 확인

```cisco
show version
show ip interface brief
show vrrp
show fhrp
```

Interface Mode에서 `vrrp ?`를 입력하면 사용 가능한 Keyword로 문법 계열을 판별할 수 있습니다.

#### 조치

```cisco
! === 구 문법 ===
interface <INTERFACE>
 vrrp 10 ip 192.168.10.254
 vrrp 10 priority 150
!
! === 신규 FHRP 문법 ===
fhrp version vrrp v3
!
interface <INTERFACE>
 vrrp 10 address-family ipv4
  address 192.168.10.254 primary
  priority 150
```

```cisco
show vrrp brief
show vrrp detail
```

> Packet Tracer는 VRRP를 지원하지 않거나 일부 옵션만 제공하는 경우가 있고 HSRPv2, MD5 Key Chain 인증도 실장비와 다르게 동작할 수 있습니다. 학습 환경에서 검증한 설정을 그대로 운영에 적용하지 말고 플랫폼별 지원 여부를 확인하십시오.

---

### 시나리오 12. HSRP는 절체됐는데 IPsec VPN이 넘어가지 않음

#### 증상

```properties
show standby brief → HQ-2가 Active (정상)
원격 지사와 VPN 불통, Active에서 show crypto isakmp peer 결과 없음
```

#### 원인 후보

- HSRP 그룹에 `name`이 지정되지 않음
- `crypto map ... redundancy <NAME>` 연동 누락
- 원격지가 물리 IP를 Peer로 지정
- 내부/외부 그룹의 교차 Tracking 미설정
- DPD 미설정으로 구 SA 잔존

#### 확인

```cisco
show standby all
show crypto isakmp peer
show crypto isakmp sa
show running-config interface <INTERFACE>
```

#### 조치

```cisco
crypto isakmp keepalive 10
!
interface <INTERFACE>
 standby 2 ip <IP-ADDRESS>
 standby 2 priority 120
 standby 2 preempt
 standby 2 name IPSEC_R
 standby 2 track <INTERFACE> 100
!
interface <INTERFACE>
 no crypto map IPSEC_REDUNDANCY
 crypto map IPSEC_REDUNDANCY redundancy IPSEC_R
```

```cisco
show standby all
show crypto isakmp peer
show crypto engine connections active
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Standby 장비에 SA가 잡히지 않는 것은 정상입니다. 상세 절차는 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동)을 참고하십시오.

---

### 시나리오 13. L2와 L3 계층 문제를 FHRP 문제로 오진한다

#### 판정 기준

```properties
L1 문제 : Interface가 down/down → Cable, SFP, Port 하드웨어
        확인 = show interfaces <INTERFACE>

L2 문제 : up/up이지만 MAC 학습 실패 → VLAN, Trunk, STP Blocking
        확인 = show vlan brief, show interfaces trunk, show spanning-tree

L3 문제 : 같은 Subnet 통신은 되나 외부 통신 실패 → Routing, ACL, NAT
        확인 = show ip route, show arp, show access-lists

FHRP 문제 : L2/L3 정상인데 Active 위치나 절체 동작이 설계와 다름
        확인 = show standby all, show track brief
```

#### 확인 순서

```cisco
show interfaces <INTERFACE>
show vlan brief
show interfaces trunk
show spanning-tree vlan <VLAN-ID>
show ip interface brief
show ip route
show standby all
```

> 계층을 건너뛰고 FHRP만 확인하면 실제 원인이 L1 회선 불량이나 Trunk Allowed VLAN 누락인 경우를 놓칩니다. 아래 계층에서 위 계층으로 순서대로 확인하십시오. SVI 및 Inter-VLAN Routing 개념은 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing, Uplink 이중화는 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3)을 참고하십시오.

---

## 5. 🔧 안전한 변경 및 복구 절차

### 5-1. 변경 순서

```properties
[1] 현재 확인 : State, Priority, Tracking, Routing, STP 위치 기록
[2] 백업      : running-config 및 대상 Interface 설정 저장
[3] 경로 확인 : Console 또는 OOB 접근 경로 확보
[4] 적용      : Standby 측 먼저 적용한 뒤 Active 측 적용
[5] 검증      : State, 가상 MAC, Tracking, Routing, 실제 서비스 확인
[6] 저장      : copy running-config startup-config
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

### 5-2. 변경 전후 수집

```cisco
! === 변경 전 ===
show running-config interface <INTERFACE>
show standby brief
show vrrp brief
show track brief
show ip route
show spanning-tree vlan <VLAN-ID>
!
! === 변경 후 ===
show standby all
show track brief
show ip route
show ip ospf neighbor
show mac address-table vlan <VLAN-ID>
show arp
show logging
```

### 5-3. Rollback

```cisco
interface <INTERFACE>
 no standby <GROUP> priority
 no standby <GROUP> preempt
 no standby <GROUP> track 10 decrement 100
 no standby version 2
 no standby <GROUP> ip <IP-ADDRESS>
!
no track 10
```

```cisco
show standby brief
show running-config interface <INTERFACE>
show ip route
```

> Rollback 명령을 미리 작성해 Console 창에 붙여 넣을 수 있도록 준비하십시오. 절체가 잘못 발생하면 원격 세션이 즉시 끊길 수 있습니다.

### 5-4. 장애 발생 시 우선 조치

```properties
[1] 현재 State와 Log를 먼저 수집 (증거 확보)
[2] 서비스 복구를 위해 정상 장비를 Active로 고정
[3] Active-Active 상태면 한쪽 그룹을 일시 제거해 중복 해소
[4] Tracking 오설정이 원인이면 Decrement를 원복
[5] 서비스 복구 후 근본 원인 분석
[6] 설정 변경 내역과 검증 결과 기록
```

```cisco
show standby all
show track brief
show ip route
show logging
```

---

> 📌 **핵심 요약**
> - FHRP 장애는 Interface와 SVI 상태부터 확인한다.
> - 그룹 번호, 가상 IP, Version, 인증, 타이머가 모두 일치해야 한다.
> - Active-Active의 대표 원인은 Version 불일치와 L2 단절이다.
> - State Init는 대부분 SVI Down 또는 VLAN·Trunk 문제다.
> - HSRP는 Preempt가 기본 비활성이라 복구 후 자동 탈환하지 않는다.
> - Priority를 양쪽 기본값 100으로 두면 Active 위치를 예측할 수 없다.
> - Tracking 없이는 WAN 장애를 감지하지 못해 Black Hole이 발생한다.
> - Decrement가 Priority 차이보다 작으면 절체가 일어나지 않는다.
> - Tracking 절체는 Standby 측 Preempt가 있어야 완성된다.
> - State가 Active여도 상위 Routing이 없으면 통신은 실패한다.
> - 경로 선택은 Longest Prefix Match → AD → Metric 순서로 판단한다.
> - STP Root와 FHRP Active 위치가 다르면 경로가 우회한다.
> - 단말 Gateway와 DHCP `default-router`는 가상 IP여야 한다.
> - Packet Tracer는 VRRP, HSRPv2, 인증 옵션이 실장비와 다를 수 있다.
> - 관련: 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy) · 7-2. 🧩 HSRP & VRRP 통합 정리 · 7-3. 🚑 HSRP & VRRP 트러블슈팅 치트시트 · 7-4. ⚡ HSRP & VRRP 명령어 퀵 레퍼런스 · 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동)
