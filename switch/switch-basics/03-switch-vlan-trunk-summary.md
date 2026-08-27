# 🧩 스위치 기본 & VLAN·Trunk 통합 정리

> **Tag:** #Network #Switch #Layer2 #VLAN #Trunk #DTP #Dot1Q #InterVLAN  
> **핵심 요약:** 스위치는 출발지 MAC을 학습하고 목적지 MAC을 기준으로 프레임을 전달한다. VLAN은 하나의 물리적 스위치를 여러 Broadcast Domain으로 논리적으로 분리하며, Trunk는 하나의 링크에서 여러 VLAN을 802.1Q Tag로 구분하여 전송한다. 서로 다른 VLAN 간 통신에는 Router-on-a-Stick 또는 L3 Switch의 Routing 기능이 필요하다.

---

## 1. 🎯 핵심 기술 개념

### Q1. Hub, Switch, Router는 무엇이 다른가?

👉 **핵심 답변:** Hub는 전기 신호를 반복하고, Switch는 MAC 주소를 기준으로 프레임을 전달하며, Router는 IP 주소를 기준으로 서로 다른 네트워크 간 패킷을 전달합니다.

| 장비 | 계층 | 판단 기준 | Collision Domain | Broadcast Domain |
|---|---:|---|---|---|
| Hub | L1 | 전기 신호 | 전체 포트가 공유 | 전체가 공유 |
| Switch | L2 | MAC 주소 | 포트별 분리 | VLAN별 분리 |
| Router | L3 | IP 주소 | 인터페이스별 분리 | 인터페이스별 분리 |
| L3 Switch | L2/L3 | MAC + IP | 포트별 분리 | VLAN/SVI별 분리 |

> 현대 Router와 L3 Switch는 플랫폼에 따라 ASIC 또는 하드웨어 기반 Forwarding을 수행하므로, Router를 단순히 “소프트웨어 기반 장비”라고 구분하는 것은 정확하지 않습니다.

---

### Q2. 스위치의 핵심 동작은 무엇인가?

👉 **핵심 답변:** 스위치는 다음 순서로 MAC Address Table을 학습하고 프레임을 전달합니다.

```properties
Learning
  출발지 Source MAC을 수신 포트와 함께 학습
        ↓
Forwarding
  목적지 MAC을 알면 해당 포트로만 전달
        ↓
Filtering
  목적지가 수신 포트와 같으면 다른 포트로 전달하지 않음
        ↓
Flooding
  목적지 MAC을 모르면 같은 VLAN의 다른 포트로 전파
        ↓
Aging
  일정 시간 사용되지 않은 동적 MAC Entry 삭제
```

스위치는 **출발지 MAC을 학습**하고 **목적지 MAC으로 전달 여부를 결정**합니다.

```properties
Source MAC      = MAC Address Table 학습에 사용
Destination MAC = Forwarding/Flooding 결정에 사용
```

---

### Q3. 스위치는 언제 Flooding하는가?

다음 프레임은 같은 VLAN 내부에서 Flooding될 수 있습니다.

- Broadcast
- Unknown Unicast
- 일부 Multicast
- MAC Address Table에 목적지 정보가 없는 프레임

```properties
Unknown Unicast
= 목적지 MAC이 MAC Address Table에 없는 Unicast Frame
```

Flooding 범위는 전체 스위치가 아니라 **해당 VLAN의 Broadcast Domain**입니다.

> Multicast는 항상 무조건 Flooding되는 것은 아닙니다. IGMP Snooping과 같은 기능이 동작하면 필요한 수신 포트로만 전달할 수 있습니다.

---

### Q4. MAC Address Table Aging이 필요한 이유는?

동적으로 학습된 MAC Entry는 영구적으로 유지되지 않습니다. Cisco Catalyst의 일반적인 기본 Aging Time은 300초이지만 플랫폼과 설정에 따라 달라질 수 있습니다.

```cisco
show mac address-table
show mac address-table dynamic
show mac address-table aging-time
```

Aging이 필요한 이유:

- 단말 이동에 대응
- 오래된 포트 정보를 제거
- MAC Table 자원 회수
- 잘못된 포트로 계속 전달되는 문제 방지

---

### Q5. VLAN이 필요한 이유는?

👉 **핵심 답변:** 스위치의 포트별 Collision Domain 분리만으로는 Broadcast 범위를 나눌 수 없기 때문에 VLAN을 사용합니다.

```properties
VLAN 10 = 하나의 Broadcast Domain
VLAN 20 = 별도의 Broadcast Domain
VLAN 30 = 별도의 Broadcast Domain
```

VLAN의 주요 목적:

- Broadcast Domain 분리
- 부서·서비스별 논리적 분리
- 보안 정책 적용 단위 제공
- 장애 및 Broadcast 영향 범위 축소
- 물리적 위치와 무관한 논리적 그룹 구성

> VLAN은 보안 경계를 만드는 중요한 요소이지만 VLAN만으로 완전한 보안이 제공되는 것은 아닙니다. VLAN 간 ACL, 방화벽, Port Security 등의 추가 정책이 필요합니다.

---

### Q6. Access Port와 Trunk Port는 무엇이 다른가?

| 구분 | Access Port | Trunk Port |
|---|---|---|
| 주요 연결 대상 | PC, Server, Printer | Switch, Router, AP, Hypervisor |
| 일반적인 VLAN 수 | 하나 | 여러 개 |
| 단말 방향 Frame | 일반적으로 Untagged | VLAN별 Tagged |
| 주요 명령 | `switchport mode access` | `switchport mode trunk` |
| VLAN 지정 | `switchport access vlan` | `switchport trunk allowed vlan` |

Access Port에서 수신한 Untagged Frame은 해당 포트에 설정된 Access VLAN 소속으로 처리됩니다.

---

### Q7. Trunk가 필요한 이유는?

Access Link는 일반적으로 하나의 VLAN만 운반합니다. 여러 VLAN을 스위치 사이에서 전달하기 위해 VLAN마다 물리 링크를 별도로 사용하면 포트와 케이블이 낭비됩니다.

```properties
Access Link
  하나의 링크 → 일반적으로 하나의 VLAN

Trunk Link
  하나의 링크 → 여러 VLAN
```

Trunk는 IEEE 802.1Q Tag를 사용해 각 Frame의 VLAN 소속을 구분합니다.

---

### Q8. 802.1Q Tag에는 무엇이 들어가는가?

802.1Q는 Ethernet Frame에 4Byte Tag를 삽입합니다.

주요 필드:

| 필드 | 역할 |
|---|---|
| TPID | 802.1Q Frame 식별 |
| PCP | 802.1p QoS 우선순위 |
| DEI | 혼잡 시 폐기 가능성 표시 |
| VLAN ID | VLAN 식별 |

VLAN ID 필드는 12bit입니다.

```properties
표현 가능한 값 = 0~4095
일반적으로 사용하는 VLAN ID = 1~4094
0과 4095 = 예약된 값
```

---

### Q9. Native VLAN은 무엇인가?

👉 **핵심 답변:** 일반적인 802.1Q Trunk에서 Native VLAN Frame은 기본적으로 Tag 없이 전송됩니다.

```cisco
interface GigabitEthernet0/1
 switchport trunk native vlan 999
```

양쪽 Trunk의 Native VLAN이 다르면 Untagged Frame을 서로 다른 VLAN으로 처리할 수 있습니다.

```properties
SW1 Native VLAN 999
SW2 Native VLAN 1
→ Native VLAN Mismatch
```

운영 권장사항:

- Trunk 양쪽 Native VLAN 일치
- 사용자 VLAN을 Native VLAN으로 사용하지 않기
- 가능하면 사용하지 않는 전용 VLAN 사용
- 플랫폼 정책에 따라 Native VLAN Tagging 검토

> ISL은 Cisco의 Legacy Encapsulation이며 802.1Q의 Native VLAN과 동일한 Untagged 동작을 사용하지 않습니다. 현대 환경에서는 일반적으로 802.1Q를 사용합니다.

---

### Q10. DTP는 무엇이며 왜 수동 설정을 권장하는가?

DTP(Dynamic Trunking Protocol)는 Cisco 장비 간 Trunk Mode를 협상하는 프로토콜입니다.

| Mode | 동작 |
|---|---|
| `access` | Access로 고정 |
| `trunk` | Trunk로 고정 |
| `dynamic desirable` | 능동적으로 Trunk 협상 |
| `dynamic auto` | 상대가 Trunk를 요청하면 Trunk 형성 |

대표적인 결과:

| 한쪽 | 반대쪽 | 결과 |
|---|---|---|
| trunk | trunk | Trunk |
| trunk | dynamic auto | Trunk |
| trunk | dynamic desirable | Trunk |
| desirable | auto | Trunk |
| desirable | desirable | Trunk |
| auto | auto | Access 상태 |
| access | trunk | Mode 불일치 가능 |

운영망에서는 예측 가능성과 보안을 위해 수동으로 고정합니다.

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
 switchport nonegotiate
```

> `switchport nonegotiate` 지원 여부와 적용 순서는 플랫폼에 따라 확인해야 합니다.

---

## 2. 🛠️ VLAN 표준 설정

### 2-1. VLAN 생성

```cisco
configure terminal
!
vlan 10
 name USERS
!
vlan 20
 name SERVERS
!
vlan 30
 name VOICE
```

다중 VLAN 생성은 플랫폼에서 VLAN List 문법을 지원할 때 사용할 수 있습니다.

```cisco
vlan 10,20,30
```

범위 생성:

```cisco
vlan 100-110
```

> 다중 및 범위 VLAN 명령의 지원 여부는 IOS/IOS XE와 플랫폼에서 확인합니다. 운영 환경에서는 VLAN별 이름을 명확히 부여하는 방식이 안전합니다.

---

### 2-2. VLAN 삭제

```cisco
no vlan 10
```

VLAN을 삭제해도 포트의 다음 설정은 남아 있을 수 있습니다.

```cisco
switchport access vlan 10
```

이 경우 포트는 존재하지 않는 VLAN을 참조하여 통신할 수 없는 Inactive 상태가 될 수 있습니다.

---

### 2-3. Access Port 설정

```cisco
interface GigabitEthernet0/1
 description USER-PC
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
```

단말 연결 포트에 필요하면 BPDU Guard도 함께 적용합니다.

```cisco
interface GigabitEthernet0/1
 spanning-tree bpduguard enable
```

> PortFast와 BPDU Guard는 단말 연결 포트에 적용하며, 다른 Switch가 연결된 포트에는 토폴로지를 검토하지 않고 적용하면 안 됩니다.

---

### 2-4. Interface Range

연속 포트:

```cisco
interface range GigabitEthernet0/1 - 8
 switchport mode access
 switchport access vlan 10
```

비연속 포트:

```cisco
interface range GigabitEthernet0/13, GigabitEthernet0/15, GigabitEthernet0/17
 switchport mode access
 switchport access vlan 20
```

혼합 범위:

```cisco
interface range GigabitEthernet0/9 - 12, GigabitEthernet0/21 - 24
 switchport mode access
 switchport access vlan 30
```

> Interface Range 문법과 공백 허용 방식은 플랫폼에 따라 차이가 있을 수 있습니다.

---

## 3. 🚚 Trunk 표준 설정

### 3-1. 스위치 간 Trunk

양쪽 스위치에 동일하게 적용합니다.

```cisco
interface GigabitEthernet0/24
 description TRUNK-TO-SW2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999
 switchport nonegotiate
```

일부 고정 802.1Q 플랫폼에는 다음 명령이 없습니다.

```cisco
switchport trunk encapsulation dot1q
```

이 경우 바로 Trunk Mode를 설정합니다.

```cisco
interface GigabitEthernet0/24
 switchport mode trunk
```

---

### 3-2. Allowed VLAN 관리

현재 목록을 지정한 값으로 교체:

```cisco
switchport trunk allowed vlan 10,20,30
```

기존 목록에 추가:

```cisco
switchport trunk allowed vlan add 40
```

기존 목록에서 제거:

```cisco
switchport trunk allowed vlan remove 20
```

특정 VLAN을 제외:

```cisco
switchport trunk allowed vlan except 100
```

기본 상태로 복원:

```cisco
no switchport trunk allowed vlan
```

> `switchport trunk allowed vlan 10`은 기존 Allowed VLAN 목록을 VLAN 10 하나로 덮어쓸 수 있습니다. 기존 목록에 추가할 때는 반드시 `add`를 사용합니다.

---

## 4. 🌐 Router-on-a-Stick

### 4-1. 동작 구조

```properties
PC VLAN 10
    ↓
Access Port VLAN 10
    ↓
Switch Trunk
    ↓
Router Sub-interface Fa0/0.10
    ↓
Routing
    ↓
Router Sub-interface Fa0/0.20
    ↓
Switch Trunk
    ↓
Access Port VLAN 20
```

서로 다른 VLAN 간 통신은 L3 Routing이 필요합니다.

---

### 4-2. Router 설정

```cisco
interface GigabitEthernet0/0
 no ip address
 no shutdown
!
interface GigabitEthernet0/0.10
 encapsulation dot1q 10
 ip address 192.168.10.254 255.255.255.0
!
interface GigabitEthernet0/0.20
 encapsulation dot1q 20
 ip address 192.168.20.254 255.255.255.0
!
interface GigabitEthernet0/0.30
 encapsulation dot1q 30
 ip address 192.168.30.254 255.255.255.0
```

Native VLAN Sub-interface가 필요하다면 양쪽 설정을 일치시킵니다.

```cisco
interface GigabitEthernet0/0.999
 encapsulation dot1q 999 native
 ip address 192.168.99.254 255.255.255.0
```

---

### 4-3. Switch 설정

```cisco
vlan 10
 name USERS
!
vlan 20
 name SERVERS
!
vlan 30
 name VOICE
!
interface GigabitEthernet0/24
 description TRUNK-TO-R1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

각 PC의 Default Gateway는 해당 VLAN의 Router Sub-interface IP로 설정합니다.

```properties
VLAN 10 Gateway = 192.168.10.254
VLAN 20 Gateway = 192.168.20.254
VLAN 30 Gateway = 192.168.30.254
```

---

## 5. 🔍 검증 명령어

### VLAN

```cisco
show vlan brief
show vlan id 10
show interfaces status
show interfaces GigabitEthernet0/1 switchport
```

### MAC Address Table

```cisco
show mac address-table
show mac address-table dynamic
show mac address-table vlan 10
show mac address-table interface GigabitEthernet0/1
show mac address-table aging-time
```

### Trunk와 DTP

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show dtp interface GigabitEthernet0/24
show interfaces GigabitEthernet0/24 status
```

### Router-on-a-Stick

```cisco
show ip interface brief
show interfaces GigabitEthernet0/0.10
show ip route connected
show arp
ping 192.168.20.10 source 192.168.10.254
```

---

## 6. ⚠️ 실무 체크리스트

- [ ] 필요한 VLAN이 모든 관련 Switch에 생성되어 있는가?
- [ ] Access Port의 VLAN 번호가 올바른가?
- [ ] 단말 연결 포트가 `switchport mode access`로 고정되어 있는가?
- [ ] Switch 간 연결 포트가 양쪽 모두 Trunk인가?
- [ ] Trunk Encapsulation이 양쪽에서 호환되는가?
- [ ] Native VLAN이 양쪽에서 일치하는가?
- [ ] Allowed VLAN 목록에 필요한 VLAN이 포함되어 있는가?
- [ ] Allowed VLAN 설정 시 기존 목록을 실수로 덮어쓰지 않았는가?
- [ ] 불필요한 VLAN까지 Trunk에 허용하지 않았는가?
- [ ] DTP를 사용할 필요가 없다면 비활성화했는가?
- [ ] Router Sub-interface의 VLAN ID가 Switch VLAN과 일치하는가?
- [ ] 각 단말의 IP, Mask, Default Gateway가 올바른가?
- [ ] VLAN 간 ACL 또는 방화벽 정책이 통신을 차단하지 않는가?
- [ ] PortFast/BPDU Guard를 Switch 연결 포트에 잘못 적용하지 않았는가?

---

> 📌 **핵심 요약**
> - Switch는 Source MAC을 학습하고 Destination MAC으로 전달한다.
> - VLAN은 Broadcast Domain을 논리적으로 분리한다.
> - Access Port는 일반적으로 하나의 VLAN을 단말에 제공한다.
> - Trunk는 하나의 링크에서 여러 VLAN을 전달한다.
> - 802.1Q는 4Byte Tag를 사용한다.
> - Native VLAN Frame은 기본적으로 Untagged로 전송된다.
> - DTP `auto-auto` 조합은 Trunk가 형성되지 않는다.
> - 운영망에서는 Access/Trunk Mode를 수동으로 고정한다.
> - 서로 다른 VLAN 간 통신에는 L3 Routing이 필요하다.
> - Router-on-a-Stick은 VLAN별 Sub-interface를 Default Gateway로 사용한다.
> - 관련: 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 1-3. 🧩 스위치 기본 & VLAN·Trunk 통합 정리 · 1-4. 🚑 스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트 · 1-5. ⚡ 스위치 기본 & VLAN·Trunk 퀵 레퍼런스
