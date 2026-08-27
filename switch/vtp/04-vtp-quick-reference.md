# VTP 명령어 퀵 레퍼런스

> **Tag:** #Network #QuickReference #VTP #VLAN #Cisco #VTPv3  
> **핵심 요약:** VTP의 Domain·Password·Version·Mode 설정과 검증 명령어를 빠르게 확인하기 위한 암기 카드. VTPv1/v2에서는 Configuration Revision을 반드시 확인하고, VTPv3에서는 Primary Server 여부를 추가로 확인한다.

---

## 1. VTP 역할

| 요구사항 | VTP Mode 또는 기능 |
|---|---|
| VLAN 생성 및 광고 | Server |
| VLAN Database 수신 전용 | Client |
| 로컬 VLAN 독립 관리 및 광고 미반영 | Transparent |
| VTP 완전 비참여 | Off |
| Extended VLAN 동기화 | VTPv3 |
| VTPv3 VLAN 변경 | Primary Server |
| 불필요한 VLAN Flooding 억제 | VTP Pruning |

---

## 2. Mode 비교

| Mode | VLAN 변경 | 수신 DB 반영 | 광고 중계 | 로컬 VLAN 관리 |
|---|:---:|:---:|:---:|:---:|
| Server | O | O | O | VTP DB 기준 |
| Client | X | O | O | X |
| Transparent | O | X | 버전별 차이 | O |
| Off | O | X | 일반적으로 X | O |

---

## 3. Version 비교

| 기능 | VTPv1 | VTPv2 | VTPv3 |
|---|---|---|---|
| Standard VLAN | O | O | O |
| Extended VLAN 동기화 | X | X | O |
| Primary Server | X | X | O |
| MST DB 지원 | X | X | O |
| Revision 보호 | 낮음 | 낮음 | 향상 |
| Transparent 중계 | 제한적 | 개선 | 개선 |

---

## 4. 기본 설정

VTP Domain:

```cisco
vtp domain GIT_NET
```

VTP Password:

```cisco
vtp password cisco1234
```

VTP Version:

```cisco
vtp version 1
vtp version 2
vtp version 3
```

Server:

```cisco
vtp mode server
```

Client:

```cisco
vtp mode client
```

Transparent:

```cisco
vtp mode transparent
```

Off Mode 지원 플랫폼:

```cisco
vtp mode off
```

---

## 5. VTPv2 Server

```cisco
configure terminal
!
vtp version 2
vtp domain GIT_NET
vtp password cisco1234
vtp mode server
!
vlan 10
 name USERS
!
vlan 20
 name SERVERS
```

---

## 6. VTPv2 Client

```cisco
configure terminal
!
vtp version 2
vtp domain GIT_NET
vtp password cisco1234
vtp mode client
```

Client에서 VLAN 생성 시:

```properties
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

---

## 7. Transparent

```cisco
configure terminal
!
vtp version 2
vtp domain GIT_NET
vtp password cisco1234
vtp mode transparent
!
vlan 30
 name LOCAL_ONLY
```

```properties
Server VLAN DB 수신 → 로컬에 반영 안 함
Local VLAN 생성     → 다른 VTP 장비에 동기화 안 함
VTP 광고 중계       → Version/Domain/플랫폼에 따라 차이
```

---

## 8. Off Mode

```cisco
configure terminal
!
vtp mode off
```

```properties
VTP VLAN DB 반영 안 함
VTP 광고 생성 안 함
VTP 광고 일반적으로 중계 안 함
Local VLAN 독립 관리
```

> Off Mode 지원 여부는 Catalyst 모델과 IOS/IOS XE 버전에서 확인한다.

---

## 9. 🆕 VTPv3 Primary Server

기본 설정:

```cisco
configure terminal
!
vtp version 3
vtp domain GIT_NET
vtp password cisco1234
vtp mode server
end
```

Primary 지정:

```cisco
vtp primary vlan
```

VLAN 생성:

```cisco
configure terminal
!
vlan 10
 name USERS
!
vlan 3001
 name EXTENDED_VLAN
```

확인:

```cisco
show vtp status
show vtp devices
```

```properties
VTPv3 Server Mode
+ Primary Server 역할
= VLAN Database 변경 가능
```

---

## 10. Trunk 설정

```cisco
interface GigabitEthernet0/1
 description VTP-TRUNK
 switchport mode trunk
```

Allowed VLAN:

```cisco
interface GigabitEthernet0/1
 switchport trunk allowed vlan 10,20,30
```

기존 목록에 추가:

```cisco
interface GigabitEthernet0/1
 switchport trunk allowed vlan add 40
```

확인:

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/1 switchport
```

> VTP로 VLAN이 생성되어도 Trunk Allowed VLAN 목록은 자동으로 변경되지 않는다.

---

## 11. Extended VLAN

### VTPv1/v2

```cisco
vtp mode transparent
!
vlan 3001
 name EXTENDED_VLAN
```

```properties
VTPv1/v2 Server/Client
→ Extended VLAN 동기화 불가

VTPv1/v2 Transparent
→ Extended VLAN 로컬 생성 가능
```

### VTPv3

```cisco
vtp version 3
vtp mode server
end
!
vtp primary vlan
```

```cisco
configure terminal
vlan 3001
 name EXTENDED_VLAN
```

---

## 12. VTP Pruning

활성화:

```cisco
vtp pruning
```

확인:

```cisco
show vtp status
show interfaces trunk
show interfaces pruning
```

비활성화:

```cisco
no vtp pruning
```

```properties
Allowed VLAN = 정적 VLAN 허용 목록
VTP Pruning  = 불필요한 Flooding 동적 억제
STP          = L2 Loop 방지
```

---

## 13. Revision 안전 확인

확인:

```cisco
show vtp status
```

확인 항목:

```properties
VTP Version
VTP Operating Mode
VTP Domain Name
Configuration Revision
Number of existing VLANs
Updater Identity
Primary ID 또는 Primary Server 상태
```

VTPv1/v2 Revision 초기화에 일반적으로 사용하는 전환:

```cisco
vtp mode transparent
```

임시 Domain 변경 방식:

```cisco
vtp domain TEMP_RESET
vtp domain GIT_NET
```

> 장비와 Version에 따라 동작이 다를 수 있으므로 명령 후 반드시 `show vtp status`에서 Revision 0 여부를 확인한다.

---

## 14. 신규 Switch 투입 순서

```properties
운영 Trunk 연결 금지
    ↓
show vtp status
    ↓
show vlan brief
    ↓
VTP Version/Mode/Domain 확인
    ↓
VTPv1/v2 Revision 초기화 및 실제 값 확인
    ↓
Password 설정
    ↓
기존 VLAN DB와 비교
    ↓
Maintenance Window에 Trunk 연결
    ↓
전체 VLAN DB 재검증
```

신규 Switch 사전 확인:

```cisco
show vtp status
show vtp password
show vlan brief
show interfaces trunk
```

VTP 미사용 환경:

```cisco
vtp mode transparent
```

또는:

```cisco
vtp mode off
```

---

## 15. 검증 명령어

VTP 상태:

```cisco
show vtp status
```

Password:

```cisco
show vtp password
```

Counter:

```cisco
show vtp counters
```

VTPv3 장비 정보:

```cisco
show vtp devices
```

VLAN:

```cisco
show vlan brief
show vlan id <VLAN-ID>
```

Trunk:

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
```

STP:

```cisco
show spanning-tree vlan <VLAN-ID>
```

Log:

```cisco
show logging
```

---

## 16. 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| 높은 Revision | 올바른 DB가 아니라 더 최신으로 판단될 수 있는 DB |
| VTP Client | VLAN 변경 불가, 수신 DB 반영 |
| VTP Transparent | 수신 DB 미반영, 로컬 VLAN 관리 |
| VTP Off | VTP 비참여, 일반적으로 광고 중계도 안 함 |
| VTPv1/v2 Extended VLAN | Transparent에서 로컬 관리 |
| VTPv3 Extended VLAN | VTP 동기화 가능 |
| VTPv3 Server | Primary가 아니면 VLAN 변경 제한 |
| VTP Domain 일치 | 동기화 조건 중 하나일 뿐 |
| 관리 IP 없음 | VTP 동작 가능 |
| VLAN 동기화 성공 | Access Port 설정 완료를 의미하지 않음 |
| VLAN 동기화 성공 | Allowed VLAN 자동 추가를 의미하지 않음 |
| Transparent Switch | Server VLAN이 자동 생성되지 않음 |
| VTP Pruning | VLAN DB에서 VLAN을 삭제하는 기능이 아님 |
| Revision Reset 명령 입력 | 실제 Reset 성공을 보장하지 않음 |
| 신규 Switch 연결 | Revision과 VLAN DB 사전 확인 필수 |

---

## 17. 빠른 장애 진단 순서

```properties
Trunk Up 확인
    ↓
VTP Version 확인
    ↓
Domain 확인
    ↓
Password 확인
    ↓
Mode 확인
    ↓
Revision 확인
    ↓
VLAN DB 확인
    ↓
Allowed VLAN 확인
    ↓
STP 확인
    ↓
Access Port 및 Gateway 확인
```

---

## 18. 안전한 변경 원칙

변경 전:

```cisco
show vtp status
show vtp password
show vlan brief
show interfaces trunk
```

변경:

```properties
VTPv1/v2 = 지정된 Server에서 VLAN 변경
VTPv3    = Primary Server에서 VLAN 변경
Transparent = 해당 Switch에서 로컬 변경
```

변경 후:

```cisco
show vtp status
show vlan brief
show interfaces trunk
show spanning-tree vlan <VLAN-ID>
```

---

> 📌 **핵심 요약**
> - Domain: `vtp domain <NAME>`
> - Password: `vtp password <PASSWORD>`
> - Version: `vtp version 1|2|3`
> - Server: `vtp mode server`
> - Client: `vtp mode client`
> - Transparent: `vtp mode transparent`
> - Off: `vtp mode off`
> - VTPv3 Primary: `vtp primary vlan`
> - Pruning: `vtp pruning`
> - 상태 확인: `show vtp status`
> - VLAN 확인: `show vlan brief`
> - Trunk 확인: `show interfaces trunk`
> - 신규 Switch는 Revision과 VLAN DB 검증 후 연결한다.
> - 관련: 2-1. 🔄 VTP 개념 & Mode & 동기화 · 2-2. 🧩 VTP 통합 정리 · 2-3. 🚑 VTP 트러블슈팅 치트시트 · 2-4. ⚡ VTP 명령어 퀵 레퍼런스 · 1-3. 🧩 스위치 기본 & VLAN·Trunk 통합 정리
