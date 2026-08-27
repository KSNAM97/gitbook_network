# 🧩 VTP 통합 정리

> **Tag:** #Network #Switch #VTP #VLAN #Cisco #VTPv1 #VTPv2 #VTPv3  
> **핵심 요약:** VTP는 Cisco Switch 사이에서 VLAN Database를 동기화하는 Cisco 전용 L2 프로토콜이다. VTPv1/v2는 Domain·Password·Version과 Configuration Revision을 기준으로 VLAN 정보를 동기화하므로, 높은 Revision을 가진 장비를 잘못 연결하면 VLAN Database가 덮어써질 수 있다. VTPv3는 Primary Server 구조와 향상된 보호 기능을 제공하지만, 운영망에서는 여전히 Transparent/Off Mode 또는 명시적인 VLAN 관리 방식을 많이 사용한다.

---

## 1. 🎯 핵심 기술 개념

### Q1. VTP는 무엇인가?

👉 **핵심 답변:** VTP(VLAN Trunking Protocol)는 Cisco Switch 사이에서 VLAN 생성·수정·삭제 정보를 광고하여 VLAN Database를 동기화하는 Cisco 전용 L2 프로토콜입니다.

```properties
VTP Server에서 VLAN 변경
        ↓
VTP Advertisement 전송
        ↓
Trunk Link를 통해 전달
        ↓
같은 VTP Domain의 Client/Server가 VLAN DB 반영
```

VTP가 동기화하는 정보의 예:

- VLAN ID
- VLAN Name
- VLAN Type
- VLAN 상태
- Configuration Revision

VTP가 자동으로 동기화하지 않는 대표 항목:

- Access Port의 VLAN 할당
- Trunk Allowed VLAN 목록
- Native VLAN
- SVI IP Address
- STP 설정
- ACL 및 보안 정책

> VTP로 VLAN 10이 생성되더라도 각 Access Port에 `switchport access vlan 10`을 자동으로 설정하지는 않습니다.

---

### Q2. VTP는 왜 위험하다고 하는가?

👉 **핵심 답변:** 특히 VTPv1/v2에서는 같은 Domain과 인증 조건을 만족하는 장비의 Configuration Revision이 기존 장비보다 높으면 그 VLAN Database를 더 최신 정보로 판단할 수 있기 때문입니다.

```properties
운영망 Revision = 15
신규 Switch Revision = 50
Domain/Password/Version 일치
        ↓
신규 Switch의 VLAN DB가 최신 정보로 판단될 가능성
        ↓
운영 VLAN 변경 또는 삭제
        ↓
Access Port Inactive 및 대규모 통신 장애
```

주요 위험 요소:

- 재사용 Switch에 기존 VTP 정보가 남아 있음
- 높은 Configuration Revision이 남아 있음
- VLAN이 거의 없는 상태로 높은 Revision을 보유
- 운영 Domain과 Password가 우연히 일치
- 장비를 사전 검증 없이 Trunk에 연결
- 여러 VTP Server에서 무분별하게 VLAN 변경

> 신규 Switch는 운영 Trunk에 연결하기 전에 격리 상태에서 VTP Mode, Domain, Version, Revision, VLAN Database를 확인해야 합니다.

---

### Q3. Configuration Revision은 무엇인가?

Configuration Revision은 VLAN Database 변경 버전을 나타내는 값입니다.

VTPv1/v2 Server에서 다음 작업을 수행하면 일반적으로 Revision이 증가합니다.

- VLAN 생성
- VLAN 이름 변경
- VLAN 삭제
- VLAN 속성 변경

```properties
Revision이 높다
= VTP가 더 최신 VLAN Database로 판단할 수 있음
```

Configuration Revision은 장비의 정상성이나 신뢰성을 의미하지 않습니다.

```properties
높은 Revision ≠ 올바른 VLAN Database
높은 Revision = 변경 횟수가 더 많은 것으로 판단되는 Database
```

---

### Q4. VTP 동작에 필요한 조건은 무엇인가?

VTP 동기화를 위해 일반적으로 다음 조건을 확인합니다.

1. Switch 사이에 동작 중인 Trunk가 존재
2. VTP Domain 일치
3. VTP Password 일치
4. VTP Version 호환
5. 수신 장비가 Server 또는 Client Mode
6. VTP Advertisement가 중간 장비에서 차단되지 않음

```cisco
show interfaces trunk
show vtp status
show vtp password
```

> VTP는 L2 프로토콜이므로 Switch에 관리 IP가 없어도 동작할 수 있습니다. 관리 IP Ping 성공 여부는 관리망 연결 검증에는 유용하지만 VTP 동작의 필수 조건은 아닙니다.

---

### Q5. VTP Mode는 어떻게 구분하는가?

| Mode | VLAN 생성·수정·삭제 | 수신 DB 반영 | VTP 광고 전달 | 로컬 VLAN 독립 운영 |
|---|:---:|:---:|:---:|:---:|
| Server | O | O | O | X |
| Client | X | O | O | X |
| Transparent | O | X | 버전에 따라 중계 | O |
| Off | O | X | 일반적으로 X | O |

#### Server

- VLAN 생성·수정·삭제 가능
- VLAN 변경 정보를 광고
- 다른 Server 또는 Client의 적절한 광고를 반영
- VTPv1/v2에서는 여러 Server가 VLAN을 변경할 수 있음
- VTPv3에서는 Primary Server만 VLAN Database 변경 가능

#### Client

- 로컬에서 VLAN 생성·수정·삭제 불가
- Server의 VLAN Database를 수신하고 반영
- 수신한 VTP Advertisement를 다른 Trunk로 전달 가능
- VTPv1/v2에서 높은 Revision 위험과 무관하지 않음

#### Transparent

- VTP Advertisement를 로컬 VLAN Database에 반영하지 않음
- VLAN을 로컬에서 직접 관리
- VTPv1/v2의 중계 동작은 Version 및 Domain 조건에 따라 차이
- Configuration Revision은 일반적으로 0으로 표시되거나 동기화에 참여하지 않음

#### Off

- VTP 동기화에 참여하지 않음
- VTP Advertisement를 일반적으로 전달하지 않음
- 로컬에서 VLAN을 독립적으로 관리
- 지원 여부는 플랫폼과 VTP Version에 따라 확인 필요

---

### Q6. Transparent와 Off는 무엇이 다른가?

👉 **핵심 답변:** Transparent는 로컬 DB에는 반영하지 않으면서 VTP Advertisement를 중계할 수 있지만, Off는 일반적으로 수신 광고를 폐기하고 중계하지 않습니다.

```properties
Transparent
수신 → 로컬 DB 반영 안 함 → 조건에 따라 다른 Trunk로 중계

Off
수신 → 로컬 DB 반영 안 함 → 일반적으로 중계 안 함
```

VTP를 사용하지 않으면서 중간 Switch가 VTP Domain 사이의 전달 경로가 될 필요도 없다면 Off Mode가 더 명확할 수 있습니다.

---

### Q7. VTP Version별 차이는 무엇인가?

| 항목 | VTPv1 | VTPv2 | VTPv3 |
|---|---|---|---|
| Standard VLAN 동기화 | O | O | O |
| Extended VLAN 동기화 | X | X | O |
| Primary Server | X | X | O |
| VLAN 변경 보호 | 낮음 | 낮음 | 향상 |
| MST Database 지원 | X | X | O |
| Transparent 중계 개선 | 제한적 | 향상 | 향상 |
| 비밀번호 보호 | 기본 인증 | 기본 인증 | 향상된 인증 지원 |

#### VTPv1

- 기본 VLAN 동기화 기능 제공
- Transparent 중계 조건이 비교적 엄격
- Token Ring 관련 제약 존재

#### VTPv2

- VTPv1과 전체적인 동작은 유사
- Transparent Mode에서 광고 중계 동작 개선
- Token Ring VLAN 지원 개선
- Revision 기반 위험이 근본적으로 제거된 것은 아님

#### VTPv3

- VLAN 1~4094 범위 지원
- Extended VLAN 동기화 지원
- Primary/Secondary Server 구조
- VLAN Database 변경 권한 강화
- MST Database 동기화 지원
- VTPv1/v2보다 안전성이 향상됨

> 세부 지원 범위는 Catalyst 모델과 IOS/IOS XE 버전에 따라 다를 수 있습니다.

---

### Q8. Extended VLAN은 어느 Mode에서 생성할 수 있는가?

#### VTPv1/v2

Extended VLAN인 1006~4094는 VTP Server/Client Database로 동기화할 수 없습니다. 일반적으로 Transparent Mode에서 로컬로 생성합니다.

```cisco
vtp mode transparent
!
vlan 3001
 name EXTENDED_VLAN
```

#### VTPv3

VTPv3는 Extended VLAN을 VTP Database에 포함하여 동기화할 수 있습니다.

```cisco
vtp version 3
vtp mode server
```

VLAN을 변경하려면 해당 Switch가 VLAN Database의 Primary Server여야 합니다.

```cisco
vtp primary vlan
```

> `vtp primary vlan`은 일반적으로 Privileged EXEC Mode에서 실행하며, 재부팅 후 Primary 역할을 다시 지정해야 하는 플랫폼이 있습니다.

---

### Q9. VTPv3 Primary Server는 무엇인가?

VTPv3에서는 Server Mode 장비가 모두 VLAN Database를 변경할 수 있는 것이 아닙니다.

```properties
VTPv3 Primary Server
= VLAN 생성·수정·삭제 가능

VTPv3 Secondary Server
= VLAN Database 보관 및 전달
= 일반적으로 직접 VLAN 변경 불가
```

설정 예:

```cisco
configure terminal
vtp version 3
vtp domain GIT_NET
vtp password cisco1234
vtp mode server
end
!
vtp primary vlan
```

확인:

```cisco
show vtp status
show vtp devices
```

> 이미 다른 Primary Server가 존재하는 환경에서 `force` 옵션을 사용하면 충돌이나 예상하지 못한 Database 변경이 발생할 수 있으므로 신중해야 합니다.

---

### Q10. VTP Domain은 자동으로 전파되는가?

VTP Domain은 모든 Switch에 명시적으로 동일하게 설정하는 것이 안전합니다.

```cisco
vtp domain GIT_NET
```

일부 VTPv1/v2 장비는 Domain이 비어 있는 상태에서 수신한 광고를 통해 Domain을 학습할 수 있지만, 이를 운영 절차로 의존하면 안 됩니다.

```properties
권장 방식
= 운영 연결 전에 모든 Switch에 Domain을 명시적으로 설정
```

---

### Q11. VTP Password는 어떻게 동작하는가?

VTP Password는 같은 Domain의 장비가 VTP Update를 인증하는 데 사용됩니다.

```cisco
vtp password cisco1234
```

Password가 다르면 VLAN Database가 동기화되지 않습니다.

```cisco
show vtp password
show vtp status
```

주의사항:

- 모든 참여 Switch에 개별 설정
- 평문 노출 가능성 및 Configuration 보안 고려
- 장비별 Password 저장 방식 확인
- VTPv3에서는 향상된 인증 옵션 지원 여부 확인

---

### Q12. VTP Pruning은 무엇인가?

VTP Pruning은 특정 VLAN의 활성 Port가 없는 하위 Switch 방향으로 불필요한 Flooding Traffic을 보내지 않도록 Trunk 전송 범위를 동적으로 줄이는 기능입니다.

```properties
VTP Pruning 대상
= Broadcast
= Unknown Unicast
= 일부 Multicast

VTP Pruning 비대상
= 일반적으로 VLAN 1
= 일부 예약 VLAN
```

Server에서 활성화:

```cisco
vtp pruning
```

확인:

```cisco
show vtp status
show interfaces trunk
show interfaces pruning
```

> VTP Pruning과 `switchport trunk allowed vlan`은 다릅니다. Allowed VLAN은 관리자가 정적으로 허용 VLAN을 지정하고, VTP Pruning은 불필요한 Flooding 전송을 동적으로 줄입니다.

---

## 2. 🛠️ VTPv1/v2 표준 설정

### 2-1. Server

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

### 2-2. Client

```cisco
configure terminal
!
vtp version 2
vtp domain GIT_NET
vtp password cisco1234
vtp mode client
```

Client에서는 VLAN 변경이 제한됩니다.

```cisco
vlan 30
```

예상 메시지:

```properties
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

---

### 2-3. Transparent

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

Transparent Switch에서 생성한 VLAN은 해당 Switch의 로컬 VLAN Database에만 존재합니다.

---

### 2-4. Trunk

양쪽 Switch에서 Trunk를 구성합니다.

```cisco
interface GigabitEthernet0/1
 description TRUNK-TO-SW2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

> VTP로 VLAN Database가 동기화되더라도 VLAN Traffic을 전달하려면 해당 VLAN이 Trunk Allowed VLAN 목록에도 포함되어야 합니다.

---

## 3. 🛠️ VTPv3 표준 설정

### 3-1. Server 및 Primary 지정

```cisco
configure terminal
!
vtp version 3
vtp domain GIT_NET
vtp password cisco1234
vtp mode server
end
!
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

---

### 3-2. Client

```cisco
configure terminal
!
vtp version 3
vtp domain GIT_NET
vtp password cisco1234
vtp mode client
```

---

### 3-3. Transparent 또는 Off

Transparent:

```cisco
vtp mode transparent
```

Off Mode 지원 플랫폼:

```cisco
vtp mode off
```

---

## 4. 🛡️ 신규 Switch 안전 투입 절차

```properties
[1] 운영 Trunk에 연결하지 않은 격리 상태 유지
[2] show vtp status로 Version/Mode/Domain/Revision 확인
[3] show vlan brief로 기존 VLAN Database 확인
[4] VTP를 사용하지 않으면 Transparent 또는 Off 설정
[5] VTPv1/v2 참여 장비는 Revision 0 여부 확인
[6] 정확한 Domain/Password/Version 설정
[7] 운영 VLAN 목록과 비교
[8] Maintenance Window에 Trunk 연결
[9] 모든 Switch의 VLAN 및 Revision 재확인
```

격리 상태에서 확인:

```cisco
show vtp status
show vtp password
show vlan brief
show interfaces trunk
```

Revision 초기화에 일반적으로 사용하는 방법:

```cisco
vtp mode transparent
```

또는 임시 Domain 변경 후 다시 설정하는 방법이 사용될 수 있습니다.

```cisco
vtp domain TEMP_RESET
vtp domain GIT_NET
```

> Revision 초기화 동작은 플랫폼과 VTP Version에 따라 다를 수 있습니다. 명령 실행 후 반드시 `show vtp status`에서 실제 Revision 값을 확인해야 합니다.

---

## 5. 🔍 검증 명령어

### VTP

```cisco
show vtp status
show vtp password
show vtp counters
show vtp devices
```

### VLAN

```cisco
show vlan brief
show vlan id 10
show vlan id 3001
```

### Trunk

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/1 switchport
show running-config interface GigabitEthernet0/1
```

### Log

```cisco
show logging
show vtp counters
```

---

## 6. ⚠️ 실무 체크리스트

- [ ] 운영망이 사용하는 VTP Version을 확인했는가?
- [ ] 신규 Switch를 Trunk에 연결하기 전에 격리했는가?
- [ ] 신규 Switch의 Configuration Revision을 확인했는가?
- [ ] VTP Domain의 대소문자와 철자가 정확한가?
- [ ] 모든 참여 Switch의 Password가 일치하는가?
- [ ] VTP Mode가 의도한 Server/Client/Transparent/Off인가?
- [ ] VTPv3 Server가 Primary인지 확인했는가?
- [ ] 필요한 VLAN이 실제 VLAN Database에 존재하는가?
- [ ] 필요한 VLAN이 Trunk Allowed List에도 포함되어 있는가?
- [ ] Transparent와 Off의 중계 차이를 고려했는가?
- [ ] Extended VLAN 사용 시 VTP Version과 Mode를 확인했는가?
- [ ] VTP Pruning과 Allowed VLAN을 혼동하지 않았는가?
- [ ] VLAN 변경 전 Configuration과 VLAN DB를 백업했는가?
- [ ] VLAN 삭제가 Access Port를 Inactive 상태로 만들지 검토했는가?

---

> 📌 **핵심 요약**
> - VTP는 Cisco Switch 사이의 VLAN Database 동기화 프로토콜이다.
> - VTP는 Access Port 설정이나 Trunk Allowed VLAN을 동기화하지 않는다.
> - VTPv1/v2는 Configuration Revision 관리가 핵심이다.
> - 높은 Revision은 올바른 Database가 아니라 더 최신으로 판단되는 Database다.
> - Server는 VLAN을 변경하고 Client는 수신한 VLAN DB를 반영한다.
> - Transparent는 로컬 VLAN을 독립적으로 관리한다.
> - Off는 일반적으로 VTP 광고를 반영하거나 중계하지 않는다.
> - VTPv1/v2의 Extended VLAN은 Transparent Mode에서 로컬 관리한다.
> - VTPv3는 Extended VLAN과 Primary Server를 지원한다.
> - VTP는 관리 IP 없이도 동작할 수 있는 L2 프로토콜이다.
> - 신규 Switch는 격리 상태에서 Revision과 VLAN DB를 검증한 뒤 투입한다.
> - 관련: 2-1. 🔄 VTP 개념 & Mode & 동기화 · 2-2. 🧩 VTP 통합 정리 · 2-3. 🚑 VTP 트러블슈팅 치트시트 · 2-4. ⚡ VTP 명령어 퀵 레퍼런스 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN
