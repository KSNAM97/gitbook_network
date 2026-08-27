# Switch 보안 통합 정리

> **Tag:** #Network #Switch #Security #PortSecurity #SPAN #Cisco  
> **핵심 요약:** Switch 보안은 Access Edge에서 Source MAC을 제한하는 Port-security, 폭주 Traffic으로부터 Switch와 Segment를 보호하는 Storm-control, Traffic을 복제해 탐지·감사에 사용하는 SPAN/RSPAN을 축으로 구성된다. 여기에 미사용 Port Shutdown, 미사용 VLAN 격리, Native VLAN 명시, BPDU Guard 연계를 더해 방어 계층을 완성한다. 각 기능은 대체 관계가 아니며, Port-security와 SPAN Destination은 같은 Port에 병행할 수 없으므로 Port 역할 설계가 먼저다.

---

## 1. 핵심 기술 개념

### Q1. Switch 보안의 방어 축은 무엇인가?

👉 **핵심 답변:** 접근 제어(Port-security·BPDU Guard), 자원 보호(Storm-control·Err-disable), 가시성 확보(SPAN/RSPAN), 구조적 축소(미사용 Port·VLAN 정리)입니다.

```properties
[접근 제어]  Port-security / BPDU Guard / 미사용 Port Shutdown
        ↓
[자원 보호]  Storm-control / Err-disable
        ↓
[가시성]     SPAN·RSPAN / Syslog / SNMP Trap
        ↓
[구조 축소]  격리 VLAN / Allowed VLAN 제한 / Native VLAN 명시
```

| 기능 | 계층 | 보호 대상 | 방어 효과 |
|---|---|---|---|
| Port-security | L2 | Access Port | 비인가 Source MAC 유입 억제 |
| Storm-control | L1/L2 Port | Switch CPU·Segment 대역폭 | 폭주 Traffic 확산 억제 |
| SPAN / RSPAN | L2 복제 | 분석 장비 | 탐지·감사·사후 분석 |
| BPDU Guard | L2/STP | Access Port | 비인가 Switch·Loop 차단 |
| 미사용 Port Shutdown | L1 | 물리 Port | 접속 가능 지점 축소 |
| Native VLAN 명시 | L2 Trunk | Trunk 경계 | VLAN 경계 모호성 제거 |

> Storm-control은 Port 단위 Traffic 양 제어이므로 L1/L2 경계에 걸쳐 있고, Port-security는 Frame의 Source MAC을 판단하는 L2 기능입니다. 두 기능을 L3 ACL과 같은 계층으로 취급하면 안 됩니다.

---

### Q2. Port-security는 무엇을 기준으로 판단하는가?

👉 **핵심 답변:** 해당 Port에서 수신한 Frame의 **Source MAC Address**입니다.

```properties
Source MAC        = MAC Address Table 학습 기준
Destination MAC   = 전달(Forwarding) 결정 기준
Destination 미학습 = 해당 VLAN의 Broadcast Domain 범위로 Flooding

Port-security = Source MAC 검사 → Secure MAC 목록·Maximum 초과 판단
```

- Port-security는 "누가 들어왔는가"를 제한하며 "어디로 나가는가"는 제어하지 않습니다.
- Source MAC을 신뢰 기준으로 사용하므로 MAC 변경에 대한 저항성은 낮습니다.
- 사용자·장비 인증이 필요하면 802.1X/MAB 계열을 함께 검토합니다. 비교는 3-3-2. 🧩 스위치 & 방어 보안 통합 정리를 참고하십시오.

---

### Q3. Secure MAC의 학습 방식은 어떻게 구분하는가?

| 학습 방식 | 등록 주체 | Running-config 기록 | 재부팅 후 유지 | 기본 Aging 대상 |
|---|---|:---:|:---:|:---:|
| Static | 관리자 | O | 저장 시 O | X |
| Dynamic | 자동 학습 | X | X | O |
| Sticky | 자동 학습 후 고정 | O | 저장 시 O | X |

```cisco
interface <INTERFACE>
 switchport mode access
 switchport port-security maximum 1
 switchport port-security mac-address <MAC-ADDRESS>
 switchport port-security mac-address sticky
 switchport port-security
```

검증:

```cisco
show port-security interface <INTERFACE>
show port-security address
show running-config interface <INTERFACE>
```

```properties
Dynamic Secure MAC → sticky 활성화 → Running-config 기록
→ copy running-config startup-config → 재부팅 후 유지
```

> Sticky는 처음 들어온 MAC을 그대로 신뢰합니다. 배선 검증 후 활성화하고, 잘못 고정되면 `clear port-security sticky interface <INTERFACE>`로 초기화합니다.

---

### Q4. Violation Mode는 어떻게 다른가?

👉 **핵심 답변:** 세 Mode 모두 위반 Frame을 폐기하지만 로그·Trap·Counter·Port 상태에서 차이가 발생합니다.

| Violation Mode | 위반 Frame 폐기 | Syslog 로그 | SNMP Trap | Violation Counter 증가 | Port 상태 |
|---|:---:|:---:|:---:|:---:|---|
| `protect` | O | X | X | X | Up 유지 |
| `restrict` | O | O | O | O | Up 유지 |
| `shutdown` | O | O | O | O | Err-disabled (Secure-shutdown) |
| `shutdown vlan` | O | O | O | O | 해당 VLAN만 차단 (플랫폼별 지원 여부 확인 필요) |

```cisco
interface <INTERFACE>
 switchport port-security violation restrict
```

검증:

```cisco
show port-security interface <INTERFACE>
show running-config interface <INTERFACE>
```

```properties
기본 Violation Mode = shutdown
기본 Maximum        = 1

protect  = 조용히 폐기 → 사고 추적 불가
restrict = 폐기 + 기록 → 가용성 + 감사 추적
shutdown = Port 격리   → 보안 최우선, 사용자 영향 최대
```

- 감사 추적과 가용성을 함께 원하는 사용자 Port: `restrict`
- 비인가 접속을 즉시 격리해야 하는 서버룸·보안망: `shutdown`
- `protect`는 위반 사실 확인이 어려워 운영 표준으로 권장하지 않습니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### Q5. Maximum과 Aging은 어떻게 설계하는가?

👉 **핵심 답변:** Maximum은 해당 Port를 정상적으로 통과하는 실제 Source MAC 개수를 조사해 정하고, Aging은 장비 교체 빈도에 맞춰 선택합니다.

```cisco
interface <INTERFACE>
 switchport port-security maximum 2
 switchport port-security aging time 60
 switchport port-security aging type inactivity
 switchport port-security aging static
```

검증:

```cisco
show port-security interface <INTERFACE>
show mac address-table interface <INTERFACE>
```

MAC 수가 늘어나는 대표 사례:

- IP Phone 하위 PC 연결: Phone MAC + PC MAC
- Hypervisor Host: 가상 머신 MAC 다수
- Docking Station, Server NIC Teaming/Bonding, Access Point

| Aging Type | 동작 |
|---|---|
| `absolute` | 등록 후 지정 시간 경과 시 삭제 |
| `inactivity` | 지정 시간 동안 Traffic이 없으면 삭제 |

```properties
aging time 단위 = 분(Minute), 기본값 0 (Aging 없음)
aging static    = Static Secure MAC도 Aging 대상 포함
Static/Sticky   = 기본적으로 Aging 대상 아님
```

> Maximum 상향은 가장 쉬운 조치이지만 보안 수준을 낮춥니다. Sticky Secure MAC의 Aging 지원 여부는 플랫폼별 지원 여부 확인 필요입니다.

---

### Q6. Err-disabled Port는 어떻게 복구하는가?

👉 **핵심 답변:** 원인을 제거한 뒤 `shutdown` → `no shutdown`으로 수동 복구하거나 `errdisable recovery`로 자동 복구를 설정합니다.

```cisco
errdisable recovery cause psecure-violation
errdisable recovery cause storm-control
errdisable recovery cause bpduguard
errdisable recovery interval 300
!
interface <INTERFACE>
 shutdown
 no shutdown
```

검증:

```cisco
show interfaces status err-disabled
show errdisable recovery
show errdisable detect
show logging
```

```properties
errdisable recovery interval 단위 = 초(Second)

자동 복구 = Port를 다시 Up
자동 복구 ≠ 원인 제거
원인 지속 → 복구 → 재위반 → 재차단 반복
```

> Interval을 지나치게 짧게 설정하면 Port Flapping과 STP 재수렴이 반복되어 장애 범위가 확대됩니다.

---

### Q7. Storm-control은 무엇을 어떻게 제어하는가?

👉 **핵심 답변:** Port 단위로 Broadcast·Multicast·Unicast 유입량을 감시하여 임계치 초과 Traffic을 억제하거나 Port를 Err-disable로 전환합니다.

```properties
Port 수신 Traffic 측정
    ↓ Upper Threshold 초과
해당 Traffic Type 억제 (기본) 또는 action에 따라 Shutdown/Trap
    ↓ Lower Threshold 이하로 감소
정상 전달 재개
```

| Traffic Type | 방어 관점 유의사항 |
|---|---|
| `broadcast` | ARP·DHCP 등 정상 Broadcast 포함, 과소 임계치 주의 |
| `multicast` | 라우팅·관리 Multicast까지 억제될 수 있어 신중히 적용 |
| `unicast` | 플랫폼에 따라 Unknown Unicast만이 아니라 전체 Unicast 대상 가능 |

| 단위 | 명령 형식 |
|---|---|
| 백분율 | `storm-control broadcast level <UPPER> [<LOWER>]` |
| pps | `storm-control broadcast level pps <UPPER> [<LOWER>]` |
| bps | `storm-control broadcast level bps <UPPER> [<LOWER>]` |

| Action | 동작 | Port 상태 | 복구 방식 |
|---|---|---|---|
| 미지정 (기본) | 초과 Traffic 억제 | Up 유지 | Lower 이하 감소 시 자동 |
| `trap` | 억제 + SNMP Trap | Up 유지 | 자동 |
| `shutdown` | Err-disable 전환 | Err-disabled | 수동 또는 `errdisable recovery cause storm-control` |

```properties
Upper만 지정     = Lower도 동일 값으로 동작
Upper/Lower 분리 = 임계치 부근 Flapping 억제
```

> Broadcast부터 적용하고 Multicast/Unicast는 Baseline 측정 후 단계적으로 확대하십시오. `pps`/`bps` 옵션, `k`/`m`/`g` 접미사, 소수점 허용 범위는 플랫폼별 지원 여부 확인 필요이며, Packet Tracer는 백분율 기반만 정상 동작하거나 `pps`/`bps` 입력을 거부할 수 있습니다.

---

### Q8. SPAN은 왜 필요하고 Local SPAN과 RSPAN은 어떻게 다른가?

👉 **핵심 답변:** Switch는 Destination MAC 기준으로 해당 Port로만 전달하므로 분석 장비를 임의 Port에 연결해도 다른 Port의 Traffic이 보이지 않습니다. SPAN은 지정한 Source의 Traffic만 Destination Port로 복제하며, 분석 장비가 다른 Switch에 있으면 RSPAN을 사용합니다.

```properties
Switch 기본 동작 = Destination MAC 기준 선택적 전달
    ↓
분석 장비는 자신에게 향하지 않은 Traffic을 볼 수 없음
    ↓
SPAN = 관리자가 명시적으로 지정한 Traffic만 복제
```

| 방향 옵션 | 복제 대상 |
|---|---|
| `rx` | Source로 들어오는 Traffic |
| `tx` | Source에서 나가는 Traffic |
| `both` | 양방향 (기본값) |

| 구분 | Local SPAN | RSPAN |
|---|---|---|
| Source와 분석 장비 위치 | 동일 Switch | 서로 다른 Switch |
| 전달 수단 | Switch 내부 복제 | 전용 RSPAN VLAN + Trunk |
| 필수 VLAN 설정 | 없음 | `remote-span` VLAN 필요 |
| 경로 요구사항 | 없음 | 경로상 모든 Trunk의 Allowed VLAN 포함 |

```properties
Local SPAN         : Source Interface → 같은 Switch의 Destination Interface
RSPAN Source       : Source Interface → Destination Remote VLAN
RSPAN Destination  : Source Remote VLAN → Destination Interface
```

> SPAN은 원본 Traffic을 전달 경로에서 제거하지 않고 복제만 수행하지만 Switch Fabric과 Destination Port 부하가 증가합니다. RSPAN VLAN은 전용 VLAN이어야 하며 MAC 학습이 수행되지 않으므로 일반 Access Port나 SVI를 할당해도 통신하지 못합니다.

---

### Q9. SPAN Destination Port의 특성과 Session 제약은 무엇인가?

| 특성 | 동작 |
|---|---|
| 일반 Forwarding | 수행하지 않음 |
| MAC 학습 | 수행하지 않음 |
| STP 참여 | 참여하지 않음 |
| 수신 Traffic | 기본 폐기 (`ingress` 옵션 지원 시 예외) |
| 기존 Port 설정 | Destination 지정 시 무효화되거나 제한됨 |
| 다중 Session Destination | 같은 Port를 여러 Session에 지정 불가 |

병행할 수 없거나 제한되는 조합:

- Port-security가 설정된 Port를 SPAN Destination으로 사용
- EtherChannel Member Port를 SPAN Destination으로 사용
- Trunk Uplink를 Destination으로 전용화 (관리 경로 단절 위험)
- 동일 Session에서 같은 Port를 Source와 Destination으로 지정

| Session 제약 | 내용 |
|---|---|
| Source 혼용 | 같은 Session에서 Source Interface와 Source VLAN 혼용 불가 |
| VLAN SPAN 방향 | 플랫폼에 따라 `rx`만 지원하거나 `tx`/`both` 제약 존재 |
| RSPAN VLAN | RSPAN VLAN은 VLAN SPAN Source VLAN으로 사용 불가 |
| Session 수 | 하드웨어 자원에 따라 소수(예: 2개)만 지원 |
| Session Number 재사용 | 기존 Session 설정이 대체됨 |

확인:

```cisco
show monitor session all
show interfaces <INTERFACE> switchport
show mac address-table interface <INTERFACE>
```

> Destination Port에서 Ping이 되지 않는 것은 장애가 아니라 정상 동작입니다. 정확한 Session 수와 VLAN SPAN 제약은 플랫폼별 지원 여부 확인 필요이며, Packet Tracer는 실장비의 Session 개수 제한과 자원 제약을 재현하지 않습니다.

---

### Q10. VLAN 분리만으로 보안 경계가 완성되는가?

👉 **핵심 답변:** 아닙니다. VLAN은 Broadcast Domain 분리이며 접근 제어 정책이 아닙니다.

```properties
VLAN 분리 = Broadcast Domain 분리, L2 Segment 분리
VLAN 분리 ≠ 완전한 보안 경계, ≠ Access Control 정책

L3 Gateway + Routing 존재 → VLAN 간 통신 가능
```

- VLAN 간 접근 제어는 ACL, VRF, Firewall 등 별도 정책으로 구현합니다.
- Trunk 설계 오류나 Native VLAN 불일치가 있으면 VLAN 경계가 모호해집니다.
- 미사용 VLAN은 SVI를 만들지 않고 격리 상태로 유지합니다.
- 기초 개념은 1-1. 🔌 스위치 동작원리 & VLAN 기초, 라우팅 구조는 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing을 참고하십시오.

---

## 2. Port 역할별 표준 설정

### 2-1. 일반 사용자 Port

```cisco
configure terminal
!
interface <INTERFACE>
 description ACCESS_USER
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security aging time 60
 switchport port-security aging type inactivity
 switchport port-security
```

검증:

```cisco
show port-security interface <INTERFACE>
show interfaces <INTERFACE> switchport
show mac address-table interface <INTERFACE>
```

---

### 2-2. 고정 장비 Port (Static MAC)

```cisco
configure terminal
!
interface <INTERFACE>
 description SERVER_FIXED
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport nonegotiate
 !
 switchport port-security maximum 1
 switchport port-security mac-address <MAC-ADDRESS>
 switchport port-security violation shutdown
 switchport port-security
```

검증:

```cisco
show port-security interface <INTERFACE>
show port-security address
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 2-3. 미사용 Port 격리

```cisco
configure terminal
!
interface range <INTERFACE>
 description UNUSED_PORT
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport nonegotiate
 no cdp enable
 shutdown
```

검증:

```cisco
show interfaces status
show interfaces description
show vlan brief
```

```properties
[1] shutdown       = 물리적 활성화 차단
[2] 격리 VLAN 할당  = SVI·Gateway 없는 VLAN
[3] nonegotiate    = Trunk 자동 형성 차단
[4] no cdp enable  = 불필요한 정보 노출 축소
```

> 격리 VLAN을 Native VLAN이나 관리 VLAN으로 재사용하면 격리 효과가 사라집니다.

---

### 2-4. Trunk Uplink와 Native VLAN

```cisco
configure terminal
!
interface <INTERFACE>
 description TRUNK_TO_DISTRIBUTION
 switchport mode trunk
 switchport trunk native vlan <VLAN-ID>
 switchport trunk allowed vlan <VLAN-ID>
 switchport nonegotiate
```

검증:

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
```

```properties
[1] 양쪽 Trunk의 Native VLAN을 동일하게 명시
[2] 사용자 VLAN과 분리된 전용 Native VLAN 사용
[3] Allowed VLAN을 필요한 VLAN으로만 제한
[4] Uplink에는 일반 PC용 Port-security 프로파일을 적용하지 않음
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Native VLAN을 한쪽만 변경하면 Untagged Frame이 서로 다른 VLAN으로 해석됩니다. 양쪽을 같은 Maintenance Window에 변경하십시오. 세부 동작은 1-2. 🚚 Switchport & Trunk & Inter-VLAN을 참고하십시오.

---

### 2-5. BPDU Guard 연계

```cisco
configure terminal
!
interface range <INTERFACE>
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

검증:

```cisco
show spanning-tree interface <INTERFACE> detail
show interfaces status err-disabled
```

```properties
PortFast   = End Device Port 전용, 즉시 Forwarding 전환
BPDU Guard = 해당 Port에서 BPDU 수신 시 Err-disable

PortFast 단독         → 비인가 Switch 연결 시 Loop 위험
PortFast + BPDU Guard → 비인가 Switch 연결 즉시 차단
Uplink/Trunk          → 이 조합 적용하지 않음
```

> PortFast는 End Device가 연결되는 Port에만 적용합니다. Switch, Bridge, Hypervisor Bridge가 연결될 Port에는 적용하지 않습니다. 상세 내용은 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard)에서 확인하십시오.

---

## 3. Storm-control 프로파일

Access Port 기본 (알림 중심):

```cisco
configure terminal
!
interface <INTERFACE>
 storm-control broadcast level 20.00 10.00
 storm-control multicast level 30.00 20.00
 storm-control action trap
```

Err-disable 전환 방식:

```cisco
configure terminal
!
interface <INTERFACE>
 storm-control broadcast level pps 1k 500
 storm-control action shutdown
!
errdisable recovery cause storm-control
errdisable recovery interval 300
```

검증:

```cisco
show storm-control <INTERFACE>
show storm-control broadcast
show interfaces status err-disabled
show errdisable recovery
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Uplink 또는 Trunk에 `storm-control action shutdown`을 적용하면 Storm 발생 시 상위 경로가 차단되어 하위 Segment 전체 단절과 STP 재수렴이 발생합니다. Uplink에는 `trap`과 모니터링을 우선 검토하십시오.

---

## 4. SPAN / RSPAN 구성

### 4-1. Local SPAN

```cisco
configure terminal
!
monitor session 1 source interface <INTERFACE> both
monitor session 1 destination interface <INTERFACE>
```

Trunk Source에서 특정 VLAN만 복제:

```cisco
monitor session 2 source interface <INTERFACE> both
monitor session 2 filter vlan <VLAN-ID>
monitor session 2 destination interface <INTERFACE>
```

VLAN SPAN:

```cisco
monitor session 3 source vlan <VLAN-ID> rx
monitor session 3 destination interface <INTERFACE>
```

검증:

```cisco
show monitor session all
show interfaces status
```

> `filter vlan`은 Trunk Source Port에서 사용하며 Access Port Source에서는 의미가 없습니다. Packet Tracer는 `filter vlan`과 VLAN SPAN을 지원하지 않을 수 있습니다.

---

### 4-2. RSPAN

Source Switch:

```cisco
configure terminal
!
vlan <VLAN-ID>
 remote-span
!
monitor session 1 source interface <INTERFACE> both
monitor session 1 destination remote vlan <VLAN-ID>
```

Destination Switch:

```cisco
configure terminal
!
vlan <VLAN-ID>
 remote-span
!
monitor session 1 source remote vlan <VLAN-ID>
monitor session 1 destination interface <INTERFACE>
```

경로 Trunk 허용:

```cisco
interface <INTERFACE>
 switchport trunk allowed vlan add <VLAN-ID>
```

검증:

```cisco
show vlan remote-span
show monitor session 1
show interfaces trunk
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> 일부 플랫폼과 Packet Tracer는 RSPAN Source Session에 `reflector-port <INTERFACE>` 지정을 요구합니다. Reflector Port는 반드시 미사용 Port여야 하며, 요구 여부는 플랫폼별 지원 여부 확인 필요입니다.

---

## 5. 안전한 변경과 Rollback

```properties
[1] 현재 확인 : show running-config interface <INTERFACE>
                show interfaces status / show mac address-table interface <INTERFACE>
[2] 백업     : Running-config 출력 저장, 변경 전 상태 확정
[3] 경로 확인 : 관리 경로가 대상 Port·VLAN·Trunk를 경유하는지 확인
                Console 또는 OOB 접근 확보
[4] 적용     : Maintenance Window에 단일 Port·단일 Session부터
[5] 검증     : show port-security / show storm-control / show monitor
                show interfaces status err-disabled / show logging
[6] 저장     : copy running-config startup-config
```

Rollback 명령:

```cisco
configure terminal
!
interface <INTERFACE>
 no switchport port-security
 no storm-control broadcast level
 no storm-control action shutdown
 no switchport trunk native vlan
!
no monitor session <NUMBER>
```

Rollback 검증:

```cisco
show port-security interface <INTERFACE>
show storm-control <INTERFACE>
show monitor
show interfaces trunk
show interfaces status
```

> Sticky MAC 초기화는 `clear port-security sticky interface <INTERFACE>`를 사용하고, Running-config에서 실제로 제거됐는지 `show running-config interface <INTERFACE>`로 확인합니다.

---

## 6. 실무 체크리스트

- [ ] Port 역할별로 사용자·Phone·Server·Uplink·SPAN Destination 프로파일을 분리했는가?
- [ ] Maximum 값을 실제 Source MAC 개수 조사 결과로 산정했는가?
- [ ] Violation Mode를 가용성과 감사 요구에 맞게 선택했는가?
- [ ] Sticky MAC을 배선 검증 후 활성화하고 Startup-config에 저장했는가?
- [ ] Sticky와 Aging 병행 시 플랫폼 지원 여부를 확인했는가?
- [ ] Err-disable 자동 복구 전에 원인 제거 절차가 있는가?
- [ ] Storm-control 임계치를 Baseline 측정 후 결정했는가?
- [ ] Uplink에 `storm-control action shutdown`을 무분별하게 적용하지 않았는가?
- [ ] SPAN Destination Port가 서비스 Port가 아닌지 확인했는가?
- [ ] Port-security와 SPAN Destination을 같은 Port에 지정하지 않았는가?
- [ ] RSPAN VLAN을 전용 VLAN으로 지정하고 경로 Trunk에 허용했는가?
- [ ] SPAN Session 수 제한을 플랫폼 문서로 확인했는가?
- [ ] 미사용 Port를 Shutdown하고 격리 VLAN에 할당했는가?
- [ ] Native VLAN을 양쪽 Trunk에 동일하게 명시했는가?
- [ ] PortFast와 BPDU Guard를 End Device Port에만 적용했는가?

---

> 📌 **핵심 요약**
> - Port-security는 Frame의 Source MAC을 기준으로 Access Port 접근을 제한한다.
> - Source MAC은 학습 기준, Destination MAC은 전달 결정 기준이며 Flooding 범위는 해당 VLAN의 Broadcast Domain이다.
> - Secure MAC은 Static·Dynamic·Sticky로 구분되고 Sticky는 Running-config에 기록된다.
> - Violation Mode는 protect·restrict·shutdown 모두 위반 Frame을 폐기하지만 로그·Trap·Counter·Port 상태가 다르다.
> - 기본값은 Maximum 1, Violation shutdown이다.
> - Maximum은 IP Phone, Hypervisor, Teaming 등 실제 MAC 개수를 조사한 뒤 산정한다.
> - Aging Time 단위는 분이고 Err-disable Recovery Interval 단위는 초다.
> - Err-disable 자동 복구는 원인 제거를 대체하지 않는다.
> - Storm-control은 Port 단위로 Broadcast·Multicast·Unicast 폭주를 억제한다.
> - Storm-control action은 미지정 시 억제, `trap`은 알림, `shutdown`은 Err-disable 전환이다.
> - SPAN Destination Port는 일반 Forwarding·MAC 학습·STP에 참여하지 않는 전용 Port가 된다.
> - RSPAN은 전용 `remote-span` VLAN과 경로 Trunk Allowed VLAN이 모두 충족되어야 동작한다.
> - SPAN Session 수와 VLAN SPAN 제약은 플랫폼별로 다르므로 문서로 확인한다.
> - VLAN은 Broadcast Domain 분리이지 완전한 보안 경계가 아니다.
> - PortFast는 End Device Port 전용이며 BPDU Guard와 함께 사용한다.
> - 관련: 6-1. 🔐 Port-security & Storm-control · 6-2. 🔎 SPAN (Port Mirroring) · 6-3. 🧩 Switch 보안 통합 정리 · 6-4. 🚑 Switch 보안 트러블슈팅 치트시트 · 6-5. ⚡ Switch 보안 명령어 퀵 레퍼런스 · 3-3-2. 🧩 스위치 & 방어 보안 통합 정리
