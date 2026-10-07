# EtherChannel 통합 정리

> **Tag:** #Network #Switch #EtherChannel #LACP #PAgP #Cisco  
> **핵심 요약:** EtherChannel은 여러 물리 링크를 하나의 논리 링크(Port-channel)로 묶어 대역폭 집약·이중화·STP 단일 논리 링크화를 동시에 달성하는 기술이다. 협상 프로토콜은 Cisco 전용 PAgP(auto/desirable)와 IEEE 802.3ad 표준 LACP(passive/active)가 있으며 서로 호환되지 않는다. 채널 형성의 성패는 모드 조합뿐 아니라 멤버 포트의 속도·듀플렉스·Trunk 모드·Allowed VLAN·Native VLAN 일치 여부에서도 갈린다.

---

## 1. 핵심 기술 개념

### Q1. EtherChannel은 무엇인가?

👉 **핵심 답변:** 물리적으로 분리된 여러 Ethernet 링크를 Port-channel이라는 하나의 논리 인터페이스로 묶어, STP와 상위 계층이 단일 링크로 인식하게 만드는 Link Aggregation 기술입니다.

```properties
물리 링크 여러 개 → channel-group 설정 → Port-channel 논리 인터페이스 생성
        ↓
STP·L2 Forwarding·L3 Routing은 Po 하나만 인식
        ↓
대역폭 집약 + STP 단일 논리 링크화 + 링크 이중화
```

멤버 하나가 Down되어도 STP 재수렴 없이 나머지 멤버로 전달이 지속됩니다.

> EtherChannel은 다수 물리 → 하나 논리 구조로, 하나 물리 → 다수 논리 구조인 Sub-interface와 정반대 개념입니다.

---

### Q2. 링크를 여러 개 연결하면 되는데 왜 묶는가?

👉 **핵심 답변:** 그냥 여러 개 연결하면 STP가 Loop 방지를 위해 하나만 Forwarding으로 남기고 나머지를 Alternate(Blocking)로 만들기 때문에 대역폭이 늘어나지 않습니다.

```properties
[묶지 않음] 링크 3개 → STP 판정 → Root Port 1개만 Forwarding
            → 나머지 2개 Alternate → 실제 대역폭 = 링크 1개분

[묶음]      링크 3개 → Po1 하나로 인식 → Loop 판정 대상 소멸
            → 실제 대역폭 = 링크 3개분
```

STP 역할 순서는 Root Bridge → Root Port → Designated Port → Alternate Port입니다. EtherChannel은 이 판정을 논리 링크 단위로 수행하게 합니다. 상세는 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence를 참고합니다.

---

### Q3. PAgP와 LACP는 무엇이 다른가?

👉 **핵심 답변:** PAgP는 Cisco 전용 협상 프로토콜, LACP는 IEEE 802.3ad 표준 협상 프로토콜입니다. 타사 장비 연동이 필요하면 LACP를 선택해야 합니다.

| 항목 | PAgP | LACP | Static(on) |
|---|---|---|---|
| 표준 여부 | Cisco 전용 | IEEE 802.3ad | 프로토콜 없음 |
| 능동 Mode | desirable | active | 해당 없음 |
| 수동 Mode | auto | passive | 해당 없음 |
| 협상 패킷 교환 | O | O | X |
| 타사 장비 연동 | 불가 | 가능 | 조건부 |
| 설정 불일치 감지 | 가능 | 가능 | 불가 |
| Standby 멤버 개념 | 제한적 | 지원 | 없음 |

> 한쪽이 PAgP, 다른 쪽이 LACP이면 채널은 절대 형성되지 않습니다.

---

### Q4. 모드 조합은 어떻게 맞춰야 하는가?

👉 **핵심 답변:** 협상 프로토콜 사용 시 최소 한쪽이 능동(desirable 또는 active)이어야 하며, 양쪽 모두 수동(auto 또는 passive)이면 형성되지 않습니다.

| SW1 Mode | SW2 Mode | 프로토콜 | 채널 형성 | 비고 |
|---|---|---|:---:|---|
| on | on | 없음 | ✅ | 협상 없이 강제 Bundle |
| on | desirable | PAgP | ❌ | 한쪽만 협상 시도 |
| on | auto | PAgP | ❌ | 한쪽만 협상 시도 |
| on | active | LACP | ❌ | 한쪽만 협상 시도 |
| on | passive | LACP | ❌ | 한쪽만 협상 시도 |
| desirable | desirable | PAgP | ✅ | 양쪽 능동 |
| desirable | auto | PAgP | ✅ | 능동 + 수동 |
| auto | auto | PAgP | ❌ | 협상 시작 주체 없음 |
| active | active | LACP | ✅ | 양쪽 능동 |
| active | passive | LACP | ✅ | 능동 + 수동 |
| passive | passive | LACP | ❌ | 협상 시작 주체 없음 |
| desirable | active | 혼합 | ❌ | PAgP와 LACP 비호환 |
| auto | passive | 혼합 | ❌ | PAgP와 LACP 비호환 |

```properties
[1] 최소 한쪽은 능동 (desirable / active)
[2] 수동 + 수동 = 실패
[3] PAgP + LACP = 실패
[4] on은 on끼리만
```

> `on + desirable`이 특히 위험합니다. 한쪽은 즉시 Bundle하고 다른 쪽은 협상 응답을 기다리므로 순간적인 Loop가 발생할 수 있습니다.

---

### Q5. Static(on) 모드는 언제 쓰는가?

👉 **핵심 답변:** on 모드는 협상 없이 강제로 묶으므로 상대 상태를 확인하지 못합니다. 협상 프로토콜 미지원 장비 연동 등 불가피한 경우가 아니면 운영망에서 권장하지 않습니다.

```properties
on 모드 특성
PAgP/LACP 패킷을 보내지도 받지도 않음 → 상대 Bundle 여부 확인 불가
오결선 또는 한쪽만 설정 → L2 Loop 발생 가능
```

사용해야 한다면 양쪽 on 설정 사전 합의, 케이블 결선 검증, 멤버 포트를 `shutdown` 상태로 두고 설정한 뒤 동시 활성화, 직후 양쪽 `show etherchannel summary` 확인을 반드시 수행합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q6. Port-channel 논리 인터페이스는 어떻게 동작하는가?

👉 **핵심 답변:** Port-channel은 멤버 포트를 대표하는 논리 인터페이스이며, Trunk·IP·STP 설정은 개별 물리 포트가 아니라 Port-channel 인터페이스에 적용해야 합니다.

```properties
Port-channel 인터페이스 설정 → 멤버 물리 포트로 상속
물리 포트에 개별 설정 → 멤버 간 불일치 → suspended 위험
```

```cisco
interface port-channel 1
 switchport mode trunk
!
interface range <INTERFACE-RANGE>
 channel-group 1 mode active
```

검증:

```cisco
show etherchannel summary
show running-config interface port-channel 1
```

> Channel Group 번호는 로컬 장비 내에서만 의미가 있습니다. 양쪽 번호가 달라도 채널은 형성되지만 운영 편의상 통일을 권장합니다.

---

### Q7. 멤버 포트의 일치 조건은 무엇인가?

👉 **핵심 답변:** 속도, 듀플렉스, Switchport 모드(Trunk/Access), Allowed VLAN, Native VLAN, STP 설정이 모두 일치해야 하며 하나라도 다르면 해당 포트가 suspended 또는 stand-alone이 됩니다.

| 확인 항목 | 불일치 시 결과 | 확인 명령 |
|---|---|---|
| Speed | 미형성 또는 suspended | `show interfaces status` |
| Duplex | 미형성 또는 suspended | `show interfaces status` |
| Switchport 모드 | suspended | `show interfaces <INTERFACE> switchport` |
| Trunk Encapsulation | 미형성 | `show interfaces <INTERFACE> switchport` |
| Allowed VLAN | suspended | `show interfaces trunk` |
| Native VLAN | suspended, VLAN 누수 | `show interfaces trunk` |
| Access VLAN | suspended | `show vlan brief` |
| STP Cost/Priority | 경로 판정 왜곡 | `show spanning-tree interface <INTERFACE>` |
| MTU | 미형성 가능 | `show interfaces <INTERFACE>` |

SPAN Destination Port, Port-security가 설정된 포트, 이미 다른 Channel Group에 속한 포트는 애초에 멤버로 사용할 수 없습니다. 제약 범위는 플랫폼별 지원 여부 확인 필요합니다.

> Trunk 설정 상세는 1-2. 🚚 Switchport & Trunk & Inter-VLAN을 참고합니다.

---

### Q8. L2 EtherChannel과 L3 EtherChannel은 무엇이 다른가?

👉 **핵심 답변:** L2는 Port-channel이 Switchport로 동작해 VLAN Traffic을 전달하고, L3는 `no switchport` 후 Port-channel에 직접 IP를 부여해 Routed Interface로 동작합니다.

| 항목 | L2 EtherChannel | L3 EtherChannel |
|---|---|---|
| Po 성격 | Switchport | Routed Interface |
| IP Address | 부여하지 않음 | Po에 직접 부여 |
| VLAN 소속 | Access 또는 Trunk | 없음 |
| STP 참여 | 참여 | 참여하지 않음 |
| Flag 표기 | S | R |
| 선행 설정 | 없음 | `ip routing` |
| 주 용도 | Access-Distribution | Distribution-Core |

> L3 구성 시 물리 멤버 포트에도 `no switchport`를 적용한 뒤 묶는 것이 안전합니다. Routed Interface와 SVI의 차이는 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing을 참고합니다.

---

### Q9. Load-balancing은 어떻게 동작하는가?

👉 **핵심 답변:** 패킷을 라운드로빈으로 나누지 않고, 지정된 해시 알고리즘 결과에 따라 각 Flow를 특정 멤버 링크에 고정 배정하므로 트래픽 패턴에 따라 편중이 발생합니다.

```properties
패킷 도착 → 기준 필드로 Hash 계산 → 멤버 포트에 매핑
        → 해당 Flow는 항상 같은 멤버로 전송
결과: 단일 세션 처리량 ≤ 멤버 링크 1개 대역폭
```

편중 발생 상황은 통신 쌍이 소수인 구간, 모든 트래픽이 하나의 Gateway MAC을 향하는 구간, 대용량 단일 세션 존재, 멤버 수가 2의 거듭제곱이 아닌 경우입니다.

```cisco
port-channel load-balance src-dst-ip
```

검증:

```cisco
show etherchannel load-balance
show interfaces counters
```

> 지원 알고리즘 목록은 플랫폼과 IOS 버전에 따라 다릅니다. Packet Tracer는 실장비보다 제한적인 옵션만 제공하므로 플랫폼별 지원 여부 확인 필요.

---

### Q10. `show etherchannel summary`의 Flag는 어떻게 읽는가?

👉 **핵심 답변:** Port-channel 뒤 괄호는 채널 전체 상태를, 멤버 포트 뒤 괄호는 개별 포트 상태를 나타냅니다. L2 정상은 `Po(SU)` + 멤버 `(P)`입니다.

```properties
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------
1      Po1(SU)       LACP        Gi0/1(P)  Gi0/2(P)
```

| 채널 Flag | 의미 | 멤버 Flag | 의미 |
|---|---|---|---|
| S | Layer2 채널 | P | bundled (정상) |
| R | Layer3 채널 | I | stand-alone (협상 실패) |
| U | in use | s | suspended (설정 불일치) |
| D | down | D | 물리 링크 Down |
| M | min-links 미충족 | H | Hot-standby (LACP 대기) |
| u | unsuitable for bundling | w | 협상 대기 중 |

```properties
정상 = L2 Po(SU) 또는 L3 Po(RU) + 멤버 전부 (P)
(I) 협상 실패    → 모드·프로토콜 확인
(s) 설정 불일치  → Speed·Duplex·Trunk·VLAN 확인
(D) 물리 문제    → 케이블·포트 상태 확인
```

> Packet Tracer는 `H`, `w`, `M` 등 일부 Flag를 실장비와 다르게 표기하거나 표시하지 않을 수 있습니다.

---

### Q11. LACP의 세부 조정 파라미터는 무엇인가?

👉 **핵심 답변:** System Priority는 어느 Switch가 멤버 선택 결정권을 가질지, Port Priority는 어떤 포트가 Active가 될지, Max Bundle은 동시 Active 멤버 수를, Min Links는 채널 Up 인정에 필요한 최소 멤버 수를 정합니다.

```properties
System Priority (기본 32768) = 낮은 쪽이 결정권, 동률이면 System MAC이 낮은 쪽
Port Priority   (기본 32768) = 낮은 포트가 Active 우선, 초과분은 Hot-standby(H)
Max Bundle 초과 멤버         = (H)로 대기, Active 장애 시 자동 승격
Min Links 미충족             = Po가 down 또는 (M) 표기
```

```cisco
lacp system-priority 100
!
interface <INTERFACE>
 lacp port-priority 100
!
interface port-channel 1
 lacp max-bundle 4
 port-channel min-links 2
```

검증:

```cisco
show lacp sys-id
show lacp neighbor
show lacp <CHANNEL-NUMBER> internal
show etherchannel summary
```

> `lacp max-bundle`과 `port-channel min-links`는 LACP 전용이며 지원 범위와 최대 멤버 수가 플랫폼마다 다릅니다. Packet Tracer는 세부 Flag와 Port Priority 반영이 실장비와 다를 수 있으므로 플랫폼별 지원 여부 확인 필요.

---

### Q12. EtherChannel Misconfig Guard와 err-disabled는 무엇인가?

👉 **핵심 답변:** 설정 불일치로 Loop가 발생할 수 있는 상황을 감지하면 STP가 해당 포트를 err-disabled로 차단합니다.

```properties
한쪽만 EtherChannel 구성 → 상대는 개별 링크로 인식
        → 동일 BPDU가 여러 포트에서 수신
        → Misconfig 감지 → 포트 err-disabled
```

```cisco
interface <INTERFACE>
 shutdown
 no shutdown
!
errdisable recovery cause channel-misconfig
errdisable recovery interval 300
```

검증:

```cisco
show interfaces status err-disabled
show errdisable recovery
show etherchannel summary
```

> 근본 원인인 설정 불일치를 먼저 해소해야 합니다. 원인을 두고 복구만 반복하면 Loop가 재발합니다.

---

### Q13. EtherChannel과 STP는 어떤 관계인가?

👉 **핵심 답변:** EtherChannel은 묶인 링크를 STP에 하나의 논리 링크로 보이게 하므로, STP는 Port-channel 단위로 Root Port와 Designated Port를 판정합니다.

```properties
STP 판정 순서
Root Bridge → Root Port → Designated Port → Alternate
EtherChannel 적용 시 판정 단위 = 물리 포트 ❌ / Port-channel ✅
```

STP Cost는 Po의 합산 대역폭 기준으로 계산되므로 멤버 수가 변하면 경로 선택이 달라질 수 있습니다.

```cisco
show spanning-tree
show spanning-tree interface port-channel 1
show spanning-tree vlan <VLAN-ID>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.
>
> RSTP/MSTP 환경 차이는 3-4. 🚀 RSTP & MSTP (초대형망 STP)를 참고합니다.

---

## 2. 표준 설정 템플릿

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

### 2-1. L2 EtherChannel - LACP (권장)

```cisco
configure terminal
!
interface port-channel 1
 description L2-ETHERCHANNEL-TO-SW2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30
!
interface range <INTERFACE-RANGE>
 channel-protocol lacp
 channel-group 1 mode active
```

상대 Switch는 동일한 Po 설정에 멤버만 `mode passive` 또는 `mode active`로 지정합니다.

검증:

```cisco
show etherchannel summary
show lacp neighbor
show interfaces trunk
show spanning-tree interface port-channel 1
```

---

### 2-2. L2 EtherChannel - PAgP

```cisco
interface port-channel 12
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface range <INTERFACE-RANGE>
 channel-protocol pagp
 channel-group 12 mode desirable
```

상대 Switch는 `desirable` 또는 `auto`로 설정한 뒤 검증합니다.

```cisco
show etherchannel summary
show pagp neighbor
```

---

### 2-3. Static(on) EtherChannel

양쪽을 동일하게 설정하며, 설정 중에는 멤버 포트를 차단합니다.

```cisco
interface range <INTERFACE-RANGE>
 shutdown
 channel-group 3 mode on
!
interface port-channel 3
 switchport mode trunk
!
interface range <INTERFACE-RANGE>
 no shutdown
```

검증:

```cisco
show etherchannel summary
show mac address-table
```

---

### 2-4. L3 EtherChannel

```cisco
configure terminal
!
ip routing
!
interface port-channel 20
 description L3-LINK-TO-CORE
 no switchport
 ip address <IP-ADDRESS> <SUBNET-MASK>
!
interface range <INTERFACE-RANGE>
 no switchport
 channel-protocol lacp
 channel-group 20 mode active
```

검증:

```cisco
show etherchannel summary
show ip interface brief
show ip route
ping <IP-ADDRESS>
```

```properties
정상 지표
show etherchannel summary → Po20(RU)
show ip interface brief   → Port-channel20 up/up
```

> `no switchport` 적용 시 기존 L2 설정이 제거되므로 운영 중 전환은 서비스 단절을 유발합니다.

---

### 2-5. Load-balancing

```cisco
port-channel load-balance src-dst-ip
```

```cisco
show etherchannel load-balance
show interfaces counters
```

| 트래픽 패턴 | 권장 기준 |
|---|---|
| 동일 L2 세그먼트 내 다수 대 다수 | `src-dst-mac` |
| L3 라우팅 경유 다수 IP | `src-dst-ip` |
| 단말이 하나의 Gateway로만 향함 | `dst-mac` 회피 |

> 지원 알고리즘은 플랫폼별 지원 여부 확인 필요.

---

## 3. 통합 실습 부가기술

### 3-1. DHCP 서버 및 Relay

Gateway IP는 반드시 임대 대상에서 제외합니다.

```cisco
ip dhcp excluded-address <IP-ADDRESS>
!
ip dhcp pool VLAN100
 network <NETWORK-ADDRESS> <SUBNET-MASK>
 default-router <IP-ADDRESS>
 dns-server <IP-ADDRESS>
 lease 5 0 0
!
interface vlan <VLAN-ID>
 ip helper-address <IP-ADDRESS>
```

검증:

```cisco
show ip dhcp binding
show ip dhcp pool
show ip interface vlan <VLAN-ID>
```

> `default-router`는 해당 VLAN의 SVI IP와 일치해야 하며, SVI가 down이면 Pool도 정상 동작하지 않습니다. 상세는 5-1. 💾 DHCP (동적 IP 할당)를 참고합니다.

---

### 3-2. NTP 시간 동기화

Server 측은 `ntp master 2`로 기준 시계를 지정하고, Client는 동일한 인증 키로 지정 서버만 신뢰합니다.

```cisco
ntp authenticate
ntp authentication-key 1 md5 cisco1234
ntp trusted-key 1
ntp server <IP-ADDRESS> key 1
```

검증:

```cisco
show ntp status
show ntp associations
```

```properties
Stratum = 기준 시계와의 거리, 숫자가 낮을수록 정확
Server stratum 2 → Client stratum 3
동기화 확인 = show ntp associations 출력의 * 표시
```

> EtherChannel 장애 분석에서 여러 장비 Log 시각을 대조하려면 NTP 동기화가 선행되어야 합니다.

---

### 3-3. Anti-Fragmentation ACL

```cisco
ip access-list extended ANTI-FRAG
 deny ip any host <IP-ADDRESS> fragments
 permit ip any any
!
interface port-channel <CHANNEL-NUMBER>
 ip access-group ANTI-FRAG in
```

검증:

```cisco
show access-lists ANTI-FRAG
show ip interface port-channel <CHANNEL-NUMBER>
```

> `fragments`는 비초기 조각에 매칭됩니다. L3 EtherChannel 구간에서는 물리 멤버가 아닌 Port-channel 인터페이스에 적용해야 전체 멤버에 일관되게 동작합니다.

---

## 4. 안전한 변경 절차

```properties
[1] 현재 확인 : show etherchannel summary / show interfaces trunk / show spanning-tree
[2] 백업      : show running-config 저장
[3] 경로 확인 : 관리 트래픽이 변경 대상 링크를 경유하는지 확인
[4] 적용      : Port-channel 설정 → 멤버 포트 channel-group
[5] 검증      : Flag (SU)/(RU) + (P), Trunk·STP·L3 도달성 확인
[6] 저장      : copy running-config startup-config
```

Rollback:

```cisco
interface range <INTERFACE-RANGE>
 no channel-group <CHANNEL-NUMBER>
!
no interface port-channel <CHANNEL-NUMBER>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 5. 검증 명령어

```cisco
show etherchannel summary
show etherchannel <CHANNEL-NUMBER> detail
show etherchannel load-balance
show lacp neighbor
show pagp neighbor
```

```cisco
show interfaces port-channel <CHANNEL-NUMBER>
show interfaces status
show interfaces trunk
show spanning-tree interface port-channel <CHANNEL-NUMBER>
show ip interface brief
show interfaces status err-disabled
show logging
```

---

## 6. 실무 체크리스트

- [ ] 양쪽 Switch의 협상 프로토콜이 동일한가(PAgP 또는 LACP)?
- [ ] 최소 한쪽이 능동 모드(desirable 또는 active)인가?
- [ ] on 모드를 사용한다면 양쪽 모두 on인가?
- [ ] 멤버 포트의 Speed와 Duplex가 일치하는가?
- [ ] 멤버 포트의 Switchport 모드가 일치하는가?
- [ ] Trunk Allowed VLAN과 Native VLAN이 양쪽에서 동일한가?
- [ ] Trunk 및 IP 설정을 Port-channel 인터페이스에 적용했는가?
- [ ] L3 채널이라면 `ip routing`과 `no switchport`를 적용했는가?
- [ ] `show etherchannel summary`에서 `(SU)`/`(RU)`와 `(P)`를 확인했는가?
- [ ] Load-balance 기준이 실제 트래픽 패턴에 적합한가?
- [ ] SPAN 또는 Port-security 포트를 멤버로 쓰지 않았는가?
- [ ] err-disabled 이력과 Log를 확인했는가?
- [ ] 변경 전 Configuration을 백업하고 Rollback을 준비했는가?
- [ ] 관리 경로가 변경 대상 링크를 경유하는지 확인했는가?

---

> 📌 **핵심 요약**
> - EtherChannel은 다수 물리 링크를 하나의 Port-channel 논리 링크로 묶는 기술이다.
> - 대역폭 집약, STP 단일 논리 링크화, 링크 이중화를 동시에 제공한다.
> - PAgP는 Cisco 전용(auto/desirable), LACP는 IEEE 802.3ad 표준(passive/active)이다.
> - PAgP와 LACP를 혼합하면 채널이 형성되지 않는다.
> - 협상 모드는 최소 한쪽이 능동이어야 하고 auto+auto와 passive+passive는 실패한다.
> - on 모드는 on끼리만 형성되며 협상이 없어 Loop 위험이 있다.
> - Trunk와 IP 설정은 멤버 포트가 아니라 Port-channel 인터페이스에 적용한다.
> - 멤버 포트의 Speed·Duplex·Trunk 모드·Allowed VLAN·Native VLAN이 일치해야 한다.
> - L3 EtherChannel은 `ip routing`과 `no switchport` 후 Po에 IP를 부여한다.
> - Flag는 S=L2, R=L3, U=in use, P=bundled, I=stand-alone, s=suspended, D=down이다.
> - Load-balancing은 Flow 단위 해시이므로 단일 세션은 링크 1개 대역폭을 넘지 못한다.
> - LACP System Priority는 결정권을, Port Priority는 Active 멤버 순위를 정한다.
> - Misconfig Guard는 설정 불일치를 감지해 포트를 err-disabled로 차단한다.
> - 부가기술은 DHCP의 Gateway 제외, NTP의 인증 동기화, ACL의 `fragments` 차단이 핵심이다.
> - 관련: 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3) · 5-2. 🧩 통합 실습 부가기술 (DHCP & NTP & Anti-Frag ACL) · 5-3. 🧩 EtherChannel 통합 정리 · 5-4. 🚑 EtherChannel 트러블슈팅 치트시트 · 5-5. ⚡ EtherChannel 명령어 퀵 레퍼런스 · 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence
