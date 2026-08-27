# 🚑 EtherChannel 트러블슈팅 치트시트

> **Tag:** #Network #Switch #EtherChannel #LACP #PAgP #Cisco  
> **핵심 요약:** EtherChannel 장애는 **물리 Link → 프로토콜 일치 → 모드 조합 → 멤버 설정 일치 → Flag 판독 → STP → 상위 서비스** 순서로 확인한다. Flag가 `(I)`면 협상 실패, `(s)`면 설정 불일치, `(D)`면 물리 문제로 원인 축이 완전히 다르다. 채널이 정상인데 통신이 안 된다면 EtherChannel이 아니라 Allowed VLAN, STP, L3 Gateway 문제일 가능성이 높다.

---

## 1. 🎯 공통 진단 순서

```properties
[1]  멤버 물리 포트가 실제로 Up인가?
[2]  양쪽이 같은 프로토콜인가? (PAgP / LACP / on)
[3]  모드 조합이 유효한가? (최소 한쪽 능동, on은 on끼리)
[4]  멤버 포트의 Speed와 Duplex가 일치하는가?
[5]  멤버 포트의 Switchport 모드(Trunk/Access)가 일치하는가?
[6]  Trunk Allowed VLAN과 Native VLAN이 일치하는가?
[7]  show etherchannel summary Flag가 (SU)/(RU) + (P)인가?
[8]  err-disabled로 차단된 포트가 있는가?
[9]  STP가 Port-channel을 단일 논리 포트로 인식하는가?
[10] 채널 정상인데 통신이 안 되면 VLAN·SVI·Routing을 확인했는가?
```

---

## 2. 🛠️ 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 채널이 아예 생기지 않음 | auto+auto 또는 passive+passive | `show etherchannel summary` | 한쪽을 능동으로 변경 |
| 멤버가 `(I)` stand-alone | 협상 실패, 프로토콜 불일치 | `show lacp neighbor` | 프로토콜·모드 통일 |
| 멤버가 `(s)` suspended | Speed·Duplex·Trunk·VLAN 불일치 | `show interfaces trunk` | 멤버 설정을 Po 기준 통일 |
| 멤버가 `(D)` down | 케이블·SFP·shutdown | `show interfaces status` | 물리 점검 후 `no shutdown` |
| 포트가 err-disabled | EtherChannel misconfig guard | `show interfaces status err-disabled` | 불일치 해소 후 shut/no shut |
| 채널 Up인데 대역폭 미증가 | Flow 단위 해시 편중 | `show etherchannel load-balance` | 알고리즘 변경 |
| 특정 링크만 부하 집중 | 단일 세션 또는 단일 MAC/IP 쌍 | `show interfaces counters` | 기준 필드 변경 |
| Trunk가 동작하지 않음 | 물리 포트에만 Trunk 설정 | `show interfaces trunk` | Po 인터페이스에 설정 |
| Po에 IP를 못 붙임 | `no switchport` 미적용 | `show running-config interface port-channel <N>` | `no switchport` 후 IP |
| L3인데 라우팅 안 됨 | `ip routing` 미설정 | `show ip route` | `ip routing` 활성화 |
| 특정 VLAN만 불통 | Allowed VLAN 누락 | `show interfaces trunk` | `allowed vlan add <VLAN-ID>` |
| Native VLAN 경고 | 양쪽 Native VLAN 상이 | `show interfaces trunk` | 양쪽 동일 값 지정 |
| STP가 링크를 Blocking | 한쪽만 채널 구성 | `show spanning-tree` | 양쪽 동일 채널 구성 |
| 멤버 추가 후 채널 불안정 | 신규 멤버 설정 불일치 | `show etherchannel <N> detail` | 신규 멤버 분리 후 정렬 |
| 일부 멤버가 `(H)` | Max Bundle 도달 | `show lacp <N> internal` | Max Bundle·Port Priority 확인 |
| DHCP 임대 실패 | SVI down, Gateway 미제외 | `show ip dhcp pool` | SVI 확인, `excluded-address` |
| Log 시각 불일치 | NTP 미동기화 | `show ntp status` | 인증 키와 서버 재확인 |

---

## 3. 🔍 핵심 진단 명령어

EtherChannel 및 협상 프로토콜:

```cisco
show etherchannel summary
show etherchannel <CHANNEL-NUMBER> detail
show etherchannel load-balance
show lacp neighbor
show lacp <CHANNEL-NUMBER> internal
show pagp neighbor
```

인터페이스, Trunk 및 VLAN:

```cisco
show interfaces status
show interfaces port-channel <CHANNEL-NUMBER>
show interfaces <INTERFACE> switchport
show interfaces trunk
show vlan brief
show running-config interface <INTERFACE>
```

STP, L3, Log:

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree interface port-channel <CHANNEL-NUMBER>
show ip interface brief
show ip route
show interfaces status err-disabled
show logging
```

---

## 4. 🚨 대표 장애 시나리오

### 시나리오 1. 채널이 형성되지 않고 멤버가 stand-alone `(I)`

증상:

```properties
1      Po1(SD)       LACP        Gi0/1(I)  Gi0/2(I)
```

원인 후보는 양쪽 모두 수동 모드, PAgP와 LACP 혼합, 한쪽만 on, 상대 포트가 다른 Channel Group 소속, 케이블 오결선입니다.

확인:

```cisco
show etherchannel summary
show lacp neighbor
show running-config interface <INTERFACE>
```

조치:

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 channel-protocol lacp
 channel-group 1 mode active
```

검증:

```cisco
show etherchannel summary
show lacp neighbor
```

> 정상 판정은 `Po1(SU)` + 멤버 전부 `(P)`이며, `show lacp neighbor`에 상대 System ID가 나타나야 합니다.

---

### 시나리오 2. 멤버 포트가 suspended `(s)`

원인 후보는 멤버 간 Speed·Duplex 불일치, 한 멤버만 Access, Allowed VLAN 목록 차이, Native VLAN 차이, MTU 차이입니다.

```properties
멤버 A: Trunk, allowed 10,20   / native 999
멤버 B: Trunk, allowed 10,20,30 / native 1
        ↓ Allowed VLAN 및 Native VLAN 불일치
멤버 B suspended
```

확인:

```cisco
show etherchannel 1 detail
show interfaces status
show interfaces trunk
```

조치:

```cisco
configure terminal
!
interface <INTERFACE>
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20
 speed auto
 duplex auto
```

검증:

```cisco
show etherchannel summary
show interfaces trunk
```

---

### 시나리오 3. 한쪽만 EtherChannel을 구성해 포트가 err-disabled

증상:

```properties
%PM-4-ERR_DISABLE: channel-misconfig (STP) error detected on Po1
```

원인:

```properties
SW1 = EtherChannel 구성 완료 / SW2 = 개별 링크 동작
        ↓ 같은 BPDU가 SW1의 여러 포트에서 수신
        ↓ Misconfig Guard 동작 → 포트 err-disabled
```

확인:

```cisco
show interfaces status err-disabled
show errdisable recovery
show logging
```

조치는 상대 Switch에 동일 구성을 적용하고 프로토콜·모드를 일치시킨 뒤 포트를 복구하는 것입니다.

```cisco
interface <INTERFACE>
 shutdown
 no shutdown
```

검증:

```cisco
show interfaces status
show etherchannel summary
show spanning-tree
```

> 원인을 해소하지 않고 복구만 반복하면 err-disabled가 즉시 재발합니다.

---

### 시나리오 4. 채널은 Up인데 트래픽이 한 링크로만 쏠림

```properties
Load-balancing = Flow 단위 Hash 배정 (패킷 단위 라운드로빈 아님)
결과: 단일 세션 처리량 ≤ 멤버 링크 1개 대역폭
```

원인 후보는 통신 쌍이 소수여서 Hash 결과가 수렴, 모든 트래픽이 Gateway MAC 하나를 향함, 대용량 단일 세션 존재, 멤버 수가 2의 거듭제곱이 아님입니다.

확인:

```cisco
show etherchannel load-balance
show interfaces counters
show interfaces port-channel 1
```

조치:

```cisco
configure terminal
!
port-channel load-balance src-dst-ip
```

검증:

```cisco
show etherchannel load-balance
show interfaces counters
```

> load-balance는 장비 전역 설정입니다. 지원 알고리즘은 플랫폼별 지원 여부 확인 필요하며, Packet Tracer는 실장비보다 옵션이 제한적입니다.

---

### 시나리오 5. Po는 정상인데 Trunk가 동작하지 않음

증상은 `show interfaces trunk`에 Port-channel이 나타나지 않는 것입니다. 원인 후보는 Trunk 설정을 물리 포트에만 적용, Po가 기본 Access 상태, Encapsulation 미지정, DTP 협상 결과가 Access입니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```cisco
configure terminal
!
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

검증:

```cisco
show interfaces trunk
show interfaces port-channel 1 switchport
```

```properties
설정 위치 원칙
Trunk / Access / IP → Port-channel 인터페이스
channel-group       → 물리 멤버 포트
```

---

### 시나리오 6. 채널은 정상인데 특정 VLAN만 통신되지 않음

```properties
EtherChannel 정상
≠ Allowed VLAN 자동 추가 ≠ Access Port VLAN 자동 할당 ≠ SVI 자동 생성
```

원인 후보는 Allowed VLAN 누락, VLAN Database에 VLAN 부재, Access Port가 다른 VLAN 소속, STP가 해당 VLAN에서 Blocking, SVI down입니다.

확인:

```cisco
show interfaces trunk
show vlan brief
show spanning-tree vlan <VLAN-ID>
show mac address-table vlan <VLAN-ID>
```

조치 후 재확인:

```cisco
configure terminal
!
interface port-channel 1
 switchport trunk allowed vlan add <VLAN-ID>
!
vlan <VLAN-ID>
 name USERS
```

```cisco
show interfaces trunk
show vlan brief
```

---

### 시나리오 7. Native VLAN 불일치 경고

```properties
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on Port-channel1
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

양쪽 Switch의 Port-channel에 동일한 값을 지정한 뒤 검증합니다.

```cisco
interface port-channel 1
 switchport trunk native vlan 999
```

```cisco
show interfaces trunk
show cdp neighbors detail
show logging
```

> Native VLAN 불일치는 VLAN Hopping 위험과 트래픽 누수를 동반합니다. 관리 VLAN과 분리된 미사용 VLAN을 Native로 지정하는 것이 안전합니다.

---

### 시나리오 8. L3 EtherChannel에 IP를 못 붙이거나 라우팅이 되지 않음

IP 설정이 거부되는 원인은 Port-channel이 여전히 Switchport 상태인 것입니다. IP는 붙었는데 라우팅이 안 된다면 `ip routing` 미설정, Subnet 또는 Mask 불일치, 라우팅 프로토콜에서 네트워크 미광고, 물리 멤버에 `no switchport` 미적용을 확인합니다.

확인:

```cisco
show running-config interface port-channel 20
show interfaces port-channel 20 switchport
show ip route
show ip protocols
```

조치:

```cisco
configure terminal
!
ip routing
!
interface port-channel 20
 no switchport
 ip address <IP-ADDRESS> <SUBNET-MASK>
!
interface range <INTERFACE-RANGE>
 no switchport
 channel-group 20 mode active
```

검증:

```cisco
show etherchannel summary
show ip interface brief
show ip route connected
ping <IP-ADDRESS>
```

```properties
정상 지표
show etherchannel summary → Po20(RU)
show ip interface brief   → Port-channel20 up/up
```

> `no switchport` 적용 시 기존 L2 설정이 제거되므로 운영 중 전환은 서비스 단절을 유발합니다. Gateway 이중화 구간이라면 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)와의 상호작용도 확인합니다.

---

### 시나리오 9. 멤버 포트를 추가했더니 채널 전체가 불안정해짐

원인 후보는 신규 멤버의 Speed/Duplex 상이, 신규 멤버가 다른 상대 장비에 연결됨, Trunk 설정 누락, 케이블·SFP 불량으로 Link Flapping, Max Bundle 초과입니다.

확인:

```cisco
show etherchannel 1 detail
show interfaces <INTERFACE> counters errors
show logging
```

조치는 신규 멤버를 분리해 안정화한 뒤 설정과 결선을 정렬하고 재투입하는 것입니다.

```cisco
configure terminal
!
interface <INTERFACE>
 shutdown
 no channel-group 1
!
interface <INTERFACE>
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
 no shutdown
```

검증:

```cisco
show etherchannel summary
show interfaces status
```

---

### 시나리오 10. STP가 EtherChannel 링크를 Blocking

```properties
멤버 개별 Blocking이 보인다 = 채널이 실제로 형성되지 않았을 가능성
Po 전체가 Alternate        = 다른 경로가 더 우수하다는 정상 판정
```

원인 후보는 한쪽만 채널 구성되어 STP가 개별 링크로 인식, 다른 경로 Cost가 더 낮음, Root Bridge 위치가 설계와 다름, 멤버 수 변경으로 Po Cost 변동입니다.

확인:

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree interface port-channel 1
show etherchannel summary
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```cisco
configure terminal
!
spanning-tree vlan <VLAN-ID> root primary
!
interface port-channel 1
 spanning-tree cost 10
```

검증:

```cisco
show spanning-tree vlan <VLAN-ID>
show etherchannel summary
```

---

### 시나리오 11. Static(on) 모드 구성 중 Loop 발생

증상은 CPU 급증, MAC Flapping, 전 구간 지연입니다.

```properties
%SW_MATM-4-MACFLAP_NOTIF: Host xxxx.xxxx.xxxx in vlan 10 is flapping

SW1 = mode on 설정 완료 / SW2 = 미설정
        ↓ SW1은 협상 없이 즉시 Bundle
        ↓ SW2는 개별 링크로 Forwarding → L2 Loop
```

즉시 조치는 멤버 포트 중 하나만 남기고 차단한 뒤 양쪽 설정을 정렬해 순차 활성화하는 것입니다.

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 shutdown
!
interface range <INTERFACE-RANGE>
 channel-group 1 mode on
 no shutdown
```

검증:

```cisco
show etherchannel summary
show mac address-table
show processes cpu sorted
```

> LACP를 사용하면 상대가 준비되지 않은 상태에서 Bundle되지 않으므로 이 유형의 Loop를 구조적으로 방지할 수 있습니다.

---

### 시나리오 12. SPAN 또는 Port-security로 채널 구성이 거부됨

원인 후보는 포트가 SPAN Destination으로 지정됨, Port-security 활성화, 이미 다른 Channel Group에 속함입니다.

확인 후 제약 기능을 제거하고 다시 묶습니다.

```cisco
show monitor
show port-security
show running-config interface <INTERFACE>
```

```cisco
configure terminal
!
no monitor session <SESSION-NUMBER>
!
interface <INTERFACE>
 no switchport port-security
 no channel-group <OLD-CHANNEL-NUMBER>
 channel-group 1 mode active
```

검증:

```cisco
show etherchannel summary
show monitor
```

> 제약 항목의 세부 범위는 플랫폼과 IOS 버전에 따라 다릅니다. 플랫폼별 지원 여부 확인 필요.

---

### 시나리오 13. LACP는 붙었는데 일부 멤버가 Hot-standby `(H)`

원인 후보는 Max Bundle 도달로 초과 멤버 대기, 플랫폼의 최대 Active 멤버 수 제한, Port Priority가 높은 값이라 대기로 밀림입니다.

```cisco
show etherchannel summary
show lacp <CHANNEL-NUMBER> internal
```

조치 후 검증:

```cisco
configure terminal
!
interface port-channel 1
 lacp max-bundle 8
!
interface <INTERFACE>
 lacp port-priority 100
```

```cisco
show etherchannel summary
show lacp 1 internal
```

> Max Bundle 최대값과 Hot-standby 동작은 플랫폼마다 다르며, Packet Tracer에서는 `(H)` 상태와 Port Priority 반영이 실장비와 다르게 표시될 수 있습니다. 플랫폼별 지원 여부 확인 필요.

---

### 시나리오 14. EtherChannel 구간의 부가 서비스 장애

#### 14-1. 단말이 DHCP로 IP를 받지 못함

원인 후보는 해당 VLAN의 SVI가 down, Gateway IP를 `excluded-address`로 제외하지 않아 충돌, `default-router`가 SVI IP와 불일치, 다른 Subnet인데 `ip helper-address` 누락, EtherChannel Trunk의 Allowed VLAN 누락입니다.

```cisco
configure terminal
!
ip dhcp excluded-address <IP-ADDRESS>
!
ip dhcp pool VLAN100
 network <NETWORK-ADDRESS> <SUBNET-MASK>
 default-router <IP-ADDRESS>
!
interface port-channel 1
 switchport trunk allowed vlan add <VLAN-ID>
```

```cisco
show ip dhcp binding
show ip dhcp pool
show ip interface vlan <VLAN-ID>
```

#### 14-2. NTP 동기화 실패 및 ACL 오차단

NTP 원인 후보는 인증 키 불일치, 서버까지 라우팅 경로 부재, 동기화 완료까지 수 분 소요입니다. ACL 원인 후보는 `permit ip any any` 누락, 물리 멤버 포트에 적용, 방향(in/out) 오류입니다.

```cisco
configure terminal
!
ntp authenticate
ntp authentication-key 1 md5 cisco1234
ntp trusted-key 1
ntp server <IP-ADDRESS> key 1
!
ip access-list extended ANTI-FRAG
 deny ip any host <IP-ADDRESS> fragments
 permit ip any any
!
interface port-channel <CHANNEL-NUMBER>
 ip access-group ANTI-FRAG in
```

```cisco
show ntp status
show ntp associations
show access-lists ANTI-FRAG
show ip interface port-channel <CHANNEL-NUMBER>
```

> L3 EtherChannel에서 ACL은 물리 멤버가 아닌 Port-channel 인터페이스에 적용해야 전체 멤버에 일관되게 동작합니다.

---

## 5. 🔧 안전한 변경 및 복구 절차

### 5-1. 변경 절차

```properties
[1]  현재 상태 확인 : show etherchannel summary / show interfaces trunk / show spanning-tree
[2]  Configuration 백업
[3]  관리 경로 확인 : 변경 대상 링크가 관리 트래픽 경로인지 판단
[4]  Console 또는 OOB 접근 확보
[5]  멤버 포트 shutdown (on 모드 사용 시 필수)
[6]  Port-channel 인터페이스 설정
[7]  멤버 포트에 channel-group 적용
[8]  멤버 포트 no shutdown
[9]  Flag 및 서비스 검증
[10] copy running-config startup-config
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

변경 후 검증 및 저장:

```cisco
show etherchannel summary
show etherchannel <CHANNEL-NUMBER> detail
show interfaces trunk
show spanning-tree interface port-channel <CHANNEL-NUMBER>
copy running-config startup-config
```

---

### 5-2. Rollback 절차

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 no channel-group <CHANNEL-NUMBER>
!
no interface port-channel <CHANNEL-NUMBER>
```

Rollback 후 검증:

```cisco
show etherchannel summary
show interfaces status
show interfaces trunk
show spanning-tree
```

```properties
Rollback 주의
Po 삭제 → 멤버가 개별 링크로 복귀 → STP 재수렴으로 일부 포트 Blocking
        → 순간적인 트래픽 단절 가능
```

---

### 5-3. 긴급 격리 절차

Loop 또는 대규모 장애가 의심되면 멤버 포트를 하나만 남기고 차단한 뒤 안정화를 확인하고 순차 복구합니다.

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 shutdown
```

```cisco
show mac address-table
show processes cpu sorted
show spanning-tree
show logging
```

안정화가 확인되면 멤버를 하나씩 `no shutdown`하며 `show etherchannel summary`와 `show spanning-tree`로 매번 검증합니다.

---

> 📌 **핵심 요약**
> - EtherChannel 장애는 물리 Link → 프로토콜 → 모드 → 멤버 설정 → Flag 순으로 확인한다.
> - `(I)`는 협상 실패, `(s)`는 설정 불일치, `(D)`는 물리 문제로 원인 축이 다르다.
> - auto+auto와 passive+passive는 채널이 형성되지 않는다.
> - PAgP와 LACP를 혼합하면 절대 묶이지 않는다.
> - on 모드는 on끼리만 형성되며 한쪽만 설정하면 Loop가 발생한다.
> - Speed·Duplex·Trunk 모드·Allowed VLAN·Native VLAN 불일치는 suspended를 유발한다.
> - Trunk와 IP 설정은 Port-channel 인터페이스에 적용해야 한다.
> - 채널 정상은 VLAN 통신 보장이 아니며 Allowed VLAN과 SVI를 따로 확인한다.
> - Load-balancing은 Flow 단위 해시이므로 단일 세션 편중은 정상 동작이다.
> - err-disabled는 설정 불일치를 해소한 뒤 shut/no shut으로 복구한다.
> - L3 채널은 `ip routing`과 `no switchport`가 선행되어야 한다.
> - 정상 채널에서 멤버가 개별 Blocking되면 채널이 실제로 형성되지 않은 것이다.
> - 부가 서비스는 DHCP의 SVI·excluded-address, NTP의 인증 키, ACL의 적용 지점을 본다.
> - 변경은 현재 확인 → 백업 → 경로 확인 → 적용 → 검증 → 저장 순으로 수행한다.
> - 관련: 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3) · 5-2. 🧩 통합 실습 부가기술 (DHCP & NTP & Anti-Frag ACL) · 5-3. 🧩 EtherChannel 통합 정리 · 5-4. 🚑 EtherChannel 트러블슈팅 치트시트 · 5-5. ⚡ EtherChannel 명령어 퀵 레퍼런스 · 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence
