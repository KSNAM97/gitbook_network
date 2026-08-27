# 🚑 Switch 보안 트러블슈팅 치트시트

> **Tag:** #Network #Switch #Security #PortSecurity #SPAN #Cisco  
> **핵심 요약:** Switch 보안 장애는 **Port 물리 상태 → Err-disable 원인 → Port-security Violation → Storm-control Counter → SPAN Session → VLAN/Trunk → L3 Gateway** 순서로 확인한다. Port가 Down이면 원인이 Port-security인지 Storm-control인지 BPDU Guard인지 먼저 분리해야 하며, `show interfaces status err-disabled`와 `show logging`이 그 분기점이다. SPAN이 잡히지 않는 문제는 대부분 방향 지정, Destination Port 충돌, RSPAN VLAN 경로 누락 중 하나다.

---

## 1. 🎯 공통 진단 순서

```properties
[1]  Port가 물리적으로 Up인가?                 show interfaces status
[2]  Err-disabled 상태인가?                   show interfaces status err-disabled
[3]  Err-disable 원인이 무엇인가?              show logging / show errdisable detect
[4]  Violation Count가 증가하는가?             show port-security interface <INTERFACE>
[5]  Last Source Address가 예상한 MAC인가?
[6]  Secure MAC 목록이 실제 장비와 일치하는가?   show port-security address
[7]  Storm-control 현재값이 임계치에 근접하는가? show storm-control <INTERFACE>
[8]  Storm-control action이 shutdown인가?
[9]  SPAN Source·Destination·방향이 의도대로인가? show monitor session all
[10] RSPAN VLAN이 경로 Trunk에 허용됐는가?      show vlan remote-span
[11] Access VLAN과 Trunk Native VLAN이 올바른가? show interfaces trunk
[12] SVI와 L3 Gateway 상태가 정상인가?         show ip interface brief
```

```properties
Port Down
    ↓ Err-disabled 여부 확인
psecure-violation → Port-security 경로로 진단
storm-control     → Storm-control 경로로 진단
bpduguard         → STP 보호 경로로 진단
link-flap / 기타  → 물리·케이블·SFP 경로로 진단
```

---

## 2. 🛠️ 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| Port가 Secure-shutdown | Port-security Violation (shutdown Mode) | `show port-security interface <INTERFACE>` | 원인 MAC 확인 → Secure MAC 재등록 → `shutdown`/`no shutdown` |
| 장비 교체 후 통신 불가 | 이전 장비 MAC이 Sticky로 고정 | `show port-security address` | `clear port-security sticky interface <INTERFACE>` 후 재학습 |
| Violation Count만 증가, Port는 Up | `restrict`/`protect`의 정상 동작 | `show port-security interface <INTERFACE>` | 초과 MAC 원인 제거 또는 Maximum 조정 |
| 위반이 발생하는데 로그가 없음 | `protect` Mode | `show running-config interface <INTERFACE>` | `restrict` 이상으로 변경 |
| Port-security 명령이 거부됨 | Access Port 아님, SPAN/EtherChannel 충돌 | `show interfaces <INTERFACE> switchport` | Port 역할 정리 후 재적용 |
| Sticky MAC이 재부팅 후 사라짐 | Startup-config 미저장 | `show startup-config interface <INTERFACE>` | `copy running-config startup-config` |
| Port가 반복적으로 Err-disable | 근본 원인 미제거 + 자동 복구 | `show logging`, `show errdisable recovery` | 원인 제거 우선, Interval 상향 |
| Storm-control로 Uplink 차단 | `action shutdown` + 실제 Storm | `show storm-control <INTERFACE>` | 원인 제거 → `trap` 전환 검토 |
| 정상 Traffic이 억제됨 | 임계치 과소 설정 | `show storm-control <INTERFACE>` | Baseline 측정 후 임계치 상향 |
| Multicast 응용 장애 | Multicast 임계치 과소 | `show storm-control multicast` | 임계치 상향 또는 제거 |
| SPAN에 Traffic이 안 잡힘 | 방향 오류, Destination 충돌 | `show monitor session <NUMBER>` | 방향 재지정, Destination 전용화 |
| SPAN Destination Port 통신 불가 | Destination Port의 정상 동작 | `show monitor` | 전용 Port로 분리 |
| SPAN Session 생성 거부 | Session 수 제한 초과 | `show monitor session all` | 불필요 Session 삭제 |
| RSPAN Traffic 미도달 | VLAN 불일치 또는 Trunk 미허용 | `show vlan remote-span`, `show interfaces trunk` | VLAN 일치 및 Allowed VLAN 추가 |
| Native VLAN 변경 후 관리 단절 | 한쪽만 변경 | `show interfaces trunk` | Console/OOB로 양쪽 일치 |
| 미사용 Port로 비인가 접속 | Shutdown·격리 누락 | `show interfaces status` | Shutdown + 격리 VLAN 할당 |
| VLAN을 나눴는데 접근됨 | L3 Gateway 경유 정상 동작 | `show ip route`, `show access-lists` | ACL·Firewall 정책 추가 |

---

## 3. 🔍 핵심 진단 명령어

### Port 상태

```cisco
show interfaces status
show interfaces status err-disabled
show interfaces <INTERFACE> switchport
show interfaces description
```

### Port-security

```cisco
show port-security
show port-security interface <INTERFACE>
show port-security address
show running-config interface <INTERFACE>
```

### Storm-control

```cisco
show storm-control <INTERFACE>
show storm-control broadcast
show storm-control multicast
show interfaces <INTERFACE> counters
```

### SPAN / RSPAN

```cisco
show monitor
show monitor session <NUMBER>
show monitor session all
show vlan remote-span
show interfaces trunk
```

### Err-disable · VLAN · STP · 로그

```cisco
show errdisable recovery
show errdisable detect
show mac address-table interface <INTERFACE>
show vlan brief
show spanning-tree interface <INTERFACE> detail
show logging
```

---

## 4. 🚨 대표 장애 시나리오

### 시나리오 1. PC 교체 후 Port가 Secure-shutdown 상태가 됨

#### 증상 → 원인

```properties
Port Status = Secure-shutdown, 통신 불가
원인 = 이전 PC MAC이 Static/Sticky Secure MAC으로 남아 있고
       신규 PC의 다른 MAC이 Violation 처리
```

#### 확인

```cisco
show interfaces status err-disabled
show port-security interface <INTERFACE>
show port-security address
show logging
```

판독 포인트:

```properties
Port Status         = Secure-shutdown
Violation Count     = 1 이상
Last Source Address = 신규 PC MAC
```

#### 조치

```cisco
clear port-security sticky interface <INTERFACE>
!
configure terminal
interface <INTERFACE>
 shutdown
 no shutdown
```

검증:

```cisco
show port-security interface <INTERFACE>
show port-security address
show interfaces status
```

> 장비 교체가 예정된 Port에는 Static 고정보다 Sticky + `inactivity` Aging 조합이 운영 부담을 줄입니다.

---

### 시나리오 2. Violation Count는 증가하는데 Port는 Up 상태임

#### 증상 → 원인

```properties
증상 = 간헐적 통신 실패, Port는 Up
원인 = Violation Mode가 restrict 또는 protect (정상 동작)

restrict = 위반 Frame 폐기 + 로그/Trap/Counter
protect  = 위반 Frame 폐기 (기록 거의 없음)
```

#### 확인

```cisco
show port-security interface <INTERFACE>
show port-security address
show mac address-table interface <INTERFACE>
```

#### 조치

허용해야 할 장비가 실제로 늘어난 경우에만 Maximum을 조정합니다.

```cisco
configure terminal
interface <INTERFACE>
 switchport port-security maximum 2
```

검증:

```cisco
show port-security interface <INTERFACE>
show mac address-table interface <INTERFACE>
```

> Maximum 상향은 가장 쉬운 조치이지만 보안 수준을 낮춥니다. 어떤 MAC이 추가로 들어왔는지 확인한 뒤 결정하십시오. 비인가 장비가 원인이면 Maximum을 올리지 않고 물리 연결을 정리합니다.

---

### 시나리오 3. Port-security 설정 명령이 거부됨

#### 원인 후보

1. Port가 `switchport mode access`가 아님 (Dynamic Auto/Desirable)
2. Port가 Trunk이며 플랫폼이 Trunk Port-security를 제한
3. Port가 EtherChannel Member
4. Port가 SPAN Destination으로 지정됨
5. Port가 Routed Interface(`no switchport`) 상태

#### 확인

```cisco
show interfaces <INTERFACE> switchport
show running-config interface <INTERFACE>
show monitor
show etherchannel summary
```

#### 조치

```cisco
configure terminal
!
no monitor session <NUMBER>
!
interface <INTERFACE>
 switchport
 switchport mode access
 switchport nonegotiate
 switchport port-security
```

검증:

```cisco
show interfaces <INTERFACE> switchport
show port-security interface <INTERFACE>
show monitor
```

> Port-security와 SPAN Destination, EtherChannel Member는 같은 Port에서 병행할 수 없습니다. Port 역할을 먼저 확정하십시오.

---

### 시나리오 4. Sticky MAC이 재부팅 후 사라짐

#### 원인

```properties
Sticky 학습 → Running-config 기록
저장 안 함  → 재부팅 후 소실
저장       → 재부팅 후 유지
```

#### 확인

```cisco
show running-config interface <INTERFACE>
show startup-config interface <INTERFACE>
show port-security address
```

#### 조치

```cisco
copy running-config startup-config
```

검증:

```cisco
show startup-config interface <INTERFACE>
show port-security address
```

> 저장하지 않은 상태에서 재부팅되면 잘못된 장비의 MAC이 새로 고정될 수 있습니다. Sticky 학습 직후 저장 절차를 표준화하십시오.

---

### 시나리오 5. Err-disabled Port가 자동 복구 후 다시 차단됨

#### 원인

```properties
자동 복구 = Port를 다시 Up
자동 복구 ≠ 원인 제거
원인 지속 → 복구 → 재위반 → 재차단 반복
```

#### 확인

```cisco
show interfaces status err-disabled
show errdisable recovery
show port-security interface <INTERFACE>
show storm-control <INTERFACE>
show logging
```

#### 조치

1. 로그에서 Err-disable 원인을 확정합니다.
2. Port-security 원인이면 Last Source Address의 장비를 확인합니다.
3. Storm-control 원인이면 Loop 또는 폭주 원인을 제거합니다.
4. BPDU Guard 원인이면 해당 Port의 비인가 Switch 연결을 정리합니다.
5. 원인 제거 후 복구 Interval을 조정합니다.

```cisco
configure terminal
!
errdisable recovery interval 300
```

검증:

```cisco
show errdisable recovery
show interfaces status
show logging
```

> 복구 Interval을 30초 수준으로 짧게 설정하면 Port Flapping과 STP 재수렴이 반복되어 장애 범위가 오히려 확대됩니다.

---

### 시나리오 6. Storm-control 적용 후 정상 Traffic이 차단됨

#### 원인 후보

- 임계치를 Baseline보다 낮게 설정
- `unicast` Storm-control이 플랫폼 구현상 전체 Unicast에 적용
- Multicast 임계치가 정상 Multicast 응용을 억제
- Upper/Lower를 동일하게 설정해 임계치 부근에서 반복 억제

#### 확인

```cisco
show storm-control <INTERFACE>
show storm-control broadcast
show storm-control unicast
show interfaces <INTERFACE> counters
```

판독 포인트:

```properties
Current 값이 Upper에 근접 또는 초과
Filter State = Forwarding / Blocking
```

#### 조치

```cisco
configure terminal
interface <INTERFACE>
 storm-control broadcast level 30.00 20.00
 no storm-control unicast level
```

검증:

```cisco
show storm-control <INTERFACE>
show interfaces <INTERFACE> counters
```

> `unicast` Storm-control이 Unknown Unicast만 대상인지 전체 Unicast 대상인지는 플랫폼별 지원 여부 확인 필요입니다. 정상 서비스 Traffic이 Unicast인 환경에서는 적용을 신중히 검토하십시오.

---

### 시나리오 7. Storm-control로 Uplink가 Err-disable되어 광범위 단절 발생

#### 원인

```properties
Storm 발생
    ↓ Uplink에 storm-control action shutdown 적용됨
Uplink Err-disable
    ↓
하위 Segment 전체 단절 + STP 재수렴
```

#### 확인

```cisco
show interfaces status err-disabled
show storm-control <INTERFACE>
show spanning-tree summary
show logging
```

#### 조치

근본 원인인 Loop 또는 폭주 지점을 먼저 격리한 뒤 Uplink 정책을 재검토합니다.

```cisco
configure terminal
interface <INTERFACE>
 storm-control action trap
 shutdown
 no shutdown
```

검증:

```cisco
show storm-control <INTERFACE>
show interfaces status
show spanning-tree summary
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Uplink에는 즉시 차단보다 알림과 모니터링을 우선하고, Access Edge에서 Storm 확산을 차단하는 설계가 안전합니다. 관련 STP 보호는 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard)를 참고하십시오.

---

### 시나리오 8. 모니터링 장비에 SPAN Traffic이 잡히지 않음

#### 확인

```cisco
show monitor session <NUMBER>
show interfaces status
show interfaces <INTERFACE> switchport
```

판독 포인트:

```properties
Type              = Local Session / Remote Source / Remote Destination
Source Ports      = Both / Rx / Tx 방향
Destination Ports = 실제 분석 장비 연결 Port
```

#### 원인 후보

1. 방향(`rx`/`tx`/`both`)이 원하는 방향과 다름
2. Destination Port 케이블 미연결 또는 Link Down
3. Destination Port가 Port-security·EtherChannel과 충돌
4. 분석 장비 NIC가 Promiscuous Mode가 아님
5. Source Port에 실제 Traffic이 흐르지 않음
6. 같은 Session Number 재사용으로 Source가 덮어써짐

#### 조치

```cisco
configure terminal
!
no monitor session <NUMBER>
!
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> destination interface <INTERFACE>
```

검증:

```cisco
show monitor session <NUMBER>
show interfaces <INTERFACE> counters
```

> Destination Port는 기본적으로 수신 Traffic을 폐기하고 MAC 학습과 STP에 참여하지 않습니다. Destination Port에서 Ping이 되지 않는 것은 정상입니다.

---

### 시나리오 9. SPAN Destination으로 지정한 Port의 서비스가 중단됨

#### 원인

```properties
Destination Port
= 일반 Forwarding 안 함
= MAC 학습 안 함
= STP 참여 안 함
= 수신 Traffic 기본 폐기
```

#### 확인

```cisco
show monitor
show interfaces <INTERFACE> switchport
show mac address-table interface <INTERFACE>
show interfaces status
```

#### 조치

Session을 삭제하고 미사용 Port로 Destination을 재지정합니다.

```cisco
configure terminal
!
no monitor session <NUMBER>
!
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> destination interface <INTERFACE>
```

검증:

```cisco
show monitor session <NUMBER>
show interfaces status
```

> Destination Port는 지정 전에 `show interfaces status`와 `show mac address-table interface <INTERFACE>`로 미사용 상태를 확인해야 합니다.

---

### 시나리오 10. SPAN Session 생성이 거부되거나 기존 Session이 사라짐

#### 원인

```properties
Session Number 재사용 = 기존 Session 설정 변경 또는 대체
Session 수 초과       = 명령 거부
```

#### 확인

```cisco
show monitor
show monitor session all
show running-config | include monitor session
```

#### 조치

```cisco
configure terminal
!
no monitor session <NUMBER>
!
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> destination interface <INTERFACE>
```

검증:

```cisco
show monitor session all
```

> 지원 Session 수는 플랫폼별 지원 여부 확인 필요입니다. Packet Tracer는 실장비의 Session 개수 제한과 자원 제약을 재현하지 않으므로, Packet Tracer에서 동작한 구성이 실장비에서 거부될 수 있습니다.

---

### 시나리오 11. RSPAN인데 원격 Switch에서 Traffic이 도달하지 않음

#### 확인

```cisco
show monitor session <NUMBER>
show vlan remote-span
show vlan brief
show interfaces trunk
```

#### 원인 후보

1. 양쪽 Switch의 RSPAN VLAN 번호 불일치
2. VLAN에 `remote-span`을 지정하지 않음
3. 경로상 Trunk의 Allowed VLAN에 RSPAN VLAN 누락
4. 경로 중간 Switch에 해당 VLAN이 존재하지 않음
5. Destination Switch에 `monitor session <NUMBER> source remote vlan <VLAN-ID>` 누락
6. Reflector Port를 요구하는 플랫폼에서 이미 사용 중인 Port를 지정

#### 조치

Source Switch:

```cisco
configure terminal
!
vlan <VLAN-ID>
 remote-span
!
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> destination remote vlan <VLAN-ID>
```

Destination Switch:

```cisco
configure terminal
!
vlan <VLAN-ID>
 remote-span
!
monitor session <NUMBER> source remote vlan <VLAN-ID>
monitor session <NUMBER> destination interface <INTERFACE>
```

경로 Trunk:

```cisco
interface <INTERFACE>
 switchport trunk allowed vlan add <VLAN-ID>
```

검증:

```cisco
show vlan remote-span
show monitor session <NUMBER>
show interfaces trunk
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> RSPAN VLAN에서는 MAC 학습이 수행되지 않으므로, 해당 VLAN에 Access Port를 할당하거나 SVI를 만들어 통신을 시도해도 동작하지 않습니다.

---

### 시나리오 12. VLAN SPAN 설정이 적용되지 않음

#### 원인 후보

- 같은 Session에 Source Interface와 Source VLAN을 혼용
- 플랫폼이 VLAN SPAN의 `tx`/`both` 방향을 제한
- Source VLAN으로 RSPAN VLAN을 지정
- 해당 VLAN에 활성 Port가 없어 Traffic 자체가 없음

#### 확인

```cisco
show monitor session <NUMBER>
show vlan id <VLAN-ID>
show mac address-table vlan <VLAN-ID>
```

#### 조치

```cisco
configure terminal
!
no monitor session <NUMBER>
!
monitor session <NUMBER> source vlan <VLAN-ID> rx
monitor session <NUMBER> destination interface <INTERFACE>
```

Trunk Source에서 특정 VLAN만 필요한 경우:

```cisco
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> filter vlan <VLAN-ID>
monitor session <NUMBER> destination interface <INTERFACE>
```

검증:

```cisco
show monitor session <NUMBER>
show interfaces trunk
```

> VLAN SPAN과 `filter vlan`의 지원 여부는 플랫폼별 지원 여부 확인 필요입니다. Packet Tracer에서는 두 기능이 지원되지 않거나 출력이 실장비와 다를 수 있습니다.

---

### 시나리오 13. Native VLAN 변경 후 관리 접근이 끊김

#### 원인

```properties
SW1 Native VLAN = <VLAN-ID>
SW2 Native VLAN = 다른 값
    ↓
Untagged Frame이 서로 다른 VLAN으로 해석
    ↓
관리 경로 단절 또는 VLAN 경계 모호
```

#### 확인

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
show cdp neighbors detail
show logging
```

#### 조치

Console 또는 OOB로 접근한 뒤 양쪽을 동일하게 설정합니다.

```cisco
configure terminal
interface <INTERFACE>
 switchport trunk native vlan <VLAN-ID>
 switchport trunk allowed vlan <VLAN-ID>
 switchport nonegotiate
```

Rollback:

```cisco
interface <INTERFACE>
 no switchport trunk native vlan
```

검증:

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
show ip interface brief
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Native VLAN 변경은 양쪽 Trunk를 같은 Maintenance Window에 처리합니다. 세부 동작은 1-2. 🚚 Switchport & Trunk & Inter-VLAN을 참고하십시오.

---

### 시나리오 14. VLAN을 분리했는데도 다른 VLAN에 접근됨

#### 원인

```properties
VLAN 분리 = Broadcast Domain 분리
VLAN 분리 ≠ 접근 제어 정책

L3 Gateway + Routing 존재 → VLAN 간 통신 가능 (정상 동작)
```

#### 확인

```cisco
show vlan brief
show ip interface brief
show ip route
show access-lists
```

#### 조치

```properties
[1] 필요 통신만 허용하는 ACL 또는 Firewall 정책 수립
[2] SVI 또는 Routed Interface에 정책 적용
[3] 미사용 VLAN은 SVI를 만들지 않고 격리 유지
[4] Trunk Allowed VLAN을 필요한 VLAN으로만 제한
```

검증:

```cisco
show access-lists
show ip interface <INTERFACE>
show interfaces trunk
```

> Inter-VLAN Routing 구조는 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing, VLAN 기초는 1-1. 🔌 스위치 동작원리 & VLAN 기초를 참고하십시오.

---

### 시나리오 15. 미사용 Port에 비인가 장비가 연결됨

#### 원인

미사용 Port가 Shutdown되지 않았고 기본 VLAN에 속한 상태로 방치됐습니다.

#### 확인

```cisco
show interfaces status
show interfaces description
show mac address-table
show interfaces <INTERFACE> switchport
```

#### 조치

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

> 격리 VLAN에는 SVI와 L3 Gateway를 만들지 않아야 격리 효과가 유지됩니다. 관리 VLAN이나 Native VLAN을 격리 VLAN으로 재사용하지 마십시오.

---

### 시나리오 16. BPDU Guard 차단을 Port-security 문제로 오진함

#### 원인

```properties
Err-disable 원인 분류
psecure-violation → Port-security
bpduguard         → 사용자 Port에서 BPDU 수신
storm-control     → 폭주 Traffic
```

Violation Count가 0인데 Port가 Err-disable이면 Port-security 원인이 아닙니다.

#### 확인

```cisco
show interfaces status err-disabled
show errdisable detect
show port-security interface <INTERFACE>
show spanning-tree interface <INTERFACE> detail
show logging
```

#### 조치

End Device Port가 아니라 Switch 연결 Port로 전환해야 하는 경우:

```cisco
configure terminal
interface <INTERFACE>
 no spanning-tree portfast
 no spanning-tree bpduguard enable
 switchport mode trunk
 switchport trunk allowed vlan <VLAN-ID>
 switchport nonegotiate
 shutdown
 no shutdown
```

검증:

```cisco
show spanning-tree interface <INTERFACE> detail
show interfaces trunk
show interfaces status
```

> PortFast는 End Device Port 전용이며 BPDU Guard와 함께 사용합니다. Uplink와 Trunk에는 이 조합을 적용하지 않습니다.

---

## 5. 🔧 안전한 변경 및 복구 절차

```properties
[1] 현재 확인 : show running-config interface <INTERFACE>
                show interfaces status / show port-security / show monitor session all
[2] 백업     : Running-config 출력 저장, 변경 전 상태 확정
[3] 경로 확인 : 관리 경로가 대상 Port·VLAN·Trunk를 경유하는지 확인
                Console 또는 OOB 접근 확보
[4] 적용     : Maintenance Window에 단일 Port·단일 Session부터
[5] 검증     : show 출력으로 Port Status·Violation Count·Current 값 확인
[6] 저장     : copy running-config startup-config
```

변경 전 상태 수집:

```cisco
show running-config
show interfaces status
show interfaces trunk
show port-security
show storm-control
show monitor session all
```

Rollback 명령:

```cisco
configure terminal
!
interface <INTERFACE>
 no switchport port-security
 no storm-control broadcast level
 no storm-control multicast level
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

Err-disable 확인 및 수동 복구:

```cisco
show interfaces status err-disabled
show errdisable recovery
!
configure terminal
interface <INTERFACE>
 shutdown
 no shutdown
```

> 복구 명령을 입력했다는 사실보다 `show` 출력에서 Port Status, Violation Count, Storm-control Current 값이 정상 범위인지 확인하는 것이 중요합니다.

---

> 📌 **핵심 요약**
> - Port Down 장애는 Err-disable 원인을 먼저 분리한다.
> - `psecure-violation`, `storm-control`, `bpduguard`는 서로 다른 원인이다.
> - Secure-shutdown은 Port-security shutdown Mode의 결과다.
> - Violation Count가 증가하는데 Port가 Up이면 restrict 또는 protect의 정상 동작이다.
> - Sticky MAC은 저장하지 않으면 재부팅 후 사라진다.
> - Err-disable 자동 복구는 원인 제거를 대체하지 않는다.
> - Storm-control 임계치는 Baseline 측정 후 결정한다.
> - Uplink의 `storm-control action shutdown`은 광범위 단절을 유발할 수 있다.
> - SPAN Destination Port에서 Ping이 되지 않는 것은 정상이다.
> - SPAN Session Number 재사용은 기존 Session을 대체한다.
> - RSPAN은 VLAN 번호 일치, `remote-span` 지정, 경로 Trunk 허용이 모두 필요하다.
> - Native VLAN은 양쪽 Trunk를 동시에 일치시켜야 한다.
> - VLAN 분리는 Broadcast Domain 분리이며 완전한 보안 경계가 아니다.
> - 미사용 Port는 Shutdown과 격리 VLAN으로 진입점을 축소한다.
> - 관련: 6-1. 🔐 Port-security & Storm-control · 6-2. 🔎 SPAN (Port Mirroring) · 6-3. 🧩 Switch 보안 통합 정리 · 6-4. 🚑 Switch 보안 트러블슈팅 치트시트 · 6-5. ⚡ Switch 보안 명령어 퀵 레퍼런스 · 3-3-2. 🧩 스위치 & 방어 보안 통합 정리
