# VTP 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #VTP #VLAN #Trunk #Cisco  
> **핵심 요약:** VTP 장애는 **Trunk 상태 → VTP Version → Domain → Password → Mode → Revision → VLAN Database → Allowed VLAN** 순서로 확인한다. VLAN이 동기화됐는데도 통신하지 못한다면 VTP가 아니라 Access Port, Trunk Allowed VLAN, STP 또는 L3 Gateway 문제일 수 있다.

---

## 1. 공통 진단 순서

```properties
[1] Switch 사이 Trunk가 실제로 Up인가?
[2] 양쪽 VTP Version이 호환되는가?
[3] VTP Domain Name이 일치하는가?
[4] VTP Password가 일치하는가?
[5] 수신 Switch가 Client 또는 Server Mode인가?
[6] Configuration Revision과 Updater 정보를 확인했는가?
[7] VLAN이 VLAN Database에 생성됐는가?
[8] VLAN이 Trunk Allowed List에 포함됐는가?
[9] STP가 해당 VLAN을 Forwarding하고 있는가?
[10] Access Port와 Gateway 설정이 올바른가?
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| VLAN이 동기화되지 않음 | Trunk, Domain, Password, Version 불일치 | `show vtp status` | 불일치 항목 수정 |
| Client에서 VLAN 생성 거부 | Client Mode의 정상 동작 | `show vtp status` | Server 또는 Primary에서 변경 |
| Transparent에서 Server VLAN이 안 보임 | 정상 동작 | VTP Mode 확인 | 필요한 VLAN을 로컬 생성 |
| 신규 Switch 연결 후 VLAN 변경 | 높은 Revision | VTP Status와 Log | 격리 후 DB 복구 |
| VTPv3 Server에서 VLAN 생성 거부 | Primary가 아님 | `show vtp status` | `vtp primary vlan` 실행 |
| VLAN은 보이지만 통신 안 됨 | Allowed VLAN, Access Port 오류 | Trunk/VLAN 확인 | Port와 Allowed List 수정 |
| Extended VLAN 생성 거부 | VTPv1/v2 Server Mode | Version/Mode 확인 | Transparent 또는 VTPv3 사용 |
| VTP 광고가 중간 Switch를 통과하지 않음 | Transparent/Off 동작 차이 | 중간 장비 Mode 확인 | Version/Mode 재설계 |
| Revision이 예상과 다름 | 다른 Server에서 변경 | VTP Counter와 Updater 확인 | 변경 주체 추적 |
| VTP Pruning 후 특정 Traffic 문제 | Pruning 또는 STP 오판 | Trunk/Pruning 확인 | VLAN 사용 상태와 Pruning 검토 |

---

## 3. 핵심 진단 명령어

### VTP 상태

```cisco
show vtp status
show vtp password
show vtp counters
show vtp devices
```

### VLAN

```cisco
show vlan brief
show vlan id <VLAN-ID>
```

### Trunk

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
show running-config interface <INTERFACE>
```

### STP와 Port

```cisco
show spanning-tree vlan <VLAN-ID>
show interfaces status
show mac address-table vlan <VLAN-ID>
```

### Log

```cisco
show logging
show vtp counters
```

---

## 4. 대표 장애 시나리오

### 시나리오 1. Server에서 만든 VLAN이 Client에 나타나지 않음

#### 확인 순서

```cisco
show interfaces trunk
show vtp status
show vtp password
show vlan brief
```

#### 원인 후보

1. Trunk가 형성되지 않음
2. VTP Domain 불일치
3. VTP Password 불일치
4. VTP Version 비호환
5. Client가 Transparent 또는 Off Mode
6. 중간 Switch가 VTP Advertisement를 전달하지 않음
7. VLAN 변경을 수행한 장비가 실제 Server가 아님

#### 정상 예시

Server:

```cisco
vtp version 2
vtp domain GIT_NET
vtp password cisco1234
vtp mode server
```

Client:

```cisco
vtp version 2
vtp domain GIT_NET
vtp password cisco1234
vtp mode client
```

Trunk:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
```

---

### 시나리오 2. VLAN은 동기화됐지만 실제 통신이 안 됨

#### 핵심 판단

VTP는 VLAN Database만 동기화합니다.

```properties
VTP 동기화 성공
≠ Access Port 자동 할당
≠ Trunk Allowed VLAN 자동 추가
≠ Gateway 자동 설정
```

#### 확인

```cisco
show vlan brief
show interfaces trunk
show interfaces GigabitEthernet0/10 switchport
show spanning-tree vlan 20
show mac address-table vlan 20
```

#### 원인 후보

- Access Port가 다른 VLAN에 속함
- Trunk Allowed List에서 VLAN 누락
- STP가 VLAN 경로를 Blocking
- 단말 IP/Mask/Gateway 오류
- SVI 또는 Router Sub-interface가 Down

#### 수정

```cisco
interface GigabitEthernet0/10
 switchport mode access
 switchport access vlan 20
!
interface GigabitEthernet0/1
 switchport trunk allowed vlan add 20
```

---

### 시나리오 3. 신규 Switch 연결 후 전체 VLAN이 변경되거나 사라짐

#### 원인

신규 또는 재사용 Switch가 다음 조건을 만족했을 가능성이 있습니다.

```properties
운영 Domain과 일치
Password와 Version 호환
운영망보다 높은 Configuration Revision
잘못된 VLAN Database 보유
```

#### 즉시 조치

1. 문제 Switch의 Trunk를 분리
2. 추가 VLAN 변경 중단
3. 영향받은 Switch의 VTP 상태와 VLAN DB 저장
4. 정상 VLAN Database를 확인
5. 정상 Server에서 VLAN Database 복구
6. 모든 Access Port와 Trunk 상태 확인

```cisco
show vtp status
show vlan brief
show interfaces trunk
show logging
```

#### 예방 절차

운영망 연결 전 격리 상태에서:

```cisco
show vtp status
show vlan brief
```

VTP를 사용하지 않는다면:

```cisco
vtp mode transparent
```

또는 지원 플랫폼:

```cisco
vtp mode off
```

VTPv1/v2 Domain에 투입한다면 Revision이 실제로 0인지 확인합니다.

```cisco
show vtp status
```

---

### 시나리오 4. Client에서 VLAN 생성·삭제가 거부됨

#### 증상

```properties
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

#### 원인

정상 동작입니다. Client는 VLAN Database를 수신하지만 로컬에서 VLAN을 변경할 수 없습니다.

#### 해결 방법 1: Server에서 변경

```cisco
vlan 30
 name VOICE
```

#### 해결 방법 2: 독립 관리가 필요하면 Transparent

```cisco
vtp mode transparent
!
vlan 30
 name LOCAL_VOICE
```

> Transparent 전환은 VLAN 관리 설계를 변경하는 작업이므로 운영 Domain 영향을 먼저 검토합니다.

---

### 시나리오 5. Transparent Switch에 Server VLAN이 생성되지 않음

#### 원인

Transparent Mode는 수신한 VTP VLAN Database를 로컬 DB에 반영하지 않습니다.

```properties
VTP Advertisement 수신
→ 로컬 VLAN DB 반영 안 함
→ 조건에 따라 다른 Trunk로 중계
```

#### 해결

Transparent Switch에도 해당 VLAN이 필요하면 로컬에서 생성합니다.

```cisco
vlan 10
 name USERS
```

> 중계 여부는 VTP Version, Domain 및 플랫폼 구현에 따라 확인해야 합니다.

---

### 시나리오 6. VTPv3 Server인데 VLAN을 생성할 수 없음

#### 증상

```properties
VTP VLAN configuration not allowed when device is not
the primary server for VLAN database.
```

#### 원인

VTPv3 Server가 Secondary 역할이며 VLAN Database의 Primary Server가 아닙니다.

#### 확인

```cisco
show vtp status
show vtp devices
```

#### 조치

```cisco
vtp primary vlan
```

그 후 VLAN을 생성합니다.

```cisco
configure terminal
vlan 3001
 name EXTENDED_VLAN
```

> 다른 Primary가 존재하는지 확인하지 않고 강제로 Primary를 변경하면 Database 충돌이 발생할 수 있습니다.

---

### 시나리오 7. Extended VLAN을 생성할 수 없음

#### 증상

VLAN 3001 생성이 거부됩니다.

#### 원인 후보

- VTPv1/v2 Server Mode
- VTPv1/v2 Client Mode
- 플랫폼이 Extended VLAN을 지원하지 않음
- VTPv3이지만 Primary Server가 아님

#### VTPv1/v2 해결

```cisco
vtp mode transparent
!
vlan 3001
 name EXTENDED_VLAN
```

#### VTPv3 해결

```cisco
vtp version 3
vtp mode server
end
!
vtp primary vlan
```

---

### 시나리오 8. VTP Domain이 같은데 동기화되지 않음

Domain만 같다고 VTP가 동작하는 것은 아닙니다.

#### 추가 확인

```cisco
show vtp status
show vtp password
show interfaces trunk
show vtp counters
```

확인 항목:

- VTP Version
- Password
- Trunk 상태
- VTP Mode
- 중간 Switch Mode
- Advertisement 수신 Counter
- VLAN Database Revision

---

### 시나리오 9. VTP Password를 맞췄는데 동기화되지 않음

#### 원인 후보

- Password 오타
- Domain Name 불일치
- VTP Version 비호환
- Trunk가 아님
- 중간 장비가 Off Mode
- 변경 장비가 Primary Server가 아님
- Password 변경 후 광고가 아직 발생하지 않음

#### 확인

```cisco
show vtp password
show vtp status
show vtp counters
show interfaces trunk
```

필요하면 Server에서 테스트 VLAN을 생성하기 전에 운영 영향과 변경 절차를 검토합니다.

---

### 시나리오 10. Revision을 초기화했는데 값이 0이 아님

#### 원인 후보

- 전환 직후 다른 Advertisement를 수신함
- VTP Version별 동작 차이
- Domain 변경 절차가 플랫폼에서 다르게 동작
- 다른 Server가 즉시 VLAN Database를 동기화함
- 격리하지 않은 상태에서 초기화 시도

#### 안전한 조치

1. 모든 Trunk에서 격리
2. 현재 VLAN DB 백업
3. Transparent 또는 Off로 전환
4. 임시 Domain 설정 여부 검토
5. `show vtp status`로 실제 Revision 확인
6. 원하는 Domain/Mode를 설정한 뒤 연결

```cisco
show vtp status
show vlan brief
```

> 명령을 입력했다는 사실보다 실제 출력에서 Revision과 VLAN DB가 의도대로 되었는지 확인하는 것이 중요합니다.

---

### 시나리오 11. 관리 IP Ping이 안 되어 VTP도 실패했다고 판단함

VTP는 L2 프로토콜이므로 관리 IP가 없어도 동작할 수 있습니다.

```properties
관리 IP Ping 실패
≠ VTP 실패 확정

VTP VLAN 동기화
= L2 Trunk와 VTP 설정으로 판단
```

#### VTP 확인

```cisco
show vtp status
show vtp counters
show vlan brief
show interfaces trunk
```

#### 관리망 확인

```cisco
show ip interface brief
show interfaces vlan 1
show ip default-gateway
```

두 문제를 분리해서 진단합니다.

---

### 시나리오 12. VTP Pruning 이후 특정 VLAN Traffic 문제

#### 확인

```cisco
show vtp status
show interfaces trunk
show interfaces pruning
show spanning-tree vlan <VLAN-ID>
```

#### 원인 후보

- 해당 VLAN의 활성 Access Port 정보가 없음
- Trunk Allowed List에서도 VLAN이 제거됨
- STP Blocking을 Pruning 문제로 오해
- VLAN 자체가 존재하지 않음

```properties
Allowed VLAN
= 정적 허용 정책

VTP Pruning
= 불필요한 Flooding 동적 억제

STP Blocking
= Loop 방지를 위한 경로 차단
```

세 기능을 구분해서 확인합니다.

---

## 5. 안전한 변경 및 복구 절차

```properties
[1] show vtp status 출력 저장
[2] show vlan brief 출력 저장
[3] Trunk와 Allowed VLAN 상태 저장
[4] 정상 VLAN Database 기준 장비 선정
[5] 신규 Switch 또는 문제 Switch 격리
[6] Version/Domain/Password/Mode 수정
[7] Revision과 VLAN DB 재확인
[8] Maintenance Window에 연결
[9] 전체 Switch의 VLAN DB 비교
[10] Access Port와 실제 서비스 검증
```

변경 전:

```cisco
show vtp status
show vtp password
show vtp counters
show vlan brief
show interfaces trunk
```

변경 후:

```cisco
show vtp status
show vlan brief
show interfaces trunk
show spanning-tree vlan <VLAN-ID>
show mac address-table vlan <VLAN-ID>
```

---

> 📌 **핵심 요약**
> - VTP 장애는 Trunk부터 확인한다.
> - Domain뿐 아니라 Version과 Password도 확인한다.
> - Transparent는 수신 VLAN DB를 로컬에 반영하지 않는다.
> - Client에서 VLAN 변경이 거부되는 것은 정상이다.
> - VTPv3 Server는 Primary가 아니면 VLAN을 변경할 수 없다.
> - VLAN이 동기화돼도 Access Port와 Allowed VLAN은 별도 설정이다.
> - 관리 IP Ping은 VTP의 필수 조건이 아니다.
> - 신규 Switch는 운영 Trunk 연결 전에 격리해서 Revision을 확인한다.
> - 높은 Revision 사고가 발생하면 문제 장비를 먼저 격리한다.
> - 관련: 2-1. 🔄 VTP 개념 & Mode & 동기화 · 2-2. 🧩 VTP 통합 정리 · 2-3. 🚑 VTP 트러블슈팅 치트시트 · 2-4. ⚡ VTP 명령어 퀵 레퍼런스 · 1-5. ⚡ 스위치 기본 & VLAN·Trunk 퀵 레퍼런스
