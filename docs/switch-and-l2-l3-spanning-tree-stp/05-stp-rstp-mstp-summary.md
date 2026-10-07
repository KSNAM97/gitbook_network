# STP·RSTP·MSTP 통합 정리

> **Tag:** #Network #Switch #STP #PVST #RSTP #RapidPVST #MSTP #LoopPrevention  
> **핵심 요약:** STP는 L2 이중화 환경에서 일부 경로를 논리적으로 차단해 Loop-Free Tree를 만든다. 가장 낮은 Bridge ID의 Switch가 Root Bridge가 되고, 각 Non-Root Switch는 최저 Root Path Cost를 가진 Root Port를 선택한다. Rapid-PVST+는 RSTP의 Proposal/Agreement와 Alternate Port를 사용해 빠르게 수렴하고, MSTP는 여러 VLAN을 소수의 MST Instance에 매핑하여 STP Instance 수를 줄인다.

---

## 1. STP가 필요한 이유

### Q1. L2 Loop는 왜 치명적인가?

👉 **핵심 답변:** Ethernet Frame에는 IP Packet의 TTL과 같은 Loop 제한 값이 없기 때문에, Loop가 발생하면 Frame이 계속 순환할 수 있습니다.

L2 Loop의 대표적인 증상:

- Broadcast Storm
- Unknown Unicast 증폭
- 동일 Frame의 중복 수신
- MAC Address Flapping
- Trunk 대역폭 고갈
- Switch CPU 사용률 증가
- 관리 접속 및 전체 서비스 장애

```properties
L3 Loop
→ TTL이 감소
→ TTL 0에서 Packet 폐기

L2 Loop
→ Ethernet Header에 TTL 없음
→ Loop가 제거될 때까지 Frame 순환 가능
```

STP는 물리적인 이중화 Link를 유지하면서 일부 Port를 논리적으로 차단합니다.

```properties
정상 상태
Primary Path = Forwarding
Backup Path  = Blocking/Discarding

Primary 장애
Backup Path  = Forwarding으로 전환
```

---

### Q2. PVST+는 무엇인가?

PVST+(Per-VLAN Spanning Tree Plus)는 Cisco의 VLAN별 STP 구현입니다.

```properties
VLAN 10 → STP Instance 10
VLAN 20 → STP Instance 20
VLAN 30 → STP Instance 30
```

장점:

- VLAN별 Root Bridge 지정 가능
- VLAN별 차단 Port 선택 가능
- VLAN 단위 L2 Load Balancing 가능

단점:

- VLAN이 많아지면 STP Instance와 BPDU 처리량 증가
- CPU와 메모리 사용량 증가
- 플랫폼별 지원 가능한 Instance 수 제한

> PVST+/Rapid-PVST+의 최대 Instance 수는 모든 장비에서 128개로 고정된 것이 아닙니다. Catalyst 모델과 IOS/IOS XE 버전에 따라 지원 한도가 다르므로 확인해야 합니다.

---

## 2. Root Bridge 선출

### Q3. Root Bridge는 어떻게 선출되는가?

가장 낮은 Bridge ID를 가진 Switch가 Root Bridge가 됩니다.

```properties
Bridge ID
= Bridge Priority + Extended System ID + MAC Address
```

선출 순서:

1. Bridge Priority가 가장 낮은 Switch
2. Priority가 같으면 MAC Address가 가장 낮은 Switch

기본 Priority는 일반적으로 32768입니다.

PVST+에서는 Extended System ID로 VLAN ID가 포함됩니다.

```properties
VLAN 10의 기본 Priority 표시값
= 32768 + 10
= 32778
```

Priority는 일반적으로 4096 단위로 설정합니다.

```cisco
spanning-tree vlan 10 priority 4096
```

---

### Q4. Root Bridge를 수동 지정해야 하는 이유는?

기본 Priority를 그대로 사용하면 MAC Address가 가장 낮은 Switch가 Root가 됩니다. 이 경우 Access 계층의 저성능 Switch가 Root가 될 수 있습니다.

운영 권장:

- Distribution/Core 계층에 Root 배치
- Primary Root와 Secondary Root를 명시적으로 지정
- VLAN별 Root를 분산하여 Load Balancing
- Root 위치와 실제 Traffic 경로 일치

```cisco
SW1(config)# spanning-tree vlan 10,20 root primary
SW2(config)# spanning-tree vlan 10,20 root secondary
```

다른 VLAN은 반대로 배치할 수 있습니다.

```cisco
SW2(config)# spanning-tree vlan 30,40 root primary
SW1(config)# spanning-tree vlan 30,40 root secondary
```

> `root primary`와 `root secondary`는 현재 토폴로지를 기준으로 Priority를 자동 조정하는 Macro 명령입니다. 장기적으로 고정된 설계가 필요하면 명시적인 Priority 사용도 고려합니다.

---

## 3. STP Port Role 선출

### Q5. Root Port는 어떻게 선택되는가?

각 Non-Root Switch는 Root Bridge까지 가장 우수한 경로 하나를 Root Port로 선택합니다.

대표적인 비교 순서:

```properties
1. 가장 낮은 Root Bridge ID
2. 가장 낮은 Root Path Cost
3. 가장 낮은 Sender Bridge ID
4. 가장 낮은 Sender Port ID
5. 필요한 경우 가장 낮은 Local Port ID
```

```properties
Root Bridge
→ Root Port가 없음

Non-Root Bridge
→ STP Instance마다 일반적으로 Root Port 하나
```

---

### Q6. Designated Port는 어떻게 선택되는가?

각 Ethernet Segment에서 Root 방향으로 가장 우수한 BPDU를 광고하는 Port가 Designated Port가 됩니다.

대표 비교 기준:

1. 가장 낮은 Root Bridge ID
2. 가장 낮은 Root Path Cost
3. 가장 낮은 Sender Bridge ID
4. 가장 낮은 Sender Port ID

Root Bridge의 동작 중인 STP Port는 일반적으로 Designated Port입니다.

---

### Q7. 나머지 Port는 어떤 역할을 하는가?

Classic 802.1D STP/PVST+에서는 Root Port나 Designated Port가 아닌 Port가 Blocking 상태가 됩니다.

RSTP에서는 다음 역할이 추가됩니다.

| 역할 | 의미 |
|---|---|
| Root Port | Root Bridge로 가는 최적 경로 |
| Designated Port | Segment에서 Forwarding을 담당하는 Port |
| Alternate Port | Root Port의 Backup 경로 |
| Backup Port | 같은 Segment의 Designated Port Backup |
| Disabled | STP에 참여하지 않는 Port |

> “Alternate Port”는 주로 RSTP 용어입니다. Classic PVST+에서는 일반적으로 Non-Designated Blocking Port로 표현합니다.

---

## 4. STP Cost와 Tie-Breaker

### Q8. STP Cost는 어떻게 계산되는가?

Root Path Cost는 Root Bridge까지 경로에 포함된 Port Cost를 누적한 값입니다.

#### Short Path Cost의 대표값

| Bandwidth | Cost |
|---:|---:|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

#### Long Path Cost의 대표값

| Bandwidth | Cost |
|---:|---:|
| 10 Mbps | 2,000,000 |
| 100 Mbps | 200,000 |
| 1 Gbps | 20,000 |
| 10 Gbps | 2,000 |
| 100 Gbps | 200 |
| 1 Tbps | 20 |

Long Path Cost 설정:

```cisco
spanning-tree pathcost method long
```

확인:

```cisco
show spanning-tree summary
```

> Long Cost는 MSTP만의 전용 값이 아닙니다. Cisco Switch에서 STP Path Cost 계산 방식을 `short` 또는 `long`으로 선택하는 개념이며, 고속 Link를 세밀하게 구분하려면 Long Cost가 유리합니다.

---

### Q9. Cost로 경로를 조정하려면 어디에서 설정하는가?

해당 Switch가 특정 Port를 Root Port로 덜 선호하게 하려면 그 Switch의 수신 Port Cost를 높입니다.

```cisco
interface GigabitEthernet0/1
 spanning-tree vlan 10 cost 50
```

```properties
낮은 Cost = 더 선호
높은 Cost = 덜 선호
```

변경 전후 확인:

```cisco
show spanning-tree vlan 10
```

---

### Q10. Port Priority는 언제 사용하는가?

동일한 Neighbor Switch의 병렬 Link처럼 Root Path Cost와 Sender Bridge ID가 같은 경우 Sender Port ID가 Tie-Breaker가 될 수 있습니다.

```cisco
interface GigabitEthernet0/1
 spanning-tree vlan 10 port-priority 64
```

```properties
낮은 Port Priority = 더 우선
높은 Port Priority = 덜 우선
```

Port Priority는 일반적으로 16 단위로 설정하며 기본값은 128입니다.

> 단순히 “차단시키려는 반대편 Port에서 무조건 설정한다”고 암기하기보다, BPDU의 Sender Port ID와 Local Port ID 중 어느 Tie-Breaker가 사용되는지 실제 토폴로지에서 확인해야 합니다.

---

## 5. ⏳ Classic STP 상태와 Timer

### Q11. 802.1D STP의 Port 상태는 무엇인가?

| 상태 | BPDU 처리 | MAC 학습 | Data 전달 |
|---|:---:|:---:|:---:|
| Disabled | X | X | X |
| Blocking | O | X | X |
| Listening | O | X | X |
| Learning | O | O | X |
| Forwarding | O | O | O |

전환 흐름:

```properties
Blocking
  ↓
Listening: Forward Delay, 기본 15초
  ↓
Learning: Forward Delay, 기본 15초
  ↓
Forwarding
```

대표 Timer:

| Timer | 기본값 | 역할 |
|---|---:|---|
| Hello Time | 2초 | Root BPDU 전송 주기 |
| Max Age | 20초 | BPDU 정보 유효 시간 |
| Forward Delay | 15초 | Listening/Learning 단계 시간 |

> 실제 수렴 시간은 장애 유형과 BPDU 수신 상태에 따라 달라집니다. Legacy STP에서 약 30초 또는 최악의 경우 Max Age를 포함해 약 50초가 걸릴 수 있지만, 모든 장애가 정확히 30초 또는 50초로 고정되는 것은 아닙니다.

---

### Q12. STP Timer를 직접 변경해도 되는가?

```cisco
spanning-tree vlan 10 hello-time 2
spanning-tree vlan 10 max-age 20
spanning-tree vlan 10 forward-time 15
```

Timer는 서로 연관되어 있으므로 임의로 줄이면 불안정한 수렴이나 Loop 위험이 생길 수 있습니다.

권장사항:

- 특별한 설계 근거 없이 기본값 유지
- 빠른 수렴이 필요하면 Rapid-PVST+ 또는 MSTP 사용
- 변경한다면 Root Bridge에서 일관되게 관리
- 변경 전 Diameter와 플랫폼 제한 검토

---

## 6. Rapid-PVST+와 RSTP

### Q13. RSTP가 빠른 이유는 무엇인가?

RSTP(IEEE 802.1w)는 단순히 Listening을 삭제한 것이 아니라 다음 기능을 사용합니다.

- Proposal/Agreement Handshake
- Alternate Port 사전 계산
- Edge Port
- Point-to-Point Link
- BPDU를 각 Switch가 직접 생성
- 장애 감지와 Port Role 재계산 개선

```properties
Proposal
→ 내가 이 Segment의 Designated Bridge로 동작하겠다는 제안

Agreement
→ 하위 Switch가 다른 경로를 동기화한 뒤 즉시 Forwarding 동의
```

Rapid Transition은 주로 다음 조건에서 동작합니다.

- Full-Duplex Point-to-Point Link
- 양쪽 장비가 RSTP/Rapid-PVST+ 지원
- Edge 또는 적절한 Proposal/Agreement 조건
- Shared Link가 아님

---

### Q14. RSTP Port 상태는 무엇인가?

| Classic STP | RSTP |
|---|---|
| Disabled | Discarding |
| Blocking | Discarding |
| Listening | Discarding |
| Learning | Learning |
| Forwarding | Forwarding |

```properties
Discarding = Data 전달 X, MAC 학습 X
Learning   = Data 전달 X, MAC 학습 O
Forwarding = Data 전달 O, MAC 학습 O
```

Rapid-PVST+ 설정:

```cisco
spanning-tree mode rapid-pvst
```

확인:

```cisco
show spanning-tree summary
show spanning-tree vlan 10
```

---

### Q15. RSTP인데 왜 느리게 수렴할 수 있는가?

원인 후보:

- 반대편이 Classic 802.1D STP
- Shared/Half-Duplex Link
- Port가 Point-to-Point로 동작하지 않음
- Proposal/Agreement가 완료되지 않음
- BPDU Filter 또는 단방향 Link
- STP Mode가 Switch 사이에서 불일치
- STP Boundary에서 Legacy 동작

RSTP는 Legacy STP Neighbor를 감지하면 해당 Port에서 호환 동작으로 전환할 수 있습니다.

---

## 7. MSTP

### Q16. MSTP가 필요한 이유는 무엇인가?

MSTP(IEEE 802.1s, 현재 802.1Q에 통합)는 여러 VLAN을 하나의 MST Instance에 매핑합니다.

```properties
VLAN 10,20 → MST Instance 1
VLAN 30,40 → MST Instance 2
나머지 VLAN → MST Instance 0
```

장점:

- STP Instance 수 감소
- CPU와 메모리 사용량 감소
- VLAN Group별 경로 제어
- RSTP 기반 빠른 수렴

---

### Q17. MST0, IST, CIST는 무엇인가?

| 용어 | 의미 |
|---|---|
| MST0 | MST Instance 0 |
| IST | Region 내부를 대표하는 Internal Spanning Tree |
| CST | MST Region 밖을 연결하는 Common Spanning Tree |
| CIST | CST와 IST를 결합한 전체 Spanning Tree |

별도 Instance에 매핑하지 않은 VLAN은 MST0에 속합니다.

---

### Q18. 같은 MST Region이 되기 위한 조건은 무엇인가?

다음 세 항목이 모두 일치해야 합니다.

1. Region Name
2. Revision Number
3. VLAN-to-Instance Mapping

```properties
Name 일치
AND Revision 일치
AND VLAN Mapping 일치
= 같은 MST Region
```

한 글자나 VLAN 하나라도 다르면 다른 MST Region으로 인식할 수 있습니다.

---

### Q19. MSTP 표준 설정은 어떻게 하는가?

모든 Region Member에 동일한 설정을 적용합니다.

```cisco
spanning-tree mode mst
!
spanning-tree mst configuration
 name CAMPUS
 revision 10
 instance 1 vlan 10,20
 instance 2 vlan 30,40
 exit
```

Root 지정:

```cisco
spanning-tree mst 1 root primary
spanning-tree mst 2 root secondary
```

다른 Switch에서 반대로 구성:

```cisco
spanning-tree mst 1 root secondary
spanning-tree mst 2 root primary
```

확인:

```cisco
show spanning-tree mst configuration
show spanning-tree mst
show spanning-tree mst 0
show spanning-tree mst 1
```

---

## 8. PortFast, UplinkFast, BackboneFast

### Q20. PortFast는 무엇인가?

PortFast는 단말 연결 Port를 Listening/Learning 대기 없이 즉시 Forwarding으로 전환합니다.

```cisco
interface GigabitEthernet0/10
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

전역 기본 설정:

```cisco
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

> PortFast는 STP를 완전히 끄는 기능이 아닙니다. Port는 여전히 BPDU를 처리할 수 있으며, BPDU Guard를 함께 사용하면 BPDU 수신 시 Err-disabled 상태가 됩니다.

---

### Q21. UplinkFast와 BackboneFast는 언제 사용하는가?

| 기능 | 목적 | 적용 |
|---|---|---|
| UplinkFast | 직접 Root Port 장애 시 Backup Port 빠른 전환 | Legacy PVST Access Switch |
| BackboneFast | 간접 Link 장애 시 Max Age 대기 단축 | Legacy PVST Domain 전체 |
| Rapid-PVST+/MSTP | RSTP 방식 빠른 수렴 | 현대 환경 권장 |

Legacy 설정:

```cisco
spanning-tree uplinkfast
spanning-tree backbonefast
```

주의사항:

- UplinkFast는 Root Bridge에 적용하지 않음
- BackboneFast는 STP Domain 전체에서 일관되게 적용
- UplinkFast는 Bridge Priority와 Port Cost를 내부적으로 조정할 수 있음
- Rapid-PVST+/MSTP에서는 기능이 내장되어 별도 설정이 일반적으로 필요하지 않음

---

## 9. STP 보호 기능

### Q22. 보호 기능별 차이는 무엇인가?

| 기능 | 권장 위치 | Trigger | 결과 |
|---|---|---|---|
| BPDU Guard | Edge/PortFast Port | BPDU 수신 | Err-disabled |
| BPDU Filter | 매우 제한적으로 사용 | BPDU 송수신 억제 | STP 가시성 제거 |
| Root Guard | 하위 Switch 방향 Designated Port | Superior BPDU 수신 | Root-inconsistent |
| Loop Guard | Root/Alternate 등 Non-Designated Port | BPDU 수신 중단 | Loop-inconsistent |
| UDLD | Fiber 등 단방향 장애 가능 Link | Unidirectional 감지 | Port 차단/Err-disable |

---

### Q23. BPDU Guard는 어떻게 설정하는가?

Interface별:

```cisco
interface GigabitEthernet0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

전역 PortFast Port 대상:

```cisco
spanning-tree portfast default
spanning-tree portfast bpduguard default
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

> 원인을 제거하지 않으면 자동 복구 후 다시 Err-disabled 상태가 됩니다.

---

### Q24. BPDU Filter는 왜 위험한가?

Interface별 BPDU Filter는 BPDU 송수신을 억제하여 STP가 이웃 Switch를 인식하지 못하게 만들 수 있습니다.

```cisco
interface GigabitEthernet0/10
 spanning-tree bpdufilter enable
```

이 설정은 잘못 사용하면 L2 Loop를 만들 수 있으므로 일반적인 단말 Port에서는 BPDU Guard가 더 안전합니다.

전역 PortFast BPDU Filter:

```cisco
spanning-tree portfast bpdufilter default
```

> 전역 PortFast BPDU Filter와 Interface별 `bpdufilter enable`은 세부 동작이 다를 수 있습니다. 플랫폼 문서를 확인하고, BPDU Filter는 명확한 설계 근거가 있을 때만 사용합니다.

---

### Q25. Root Guard와 Loop Guard는 어떻게 다른가?

#### Root Guard

```cisco
interface GigabitEthernet0/20
 spanning-tree guard root
```

- Superior BPDU 수신 시 Root-inconsistent
- Root Bridge 위치 보호
- 하위 Access Switch 방향 Designated Port에 적용
- Superior BPDU가 사라지면 자동 복구

#### Loop Guard

```cisco
interface GigabitEthernet0/1
 spanning-tree guard loop
```

- 수신하던 BPDU가 갑자기 사라지면 Loop-inconsistent
- Alternate/Root Port가 Designated Port로 잘못 전환되는 것을 방지
- 단방향 Link 및 BPDU 유실 보호
- 정상 BPDU가 다시 수신되면 자동 복구

> Root Guard와 Loop Guard는 목적과 적용 Port Role이 다르며, 일반적으로 같은 Port에 동시에 적용하지 않습니다.

---

## 10. 검증 명령어

### 기본 STP

```cisco
show spanning-tree
show spanning-tree vlan 10
show spanning-tree root
show spanning-tree summary
show spanning-tree detail
```

### Port

```cisco
show spanning-tree interface GigabitEthernet0/1 detail
show interfaces status
show interfaces status err-disabled
```

### MSTP

```cisco
show spanning-tree mst
show spanning-tree mst 0
show spanning-tree mst 1
show spanning-tree mst configuration
```

### 보호 기능

```cisco
show spanning-tree inconsistentports
show errdisable recovery
show logging
```

### Debug

```cisco
debug spanning-tree events
```

> Debug는 CPU와 Log 부하를 발생시킬 수 있으므로 운영망에서는 제한적으로 사용하고 종료 시 `undebug all`을 실행합니다.

---

## 11. 실무 체크리스트

- [ ] Root Bridge와 Secondary Root를 수동 지정했는가?
- [ ] Root 위치가 실제 Traffic 및 Gateway 위치와 일치하는가?
- [ ] VLAN별 Root 분산 설계가 의도한 경로와 일치하는가?
- [ ] Short/Long Path Cost 방식을 확인했는가?
- [ ] Cost 변경 전후 Root Port와 Blocking Port를 비교했는가?
- [ ] Rapid-PVST+ 또는 MSTP Mode가 모든 관련 Switch에서 일관적인가?
- [ ] MST Region Name, Revision, VLAN Mapping이 완전히 같은가?
- [ ] 단말 Port에 PortFast와 BPDU Guard를 적용했는가?
- [ ] Switch 간 Trunk에 PortFast를 잘못 적용하지 않았는가?
- [ ] BPDU Filter를 불필요하게 사용하지 않았는가?
- [ ] Root Guard를 실제 Root 방향 Port에 잘못 적용하지 않았는가?
- [ ] Loop Guard와 UDLD 적용 위치를 검토했는가?
- [ ] STP Timer를 근거 없이 변경하지 않았는가?
- [ ] STP Mode 변경 전 전체 토폴로지와 Boundary를 검토했는가?

---

> 📌 **핵심 요약**
> - STP는 L2 Loop를 차단하는 Control Plane Protocol이다.
> - 가장 낮은 Bridge ID의 Switch가 Root Bridge가 된다.
> - Non-Root Switch는 최저 Root Path Cost Port를 Root Port로 선택한다.
> - 각 Segment에는 하나의 Designated Port가 선택된다.
> - Classic STP는 Blocking→Listening→Learning→Forwarding으로 전환한다.
> - Rapid-PVST+는 Proposal/Agreement와 Alternate Port로 빠르게 수렴한다.
> - MSTP는 여러 VLAN을 하나의 MST Instance에 매핑한다.
> - MST Region은 Name, Revision, VLAN Mapping이 모두 일치해야 한다.
> - PortFast는 단말 Port에 적용하고 BPDU Guard와 함께 사용한다.
> - Root Guard는 Superior BPDU를, Loop Guard는 BPDU 유실을 방어한다.
> - BPDU Filter는 STP 가시성을 제거할 수 있어 매우 주의해야 한다.
> - 관련: 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence · 3-2. ⚡ Fast 계열 (PortFast, UplinkFast, BackboneFast) · 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard) · 3-4. 🚀 RSTP & MSTP (초대형망 STP) · 3-5. 🧩 STP·RSTP·MSTP 통합 정리 · 3-6. 🚑 STP·RSTP·MSTP 트러블슈팅 치트시트 · 3-7. ⚡ STP·RSTP·MSTP 퀵 레퍼런스
