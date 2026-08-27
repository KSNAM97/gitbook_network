# STP·RSTP·MSTP 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #STP #PVST #RSTP #MSTP #LoopGuard #RootGuard  
> **핵심 요약:** STP 장애는 **물리 Link → STP Mode → VLAN/Instance → Root Bridge → Port Role → Cost/Tie-Breaker → 보호 기능 → Log** 순서로 확인한다. Blocking/Discarding은 대부분 정상적인 Loop 방지 결과이므로, Port를 강제로 활성화하기 전에 왜 차단되었는지 확인해야 한다.

---

## 1. 공통 진단 순서

```properties
[1] 물리 Port와 Trunk가 Up인가?
[2] 양쪽 Switch의 STP Mode가 호환되는가?
[3] 해당 VLAN이 Trunk에 존재하고 허용되는가?
[4] 현재 Root Bridge가 의도한 장비인가?
[5] Root Port와 Designated Port가 예상 위치인가?
[6] Blocking/Discarding Port가 정상적인 중복 경로인가?
[7] Cost, Priority, Port Priority 변경이 있는가?
[8] MST Region 설정이 완전히 일치하는가?
[9] Root/Loop/BPDU Guard가 Port를 차단했는가?
[10] MAC Flapping이나 Broadcast Storm Log가 있는가?
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 예상하지 않은 Switch가 Root | Priority 미설정 | `show spanning-tree root` | Root/Secondary 수동 지정 |
| 예상 경로가 Blocking | Cost 또는 Tie-Breaker | `show spanning-tree vlan` | Cost/Priority 조정 |
| Link 장애 후 수렴이 느림 | Classic STP, Legacy Neighbor | STP Protocol 확인 | Rapid-PVST+/MSTP 통일 |
| RSTP인데 30초 이상 지연 | 802.1D Fallback, Shared Link | Port Detail 확인 | Mode/Duplex/Link Type 수정 |
| MST Port가 Boundary로 표시 | Region 설정 불일치 | MST Configuration 비교 | Name/Revision/Mapping 통일 |
| Port가 Err-disabled | BPDU Guard | Status와 Log 확인 | 오연결 제거 후 복구 |
| Port가 Root-inconsistent | Root Guard + Superior BPDU | Inconsistent Port 확인 | 하위 Priority 수정 |
| Port가 Loop-inconsistent | Loop Guard + BPDU 유실 | STP Detail/Log | 단방향 Link와 BPDU Filter 제거 |
| MAC Flapping 반복 | L2 Loop | MAC Table/Log/STP | Loop 경로 격리 |
| PortFast Port에서 Loop | Switch 오연결, Guard 없음 | Port 설정 확인 | BPDU Guard 적용 |
| 특정 VLAN만 Loop/차단 | PVST Root/Cost 차이 | VLAN별 STP 확인 | VLAN별 정책 수정 |
| Root가 자주 변경됨 | Rogue Switch, Link 불안정 | Root Change Log | Root Guard와 Priority 설계 |

---

## 3. 핵심 진단 명령어

### STP 전체

```cisco
show spanning-tree
show spanning-tree summary
show spanning-tree root
show spanning-tree detail
```

### VLAN별 확인

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree vlan <VLAN-ID> detail
```

### Interface별 확인

```cisco
show spanning-tree interface <INTERFACE> detail
show interfaces <INTERFACE>
show interfaces <INTERFACE> switchport
show interfaces trunk
```

### MSTP

```cisco
show spanning-tree mst
show spanning-tree mst <INSTANCE>
show spanning-tree mst configuration
```

### 보호 기능과 Log

```cisco
show spanning-tree inconsistentports
show interfaces status err-disabled
show errdisable recovery
show logging
```

### MAC Flapping

```cisco
show mac address-table dynamic
show mac address-table address <MAC>
show logging | include MAC|SPANTREE
```

---

## 4. 대표 장애 시나리오

### 시나리오 1. 예상하지 않은 Access Switch가 Root Bridge가 됨

#### 증상

```cisco
show spanning-tree root
```

출력에서 Access 계층 Switch가 Root로 표시됩니다.

#### 원인

모든 Switch가 기본 Priority를 사용하여 MAC Address가 가장 낮은 장비가 Root로 선출됐습니다.

#### 확인

```cisco
show spanning-tree vlan 10
show spanning-tree root
```

#### 조치

Primary Root:

```cisco
spanning-tree vlan 10,20 root primary
```

Secondary Root:

```cisco
spanning-tree vlan 10,20 root secondary
```

또는 명시적인 Priority:

```cisco
spanning-tree vlan 10,20 priority 4096
```

> Root 변경은 여러 Port Role과 Traffic 경로를 동시에 바꿀 수 있으므로 Maintenance Window에서 수행합니다.

---

### 시나리오 2. Port가 Blocking이어서 장애라고 판단함

Blocking/Discarding은 중복 경로에서 정상적인 Loop 방지 동작일 수 있습니다.

#### 확인

```cisco
show spanning-tree vlan 10
show spanning-tree interface GigabitEthernet0/2 detail
```

확인 항목:

- Root ID
- Root Path Cost
- Port Role
- Port State
- Designated Bridge ID
- Designated Port ID

#### 판단

```properties
중복 경로가 존재
+ 하나의 Port가 Blocking/Discarding
= 일반적으로 정상

중복 경로가 없는데 Blocking
= Root Guard, Loop Guard, STP 불일치 등 추가 확인
```

Port를 살리기 위해 STP를 끄면 Loop가 발생할 수 있습니다.

---

### 시나리오 3. 예상과 다른 Link가 Root Port로 선택됨

#### 원인 후보

1. Path Cost가 더 낮음
2. Sender Bridge ID가 더 낮음
3. Sender Port Priority가 더 낮음
4. Local Port ID가 더 낮음
5. EtherChannel과 개별 Link의 Cost 차이
6. Short/Long Cost 방식 차이

#### 확인

```cisco
show spanning-tree vlan 10
show spanning-tree summary
show running-config interface GigabitEthernet0/1
show running-config interface GigabitEthernet0/2
```

#### 조치

덜 선호할 Port의 Cost를 높입니다.

```cisco
interface GigabitEthernet0/2
 spanning-tree vlan 10 cost 50
```

또는 선호 Port의 Cost를 낮춥니다.

```cisco
interface GigabitEthernet0/1
 spanning-tree vlan 10 cost 10
```

변경 후:

```cisco
show spanning-tree vlan 10
```

---

### 시나리오 4. Classic STP Link 장애 후 복구가 느림

#### 증상

Link 장애 후 약 30초 또는 일부 경우 약 50초 동안 통신이 끊깁니다.

#### 원인

Classic 802.1D STP가 다음 단계를 수행할 수 있습니다.

```properties
Max Age 대기: 장애 유형에 따라 최대 약 20초
Listening: 약 15초
Learning: 약 15초
```

#### 확인

```cisco
show spanning-tree summary
show spanning-tree vlan 10
```

#### 개선

모든 관련 Switch에서 Rapid-PVST+를 사용합니다.

```cisco
spanning-tree mode rapid-pvst
```

> STP Mode 변경은 전체 VLAN의 STP 재수렴을 일으킬 수 있으므로 한 대씩 임의 변경하지 말고 Migration Plan을 수립합니다.

---

### 시나리오 5. Rapid-PVST+인데 수렴이 느림

#### 원인 후보

- 상대 Switch가 PVST+/802.1D
- Port가 Shared Link로 동작
- Half-Duplex 설정
- Proposal/Agreement 실패
- BPDU Filter
- 단방향 Link
- STP Mode 불일치

#### 확인

```cisco
show spanning-tree summary
show spanning-tree interface GigabitEthernet0/1 detail
show interfaces GigabitEthernet0/1
```

확인할 내용:

```properties
Protocol = rstp
Link Type = P2p
Port Type = Edge 또는 Network 조건
Duplex = Full
```

필요하면 Link Type을 명시할 수 있습니다.

```cisco
interface GigabitEthernet0/1
 spanning-tree link-type point-to-point
```

> 물리 환경이 실제 Point-to-Point인지 확인한 뒤 설정합니다.

---

### 시나리오 6. MST Switch가 같은 Region으로 묶이지 않음

#### 확인

각 Switch에서 다음 출력을 비교합니다.

```cisco
show spanning-tree mst configuration
```

필수 일치 항목:

```properties
Region Name
Revision Number
VLAN-to-Instance Mapping
```

#### 잘못된 예

```properties
SW1: Name CAMPUS, Revision 10, VLAN 10-20 → Instance 1
SW2: Name CAMPUS, Revision 10, VLAN 10-19 → Instance 1
```

VLAN 하나가 다르므로 다른 Region으로 인식될 수 있습니다.

#### 수정

```cisco
spanning-tree mst configuration
 name CAMPUS
 revision 10
 instance 1 vlan 10-20
 exit
```

변경 후:

```cisco
show spanning-tree mst
show spanning-tree mst configuration
```

---

### 시나리오 7. MST Instance의 Root가 예상과 다름

MST0의 Root와 MST1의 Root는 다를 수 있습니다.

#### 확인

```cisco
show spanning-tree mst 0
show spanning-tree mst 1
show spanning-tree mst 2
```

#### 수정

```cisco
spanning-tree mst 1 root primary
spanning-tree mst 2 root secondary
```

다른 Switch:

```cisco
spanning-tree mst 1 root secondary
spanning-tree mst 2 root primary
```

> VLAN이 어느 Instance에 매핑되었는지도 함께 확인해야 합니다.

---

### 시나리오 8. PortFast Port가 Err-disabled 됨

#### 증상

```properties
%SPANTREE-2-BLOCK_BPDUGUARD
```

#### 확인

```cisco
show interfaces status err-disabled
show logging
show running-config interface GigabitEthernet0/10
```

#### 원인

PortFast/BPDU Guard Port에서 BPDU를 수신했습니다.

가능한 원인:

- 사용자가 소형 Switch 연결
- IP Phone 뒤에 Switch 연결
- 잘못된 Patch
- 단말이 Bridge 기능 수행
- Switch 간 Link를 Access Port로 오인

#### 복구

먼저 BPDU 발생 장비 또는 Cable을 제거한 뒤:

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

---

### 시나리오 9. Root-inconsistent 상태

#### 확인

```cisco
show spanning-tree inconsistentports
show spanning-tree vlan 10
show logging
```

#### 원인

Root Guard Port에서 현재 Root보다 우수한 Superior BPDU를 수신했습니다.

#### 조치

하위 Switch의 잘못된 Priority를 수정합니다.

```cisco
no spanning-tree vlan 10 priority 0
```

또는 올바른 Priority:

```cisco
spanning-tree vlan 10 priority 32768
```

Superior BPDU가 사라지면 Port는 자동 복구됩니다.

> Root Guard를 제거하는 것보다 Superior BPDU가 발생한 원인을 먼저 수정합니다.

---

### 시나리오 10. Loop-inconsistent 상태

#### 확인

```cisco
show spanning-tree inconsistentports
show spanning-tree interface GigabitEthernet0/1 detail
show logging
```

#### 원인 후보

- 단방향 Fiber Link
- 상대 Port의 BPDU Filter
- Control Plane 혼잡
- Link는 Up이지만 BPDU만 수신되지 않음
- 중간 장비가 BPDU를 전달하지 않음

#### 조치

1. Fiber TX/RX 및 광모듈 확인
2. 양쪽 Error Counter 확인
3. BPDU Filter 제거
4. UDLD 상태 확인
5. 정상 BPDU 수신 복구

```cisco
show interfaces GigabitEthernet0/1 counters errors
show udld interface GigabitEthernet0/1
```

정상 BPDU가 다시 수신되면 Loop-inconsistent 상태는 자동 복구됩니다.

---

### 시나리오 11. BPDU Filter 적용 후 Loop 발생

#### 원인

Interface별 BPDU Filter가 BPDU 송수신을 억제하여 STP가 실제 Switch 연결을 감지하지 못했습니다.

```cisco
interface GigabitEthernet0/10
 spanning-tree bpdufilter enable
```

#### 조치

```cisco
interface GigabitEthernet0/10
 no spanning-tree bpdufilter enable
 spanning-tree bpduguard enable
```

단말 Port라면:

```cisco
spanning-tree portfast
spanning-tree bpduguard enable
```

> 단순히 BPDU가 보기 싫다는 이유로 BPDU Filter를 사용하면 안 됩니다.

---

### 시나리오 12. MAC Flapping과 Broadcast Storm 발생

#### 증상

- 동일 MAC이 여러 Port에서 반복 학습
- CPU 상승
- Ping Loss
- Trunk Traffic 급증
- STP Topology Change 반복

#### 확인

```cisco
show logging | include MAC|SPANTREE
show mac address-table address <MAC>
show spanning-tree detail
show interfaces counters errors
```

#### 긴급 조치

1. MAC이 이동하는 두 Port 식별
2. 의심 Link 중 영향이 적은 쪽을 격리
3. Traffic 안정화 확인
4. STP Mode, VLAN, BPDU Filter, EtherChannel 확인
5. 원인 제거 후 단계적으로 복구

```cisco
interface <SUSPECT-INTERFACE>
 shutdown
```

> 무작위로 여러 Uplink를 Shutdown하면 장애 범위가 커질 수 있으므로 MAC 이동 경로와 토폴로지를 기준으로 격리합니다.

---

### 시나리오 13. Root Bridge가 반복적으로 변경됨

#### 확인

```cisco
show spanning-tree detail
show logging | include ROOT|SPANTREE
show spanning-tree root
```

#### 원인 후보

- 신규 Switch의 낮은 Priority
- Access Port에 연결된 Rogue Switch
- Root Uplink Flapping
- VLAN별 Priority 불일치
- VTP로 새 VLAN이 생성됐으나 Root 정책 미적용

#### 예방

Root 수동 지정:

```cisco
spanning-tree vlan 10,20 root primary
```

하위 방향 Root Guard:

```cisco
interface GigabitEthernet0/20
 spanning-tree guard root
```

단말 Port BPDU Guard:

```cisco
interface range GigabitEthernet0/1 - 18
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

### 시나리오 14. UplinkFast 적용 후 Priority와 Cost가 예상과 다름

UplinkFast는 Legacy PVST 환경에서 빠른 Uplink 전환을 위해 Bridge Priority와 Port Cost에 영향을 줄 수 있습니다.

#### 확인

```cisco
show spanning-tree summary
show spanning-tree vlan 10
show running-config | include uplinkfast
```

현대 환경이라면 Rapid-PVST+ 또는 MSTP 전환을 검토합니다.

```cisco
spanning-tree mode rapid-pvst
```

> Mode 전환 전에 UplinkFast/BackboneFast와 기존 STP 정책의 호환성을 확인합니다.

---

## 5. 안전한 변경 절차

```properties
[1] 현재 Root와 Secondary Root 기록
[2] VLAN/Instance별 Port Role 기록
[3] STP Mode와 Path Cost Method 확인
[4] 변경 대상 Port와 예상 Blocking 위치 계산
[5] Root/Cost/Priority를 한 항목씩 변경
[6] STP 재수렴 대기
[7] Root/Port Role/Traffic 경로 검증
[8] MAC Flapping과 Log 확인
[9] 이상 시 기존 값으로 Rollback
```

변경 전:

```cisco
show spanning-tree summary
show spanning-tree root
show spanning-tree vlan <VLAN-ID>
show spanning-tree inconsistentports
show interfaces trunk
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
> - Blocking/Discarding은 대부분 정상적인 Loop 방지 상태다.
> - 예상하지 않은 Root는 Priority 미설정부터 확인한다.
> - 경로 선택은 Cost와 BPDU Tie-Breaker를 순서대로 확인한다.
> - RSTP가 느리면 Legacy Neighbor, Shared Link, BPDU Filter를 확인한다.
> - MST Region은 Name, Revision, VLAN Mapping이 모두 같아야 한다.
> - BPDU Guard는 Err-disabled, Root Guard는 Root-inconsistent를 만든다.
> - Loop Guard는 BPDU 유실 시 Loop-inconsistent 상태로 보호한다.
> - BPDU Filter는 STP를 보이지 않게 만들어 Loop를 유발할 수 있다.
> - MAC Flapping이 발생하면 실제 Loop 경로를 추적해 격리한다.
> - 관련: 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence · 3-2. ⚡ Fast 계열 (PortFast, UplinkFast, BackboneFast) · 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard) · 3-4. 🚀 RSTP & MSTP (초대형망 STP) · 3-5. 🧩 STP·RSTP·MSTP 통합 정리 · 3-6. 🚑 STP·RSTP·MSTP 트러블슈팅 치트시트 · 3-7. ⚡ STP·RSTP·MSTP 퀵 레퍼런스
