# STP·RSTP·MSTP 퀵 레퍼런스

> **Tag:** #Network #QuickReference #STP #PVST #RapidPVST #MSTP #STPGuard  
> **핵심 요약:** STP의 Root 선출, Port Role, Cost 조정, Rapid-PVST+/MSTP 설정 및 보호 기능 명령어를 빠르게 확인하기 위한 암기 카드다.

---

## 1. STP 핵심 공식

```properties
Bridge ID
= Bridge Priority + Extended System ID + MAC Address

Root Bridge
= 가장 낮은 Bridge ID

Root Port
= Root까지 가장 우수한 BPDU를 수신하는 Port

Designated Port
= Segment에서 가장 우수한 BPDU를 광고하는 Port

Blocking/Discarding Port
= Loop 방지를 위해 Data 전달을 차단한 Port
```

---

## 2. Root Bridge 선출 순서

```properties
1. 가장 낮은 Bridge Priority
2. 가장 낮은 MAC Address
```

기본 Priority:

```properties
32768
```

PVST+ 표시값:

```properties
Priority + VLAN ID
예: 32768 + VLAN 10 = 32778
```

Priority 단위:

```properties
4096의 배수
```

---

## 3. Root Port Tie-Breaker

```properties
1. 가장 낮은 Root Bridge ID
2. 가장 낮은 Root Path Cost
3. 가장 낮은 Sender Bridge ID
4. 가장 낮은 Sender Port ID
5. 가장 낮은 Local Port ID
```

```properties
낮은 값 = 우선
높은 값 = 후순위
```

---

## 4. Root Bridge 설정

Primary Root:

```cisco
spanning-tree vlan 10,20 root primary
```

Secondary Root:

```cisco
spanning-tree vlan 10,20 root secondary
```

Priority 직접 지정:

```cisco
spanning-tree vlan 10,20 priority 4096
```

기본값 복원:

```cisco
no spanning-tree vlan 10,20 priority
```

확인:

```cisco
show spanning-tree root
show spanning-tree vlan 10
```

---

## 5. STP Cost

### Short Cost

| Bandwidth | Cost |
|---:|---:|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

### Long Cost

| Bandwidth | Cost |
|---:|---:|
| 10 Mbps | 2,000,000 |
| 100 Mbps | 200,000 |
| 1 Gbps | 20,000 |
| 10 Gbps | 2,000 |
| 100 Gbps | 200 |
| 1 Tbps | 20 |

Long Cost 사용:

```cisco
spanning-tree pathcost method long
```

Short Cost 사용:

```cisco
spanning-tree pathcost method short
```

확인:

```cisco
show spanning-tree summary
```

---

## 6. Cost 및 Port Priority

VLAN별 Cost:

```cisco
interface GigabitEthernet0/1
 spanning-tree vlan 10 cost 50
```

기본값 복원:

```cisco
interface GigabitEthernet0/1
 no spanning-tree vlan 10 cost
```

Port Priority:

```cisco
interface GigabitEthernet0/1
 spanning-tree vlan 10 port-priority 64
```

기본값 복원:

```cisco
interface GigabitEthernet0/1
 no spanning-tree vlan 10 port-priority
```

```properties
Cost가 낮을수록 선호
Port Priority가 낮을수록 선호
Port Priority 기본값 = 128
Port Priority 설정 단위 = 일반적으로 16
```

---

## 7. ⏳ Classic STP 상태

| 상태 | BPDU | MAC 학습 | Data 전달 |
|---|:---:|:---:|:---:|
| Disabled | X | X | X |
| Blocking | O | X | X |
| Listening | O | X | X |
| Learning | O | O | X |
| Forwarding | O | O | O |

전환:

```properties
Blocking
→ Listening 15초
→ Learning 15초
→ Forwarding
```

Timer:

```properties
Hello Time    = 2초
Max Age       = 20초
Forward Delay = 15초
```

---

## 8. ⏱ STP Timer

```cisco
spanning-tree vlan 10 hello-time 2
spanning-tree vlan 10 max-age 20
spanning-tree vlan 10 forward-time 15
```

기본값 복원:

```cisco
no spanning-tree vlan 10 hello-time
no spanning-tree vlan 10 max-age
no spanning-tree vlan 10 forward-time
```

> 근거 없이 Timer를 줄이지 말고 Rapid-PVST+ 또는 MSTP 사용을 우선 검토한다.

---

## 9. Rapid-PVST+

설정:

```cisco
spanning-tree mode rapid-pvst
```

확인:

```cisco
show spanning-tree summary
show spanning-tree vlan 10
```

RSTP 상태:

| 상태 | Data 전달 | MAC 학습 |
|---|:---:|:---:|
| Discarding | X | X |
| Learning | X | O |
| Forwarding | O | O |

RSTP Port Role:

```properties
Root
Designated
Alternate
Backup
Disabled
```

Rapid Transition 조건:

```properties
Edge Port
Point-to-Point Link
Proposal/Agreement
Alternate Port
호환 가능한 RSTP Neighbor
```

---

## 10. RSTP Link Type

Point-to-Point:

```cisco
interface GigabitEthernet0/1
 spanning-tree link-type point-to-point
```

Shared:

```cisco
interface GigabitEthernet0/1
 spanning-tree link-type shared
```

자동:

```cisco
interface GigabitEthernet0/1
 no spanning-tree link-type
```

```properties
Full-Duplex = 일반적으로 Point-to-Point
Half-Duplex = 일반적으로 Shared
```

---

## 11. MSTP

MST Mode:

```cisco
spanning-tree mode mst
```

Region 설정:

```cisco
spanning-tree mst configuration
 name CAMPUS
 revision 10
 instance 1 vlan 10,20
 instance 2 vlan 30,40
 exit
```

Instance Root:

```cisco
spanning-tree mst 1 root primary
spanning-tree mst 1 root secondary
```

Priority 직접 지정:

```cisco
spanning-tree mst 1 priority 4096
```

Cost:

```cisco
interface GigabitEthernet0/1
 spanning-tree mst 1 cost 50000
```

Port Priority:

```cisco
interface GigabitEthernet0/1
 spanning-tree mst 1 port-priority 64
```

확인:

```cisco
show spanning-tree mst configuration
show spanning-tree mst
show spanning-tree mst 0
show spanning-tree mst 1
```

---

## 12. MST Region 조건

```properties
Region Name 일치
AND
Revision Number 일치
AND
VLAN-to-Instance Mapping 일치
=
같은 MST Region
```

```properties
MST0 = IST
매핑하지 않은 VLAN = MST0
```

---

## 13. PortFast

Access Port:

```cisco
interface GigabitEthernet0/10
 switchport mode access
 spanning-tree portfast
```

Trunk Edge가 명확히 필요한 특수 환경:

```cisco
interface GigabitEthernet0/10
 spanning-tree portfast trunk
```

> `portfast trunk`는 Server, Hypervisor, Router 등 Loop를 만들지 않는 것이 확실한 Edge Trunk에서만 사용한다. Switch 간 Trunk에는 사용하지 않는다.

전역 기본:

```cisco
spanning-tree portfast default
```

확인:

```cisco
show spanning-tree interface GigabitEthernet0/10 detail
```

---

## 14. BPDU Guard

Interface별:

```cisco
interface GigabitEthernet0/10
 spanning-tree bpduguard enable
```

전역 PortFast Port 대상:

```cisco
spanning-tree portfast bpduguard default
```

비활성화:

```cisco
interface GigabitEthernet0/10
 spanning-tree bpduguard disable
```

복구:

```cisco
interface GigabitEthernet0/10
 shutdown
 no shutdown
```

자동 복구:

```cisco
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

확인:

```cisco
show interfaces status err-disabled
show logging
show errdisable recovery
```

---

## 15. BPDU Filter

Interface별:

```cisco
interface GigabitEthernet0/10
 spanning-tree bpdufilter enable
```

제거:

```cisco
interface GigabitEthernet0/10
 no spanning-tree bpdufilter enable
```

전역 PortFast 대상:

```cisco
spanning-tree portfast bpdufilter default
```

```properties
BPDU Filter
= STP의 이웃 감지를 방해할 수 있음
= 잘못 적용하면 L2 Loop 가능
= 일반적인 단말 Port에서는 BPDU Guard 우선
```

---

## 16. Root Guard

설정:

```cisco
interface GigabitEthernet0/20
 spanning-tree guard root
```

제거:

```cisco
interface GigabitEthernet0/20
 no spanning-tree guard root
```

Trigger:

```properties
Superior BPDU 수신
```

결과:

```properties
Root-inconsistent
```

복구:

```properties
Superior BPDU가 사라지면 자동 복구
```

확인:

```cisco
show spanning-tree inconsistentports
show spanning-tree vlan 10
show logging
```

---

## 17. Loop Guard

Interface별:

```cisco
interface GigabitEthernet0/1
 spanning-tree guard loop
```

전역 기본:

```cisco
spanning-tree loopguard default
```

제거:

```cisco
interface GigabitEthernet0/1
 no spanning-tree guard loop
```

Trigger:

```properties
수신하던 BPDU가 중단됨
```

결과:

```properties
Loop-inconsistent
```

복구:

```properties
정상 BPDU가 다시 수신되면 자동 복구
```

확인:

```cisco
show spanning-tree inconsistentports
show spanning-tree interface GigabitEthernet0/1 detail
```

---

## 18. Legacy Fast 기능

UplinkFast:

```cisco
spanning-tree uplinkfast
```

BackboneFast:

```cisco
spanning-tree backbonefast
```

확인:

```cisco
show spanning-tree summary
```

| 기능 | 목적 |
|---|---|
| PortFast | Edge Port 즉시 Forwarding |
| UplinkFast | 직접 Root Port 장애 빠른 전환 |
| BackboneFast | 간접 Link 장애 시 Max Age 단축 |
| Rapid-PVST+/MSTP | 현대적인 빠른 수렴 |

```properties
Rapid-PVST+/MSTP
→ UplinkFast/BackboneFast 기능이 프로토콜에 통합
→ 별도 Legacy Fast 설정 일반적으로 불필요
```

---

## 19. 필수 검증 명령어

전체:

```cisco
show spanning-tree
show spanning-tree summary
show spanning-tree root
show spanning-tree detail
```

VLAN:

```cisco
show spanning-tree vlan <VLAN-ID>
```

Interface:

```cisco
show spanning-tree interface <INTERFACE> detail
```

MST:

```cisco
show spanning-tree mst
show spanning-tree mst <INSTANCE>
show spanning-tree mst configuration
```

불일치 Port:

```cisco
show spanning-tree inconsistentports
```

Err-disabled:

```cisco
show interfaces status err-disabled
show errdisable recovery
```

Log:

```cisco
show logging
```

Debug:

```cisco
debug spanning-tree events
undebug all
```

---

## 20. 암기용 함정표

| 항목 | 실제 의미 |
|---|---|
| 낮은 Bridge ID | Root 선출에 유리 |
| 낮은 Path Cost | Root Port 선출에 유리 |
| 낮은 Port Priority | Tie-Breaker에 유리 |
| Root Bridge | Root Port가 없음 |
| Root Bridge Port | 일반적으로 Designated Forwarding |
| Blocking/Discarding | 대부분 정상적인 Loop 방지 |
| Alternate Port | RSTP의 Root Port Backup |
| PortFast | STP를 완전히 끄는 기능이 아님 |
| BPDU Guard | BPDU 수신 시 Err-disabled |
| BPDU Filter | BPDU 송수신을 억제해 Loop 위험 |
| Root Guard | Superior BPDU 수신 시 Root-inconsistent |
| Loop Guard | BPDU 유실 시 Loop-inconsistent |
| Root/Loop inconsistent | 원인 제거 시 자동 복구 |
| PVST 최대 128 Instance | 모든 플랫폼에 고정된 값이 아님 |
| Long Cost | MSTP만의 전용 Cost가 아님 |
| RSTP | 모든 환경에서 무조건 즉시 수렴하는 것은 아님 |
| MST Region | Name, Revision, VLAN Mapping 모두 일치 |
| UplinkFast/BackboneFast | Legacy STP용, RSTP에 기능 통합 |

---

## 21. 빠른 장애 진단 순서

```properties
Physical Link
    ↓
Trunk/VLAN
    ↓
STP Mode
    ↓
Root Bridge
    ↓
Root/Designated/Alternate Role
    ↓
Path Cost
    ↓
Port Priority와 Tie-Breaker
    ↓
MST Region
    ↓
Inconsistent/Err-disabled
    ↓
MAC Flapping 및 Log
```

---

## 22. 안전한 변경 절차

변경 전:

```cisco
show spanning-tree summary
show spanning-tree root
show spanning-tree vlan <VLAN-ID>
show spanning-tree inconsistentports
```

변경:

```properties
Root Priority
→ Cost
→ Port Priority
→ 보호 기능
순서로 한 항목씩 변경
```

변경 후:

```cisco
show spanning-tree root
show spanning-tree vlan <VLAN-ID>
show spanning-tree detail
show mac address-table dynamic
show logging
```

---

> 📌 **핵심 요약**
> - Root Primary: `spanning-tree vlan <ID> root primary`
> - Root Secondary: `spanning-tree vlan <ID> root secondary`
> - Cost: `spanning-tree vlan <ID> cost <VALUE>`
> - Rapid-PVST+: `spanning-tree mode rapid-pvst`
> - MSTP: `spanning-tree mode mst`
> - PortFast: `spanning-tree portfast`
> - BPDU Guard: `spanning-tree bpduguard enable`
> - Root Guard: `spanning-tree guard root`
> - Loop Guard: `spanning-tree guard loop`
> - 상태 확인: `show spanning-tree`
> - Root 확인: `show spanning-tree root`
> - 보호 상태: `show spanning-tree inconsistentports`
> - 관련: 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence · 3-2. ⚡ Fast 계열 (PortFast, UplinkFast, BackboneFast) · 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard) · 3-4. 🚀 RSTP & MSTP (초대형망 STP) · 3-5. 🧩 STP·RSTP·MSTP 통합 정리 · 3-6. 🚑 STP·RSTP·MSTP 트러블슈팅 치트시트 · 3-7. ⚡ STP·RSTP·MSTP 퀵 레퍼런스
