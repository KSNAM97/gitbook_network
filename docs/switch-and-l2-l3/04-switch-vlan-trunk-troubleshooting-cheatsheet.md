# 스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #Switch #VLAN #Trunk #DTP #InterVLAN  
> **핵심 요약:** VLAN 장애는 **물리 상태 → VLAN 존재 여부 → Access VLAN → Trunk 상태 → Allowed/Native VLAN → MAC 학습 → Gateway와 L3 Routing** 순서로 확인한다. 같은 VLAN 통신은 L2 경로를, 서로 다른 VLAN 통신은 L2와 L3 경로를 모두 검증해야 한다.

---

## 1. 공통 진단 순서

```properties
[1] 포트가 물리적으로 Up인가?
[2] 포트가 올바른 Access/Trunk Mode인가?
[3] 필요한 VLAN이 실제로 생성되어 있는가?
[4] Access Port가 올바른 VLAN에 속하는가?
[5] 스위치 간 Trunk가 실제로 Trunking 상태인가?
[6] VLAN이 Trunk Allowed List에 포함되어 있는가?
[7] Native VLAN과 Encapsulation이 양쪽에서 일치하는가?
[8] MAC Address가 예상 포트에서 학습되는가?
[9] 단말 IP, Mask, Gateway가 올바른가?
[10] Router Sub-interface 또는 SVI가 Up인가?
[11] ACL, Port Security, STP가 차단하고 있지 않은가?
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 명령 | 조치 |
|---|---|---|---|
| 포트가 연결되지 않음 | Cable, Shutdown, Err-disabled | `show interfaces status` | Cable 및 포트 상태 확인 |
| 같은 VLAN인데 Ping 실패 | VLAN 오할당, VLAN 미생성 | `show vlan brief` | VLAN 생성 및 Access VLAN 수정 |
| VLAN 삭제 후 포트 통신 불가 | Access VLAN 설정은 남고 VLAN만 삭제 | `show interfaces switchport` | VLAN 복구 또는 포트 재할당 |
| 다른 Switch의 같은 VLAN과 통신 불가 | Trunk 또는 Allowed VLAN 문제 | `show interfaces trunk` | Trunk 및 Allowed List 수정 |
| Trunk가 형성되지 않음 | Mode/Encapsulation 불일치 | `show interfaces switchport` | 양쪽을 수동 Trunk로 고정 |
| 특정 VLAN만 Trunk 통과 불가 | Allowed VLAN 누락 | `show interfaces trunk` | `allowed vlan add` 사용 |
| Native VLAN 경고 발생 | Native VLAN 불일치 | `show interfaces trunk` | 양쪽 Native VLAN 통일 |
| MAC Address가 학습되지 않음 | 단말 무응답, VLAN/포트 오류 | `show mac address-table` | 단말 Traffic 생성 및 포트 확인 |
| MAC이 여러 포트 사이를 이동 | L2 Loop, 이중 연결 | Log와 MAC Table 확인 | Loop 제거 및 STP 검토 |
| VLAN 간 통신 실패 | Gateway/Sub-interface 오류 | `show ip interface brief` | Dot1Q와 Gateway 수정 |
| 한 방향만 통신 | ACL, Mask, 비대칭 경로 | ACL/Route/ARP 확인 | 정책과 주소 수정 |
| Port가 Err-disabled | BPDU Guard, Port Security 등 | `show interfaces status err-disabled` | 원인 제거 후 복구 |

---

## 3. 핵심 진단 명령어

### 물리 및 Port 상태

```cisco
show interfaces status
show interfaces GigabitEthernet0/1
show interfaces GigabitEthernet0/1 counters errors
show running-config interface GigabitEthernet0/1
```

### VLAN 및 Switchport

```cisco
show vlan brief
show vlan id 10
show interfaces GigabitEthernet0/1 switchport
show interfaces status
```

### MAC Address Table

```cisco
show mac address-table
show mac address-table dynamic
show mac address-table vlan 10
show mac address-table interface GigabitEthernet0/1
```

### Trunk 및 DTP

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show dtp interface GigabitEthernet0/24
show cdp neighbors detail
```

### Router-on-a-Stick

```cisco
show ip interface brief
show interfaces GigabitEthernet0/0.10
show ip route connected
show arp
show running-config interface GigabitEthernet0/0.10
```

### STP 및 Err-disabled

```cisco
show spanning-tree vlan 10
show spanning-tree interface GigabitEthernet0/24
show interfaces status err-disabled
show errdisable recovery
show logging
```

---

## 4. 대표 장애 시나리오

### 시나리오 1. 같은 VLAN인데 통신이 안 됨

#### 증상

VLAN 10에 연결된 두 PC 사이의 Ping이 실패합니다.

#### 확인

```cisco
show vlan brief
show interfaces status
show interfaces GigabitEthernet0/1 switchport
show interfaces GigabitEthernet0/2 switchport
show mac address-table vlan 10
```

#### 원인 후보

- 두 포트 중 하나가 다른 VLAN에 속함
- VLAN 10이 생성되지 않음
- 포트가 Shutdown 또는 Err-disabled
- 단말 IP Subnet Mask가 다름
- Port Security 또는 ACL이 차단
- 두 단말이 다른 Switch에 있는데 Trunk가 없음

#### 수정

```cisco
vlan 10
 name USERS
!
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 no shutdown
```

---

### 시나리오 2. VLAN 삭제 후 Access Port가 통신하지 못함

#### 기존 설정

```cisco
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
```

다음 명령으로 VLAN만 삭제:

```cisco
no vlan 10
```

#### 원인

VLAN 10은 삭제됐지만 Port의 `switchport access vlan 10` 설정은 남아 있을 수 있습니다. 포트는 존재하지 않는 VLAN을 참조하여 Inactive 상태가 됩니다.

#### 확인

```cisco
show vlan brief
show interfaces GigabitEthernet0/1 switchport
show interfaces status
```

#### 해결 방법 1: VLAN 복구

```cisco
vlan 10
 name USERS
```

#### 해결 방법 2: 다른 VLAN으로 재할당

```cisco
interface GigabitEthernet0/1
 switchport access vlan 20
```

---

### 시나리오 3. Trunk가 형성되지 않음

#### 확인

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show dtp interface GigabitEthernet0/24
```

#### 원인 후보

- 양쪽 모두 `dynamic auto`
- 한쪽이 `switchport mode access`
- Encapsulation 불일치
- 포트가 물리적으로 Down
- Routed Port로 설정되어 있음
- EtherChannel Member 설정 불일치

#### 수정

양쪽 Switch에서 수동 Trunk로 설정합니다.

```cisco
interface GigabitEthernet0/24
 switchport
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 no shutdown
```

> 고정 802.1Q 플랫폼에서는 `switchport trunk encapsulation dot1q` 명령을 지원하지 않을 수 있습니다.

---

### 시나리오 4. `trunk`와 `dynamic auto`인데 Trunk가 안 된다고 판단함

`trunk`와 `dynamic auto` 조합은 일반적으로 Trunk를 형성할 수 있습니다.

```properties
trunk + dynamic auto = Trunk
dynamic desirable + dynamic auto = Trunk
dynamic auto + dynamic auto = Trunk 형성 안 됨
```

문제가 지속된다면 다음을 확인합니다.

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show dtp interface GigabitEthernet0/24
```

운영망에서는 협상 결과에 의존하지 않고 양쪽을 수동 Trunk로 고정하는 것이 안전합니다.

---

### 시나리오 5. Trunk는 정상인데 특정 VLAN만 통신하지 못함

#### 확인

```cisco
show interfaces trunk
show vlan brief
show spanning-tree vlan 20
```

`show interfaces trunk`에서 다음 세 항목을 구분합니다.

```properties
Vlans allowed on trunk
Vlans allowed and active in management domain
Vlans in spanning tree forwarding state and not pruned
```

#### 원인 후보

- VLAN이 Allowed List에 없음
- 해당 VLAN이 Switch에 생성되지 않음
- STP가 해당 VLAN을 Blocking
- VTP Pruning 또는 유사 기능으로 제거
- 반대쪽 Switch의 Allowed List가 다름

#### 수정

```cisco
interface GigabitEthernet0/24
 switchport trunk allowed vlan add 20
```

필요한 VLAN도 생성합니다.

```cisco
vlan 20
 name SERVERS
```

---

### 시나리오 6. Allowed VLAN 변경 후 여러 VLAN이 갑자기 끊김

#### 위험 설정

```cisco
interface GigabitEthernet0/24
 switchport trunk allowed vlan 40
```

#### 원인

위 명령은 기존 Allowed VLAN 목록에 VLAN 40을 추가하는 것이 아니라, 목록 전체를 VLAN 40으로 교체할 수 있습니다.

#### 올바른 추가

```cisco
interface GigabitEthernet0/24
 switchport trunk allowed vlan add 40
```

#### 변경 전후 확인

```cisco
show interfaces trunk
show running-config interface GigabitEthernet0/24
```

---

### 시나리오 7. Native VLAN Mismatch 경고 발생

#### 증상

CDP 또는 Log에 Native VLAN Mismatch 경고가 표시됩니다.

#### 확인

```cisco
show interfaces trunk
show cdp neighbors detail
show logging
```

#### 원인

```properties
SW1 Trunk Native VLAN = 999
SW2 Trunk Native VLAN = 1
```

Untagged Frame이 양쪽에서 서로 다른 VLAN으로 분류될 수 있습니다.

#### 수정

양쪽 Trunk의 Native VLAN을 동일하게 설정합니다.

```cisco
interface GigabitEthernet0/24
 switchport trunk native vlan 999
```

VLAN도 생성합니다.

```cisco
vlan 999
 name NATIVE-BLACKHOLE
```

---

### 시나리오 8. MAC Address가 학습되지 않음

#### 확인

```cisco
show mac address-table interface GigabitEthernet0/1
show interfaces GigabitEthernet0/1 counters
show interfaces GigabitEthernet0/1 switchport
```

#### 원인 후보

- 단말이 Frame을 전송하지 않음
- 포트가 Down
- Access VLAN 오류
- Port Security 위반
- 단말 NIC 또는 Cable 문제
- MAC Entry Aging 후 새 Traffic이 없음

#### 조치

단말에서 Gateway Ping 또는 ARP Traffic을 발생시킨 뒤 다시 확인합니다.

```cisco
show mac address-table dynamic
show port-security interface GigabitEthernet0/1
```

> MAC Address Table을 무조건 삭제하기 전에 물리 상태와 VLAN 설정을 먼저 확인합니다.

---

### 시나리오 9. 같은 MAC이 여러 포트 사이에서 이동함

#### 증상

Log에 MAC Flapping 또는 MAC Move 메시지가 반복됩니다.

#### 원인 후보

- L2 Loop
- 잘못된 이중 연결
- NIC Teaming 또는 가상화 환경의 설정 오류
- Wireless Roaming
- MLAG/VSS/Stack 구성 문제
- Access/Trunk 오구성

#### 확인

```cisco
show logging
show mac address-table address <MAC>
show spanning-tree vlan <VLAN-ID>
show interfaces trunk
```

#### 조치

MAC이 이동하는 두 포트를 추적하고 Loop 여부를 우선 확인합니다. 원인 분석 없이 MAC Table만 반복 삭제하면 문제가 재발합니다.

---

### 시나리오 10. Router-on-a-Stick 구성 후 VLAN 간 통신 불가

#### 확인

```cisco
show interfaces trunk
show ip interface brief
show ip route connected
show arp
show running-config interface GigabitEthernet0/0.10
```

#### 원인 후보

- Switch-Router 연결 포트가 Access Mode
- VLAN이 Allowed List에서 누락
- Sub-interface의 Dot1Q VLAN ID 불일치
- 물리 Interface가 Shutdown
- 단말 Default Gateway 오류
- IP Address 또는 Subnet Mask 중복
- Native VLAN 설정 불일치
- Router ACL이 차단

#### 정상 예시

Router:

```cisco
interface GigabitEthernet0/0
 no shutdown
!
interface GigabitEthernet0/0.10
 encapsulation dot1q 10
 ip address 192.168.10.254 255.255.255.0
```

Switch:

```cisco
interface GigabitEthernet0/24
 switchport mode trunk
 switchport trunk allowed vlan add 10
```

PC:

```properties
IP Address      = 192.168.10.10
Subnet Mask     = 255.255.255.0
Default Gateway = 192.168.10.254
```

---

### 시나리오 11. Sub-interface가 Down 상태임

Sub-interface 상태는 물리 Interface에 의존합니다.

```properties
Physical Interface Down
→ 모든 Sub-interface Down

Physical Interface Administratively Down
→ 모든 Sub-interface Administratively Down
```

#### 확인

```cisco
show ip interface brief
show interfaces GigabitEthernet0/0
```

#### 수정

```cisco
interface GigabitEthernet0/0
 no shutdown
```

Switch 연결 포트와 Cable 상태도 확인합니다.

---

### 시나리오 12. Port가 Err-disabled 상태임

#### 확인

```cisco
show interfaces status err-disabled
show logging
show errdisable recovery
show port-security interface GigabitEthernet0/1
```

#### 주요 원인

- BPDU Guard
- Port Security Violation
- Link Flap
- UDLD
- EtherChannel Misconfiguration
- Storm Control

#### 안전한 복구

먼저 원인을 제거한 후 Port를 복구합니다.

```cisco
interface GigabitEthernet0/1
 shutdown
 no shutdown
```

> 원인을 제거하지 않고 `shutdown`과 `no shutdown`만 수행하면 다시 Err-disabled 상태가 될 수 있습니다.

---

## 5. 안전한 변경 절차

```properties
[1] 현재 VLAN 및 Trunk 상태 저장
[2] 양쪽 장비 설정 비교
[3] 변경 대상 Interface 확인
[4] VLAN을 먼저 생성
[5] Access 또는 Trunk Mode 설정
[6] Allowed VLAN은 add/remove로 변경
[7] Native VLAN 양쪽 일치 확인
[8] MAC 학습과 STP 상태 확인
[9] Gateway 및 Routing 확인
[10] Ping과 실제 서비스로 검증
```

변경 전 확인:

```cisco
show vlan brief
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show spanning-tree vlan 10
show mac address-table vlan 10
show running-config interface GigabitEthernet0/24
```

변경 후 확인:

```cisco
show interfaces trunk
show vlan brief
show mac address-table dynamic
show ip interface brief
ping <DESTINATION>
traceroute <DESTINATION>
```

---

> 📌 **핵심 요약**
> - 같은 VLAN 장애는 Port 상태, VLAN 할당, MAC 학습 순서로 확인한다.
> - 다른 Switch까지 연결되면 Trunk와 Allowed VLAN을 추가 확인한다.
> - `dynamic auto + dynamic auto`는 Trunk가 형성되지 않는다.
> - `trunk + dynamic auto`는 일반적으로 Trunk가 형성된다.
> - `allowed vlan`은 목록을 덮어쓸 수 있으므로 추가 시 `add`를 사용한다.
> - VLAN을 삭제해도 Port의 Access VLAN 설정이 남을 수 있다.
> - Native VLAN은 Trunk 양쪽에서 동일해야 한다.
> - Router-on-a-Stick 장애는 Trunk, Dot1Q VLAN ID, Gateway를 함께 확인한다.
> - Err-disabled는 원인을 제거한 뒤 Port를 복구한다.
> - 관련: 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 1-3. 🧩 스위치 기본 & VLAN·Trunk 통합 정리 · 1-4. 🚑 스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트 · 1-5. ⚡ 스위치 기본 & VLAN·Trunk 퀵 레퍼런스
