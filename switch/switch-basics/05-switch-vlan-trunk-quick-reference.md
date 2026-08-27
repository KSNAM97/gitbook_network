# 스위치 기본 & VLAN·Trunk 퀵 레퍼런스

> **Tag:** #Network #QuickReference #Switch #VLAN #Trunk #DTP #Dot1Q #InterVLAN  
> **핵심 요약:** VLAN·Access Port·Trunk·Router-on-a-Stick의 핵심 문법과 검증 명령어를 빠르게 확인하기 위한 암기 카드. VLAN은 Broadcast Domain을 분리하고, Access Port는 단말을 VLAN에 연결하며, Trunk는 여러 VLAN을 하나의 링크로 전달한다.

---

## 1. 도구 선택표

| 요구사항 | 설정 또는 기술 |
|---|---|
| Broadcast Domain 분리 | VLAN |
| 단말을 특정 VLAN에 연결 | Access Port |
| 여러 VLAN을 하나의 링크로 전달 | 802.1Q Trunk |
| Trunk 통과 VLAN 제한 | Allowed VLAN |
| Untagged VLAN 지정 | Native VLAN |
| Trunk 자동 협상 | DTP |
| Router로 VLAN 간 Routing | Router-on-a-Stick |
| L3 Switch로 VLAN 간 Routing | SVI + `ip routing` |
| MAC과 Port 매핑 확인 | MAC Address Table |
| L2 Loop 방지 | STP |

---

## 2. 스위치 동작 암기

```properties
Learning   = Source MAC 학습
Forwarding = Destination MAC을 알면 해당 Port로 전달
Filtering  = 목적지가 수신 Port와 같으면 다른 Port로 전달하지 않음
Flooding   = 목적지 MAC을 모르면 같은 VLAN에 전파
Aging      = 오래된 Dynamic MAC Entry 삭제
```

```properties
Switch는 Source MAC을 학습한다.
Switch는 Destination MAC으로 전달한다.
Flooding 범위는 VLAN 단위이다.
```

---

## 3. VLAN 생성

단일 VLAN:

```cisco
configure terminal
vlan 10
 name USERS
```

여러 VLAN:

```cisco
vlan 10,20,30
```

VLAN 범위:

```cisco
vlan 100-110
```

VLAN 삭제:

```cisco
no vlan 10
```

VLAN 확인:

```cisco
show vlan brief
show vlan id 10
```

> VLAN List와 Range 문법 지원 여부는 플랫폼에서 확인합니다.

---

## 4. Access Port

단일 Port:

```cisco
interface GigabitEthernet0/1
 description USER-PC
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
```

BPDU Guard:

```cisco
interface GigabitEthernet0/1
 spanning-tree bpduguard enable
```

연속 범위:

```cisco
interface range GigabitEthernet0/1 - 8
 switchport mode access
 switchport access vlan 10
```

비연속 범위:

```cisco
interface range GigabitEthernet0/13, GigabitEthernet0/15
 switchport mode access
 switchport access vlan 20
```

Access Port 확인:

```cisco
show interfaces GigabitEthernet0/1 switchport
show interfaces status
show vlan brief
```

---

## 5. Trunk 기본 설정

고정 802.1Q 플랫폼:

```cisco
interface GigabitEthernet0/24
 description TRUNK-TO-SW2
 switchport mode trunk
```

Encapsulation 선택을 지원하는 플랫폼:

```cisco
interface GigabitEthernet0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

DTP 비활성화:

```cisco
interface GigabitEthernet0/24
 switchport mode trunk
 switchport nonegotiate
```

Trunk 확인:

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show dtp interface GigabitEthernet0/24
```

---

## 6. DTP 조합

| SW1 | SW2 | 결과 |
|---|---|---|
| trunk | trunk | Trunk |
| trunk | dynamic auto | Trunk |
| trunk | dynamic desirable | Trunk |
| dynamic desirable | dynamic auto | Trunk |
| dynamic desirable | dynamic desirable | Trunk |
| dynamic auto | dynamic auto | Access 상태 |
| access | access | Access |
| access | trunk | Mode 불일치 |

DTP Mode:

```cisco
switchport mode dynamic desirable
switchport mode dynamic auto
```

운영 권장:

```cisco
switchport mode access
```

또는:

```cisco
switchport mode trunk
switchport nonegotiate
```

---

## 7. Allowed VLAN

현재 목록을 교체:

```cisco
switchport trunk allowed vlan 10,20,30
```

VLAN 추가:

```cisco
switchport trunk allowed vlan add 40
```

VLAN 제거:

```cisco
switchport trunk allowed vlan remove 20
```

특정 VLAN을 제외:

```cisco
switchport trunk allowed vlan except 100
```

모든 VLAN 허용 상태로 복원:

```cisco
no switchport trunk allowed vlan
```

확인:

```cisco
show interfaces trunk
show running-config interface GigabitEthernet0/24
```

> 기존 목록에 VLAN을 추가할 때는 `add`를 사용한다. `allowed vlan 40`은 기존 목록을 VLAN 40으로 덮어쓸 수 있다.

---

## 8. Native VLAN

Native VLAN 변경:

```cisco
vlan 999
 name NATIVE-BLACKHOLE
!
interface GigabitEthernet0/24
 switchport trunk native vlan 999
```

확인:

```cisco
show interfaces trunk
show interfaces GigabitEthernet0/24 switchport
show logging
```

```properties
Trunk 양쪽 Native VLAN은 반드시 일치
Native VLAN Frame은 기본적으로 Untagged
사용자 VLAN을 Native VLAN으로 사용하지 않는 방식 권장
```

---

## 9. Router-on-a-Stick

Router:

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
```

Switch:

```cisco
vlan 10
 name USERS
!
vlan 20
 name SERVERS
!
interface GigabitEthernet0/24
 description TRUNK-TO-R1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

PC Gateway:

```properties
VLAN 10 → 192.168.10.254
VLAN 20 → 192.168.20.254
```

Native VLAN Sub-interface:

```cisco
interface GigabitEthernet0/0.999
 encapsulation dot1q 999 native
 ip address 192.168.99.254 255.255.255.0
```

---

## 10. MAC Address Table

전체 확인:

```cisco
show mac address-table
```

Dynamic Entry:

```cisco
show mac address-table dynamic
```

VLAN별 확인:

```cisco
show mac address-table vlan 10
```

Port별 확인:

```cisco
show mac address-table interface GigabitEthernet0/1
```

특정 MAC 확인:

```cisco
show mac address-table address aaaa.bbbb.cccc
```

Aging Time:

```cisco
show mac address-table aging-time
```

Dynamic MAC 삭제:

```cisco
clear mac address-table dynamic
```

특정 Interface의 Dynamic MAC 삭제:

```cisco
clear mac address-table dynamic interface GigabitEthernet0/1
```

> MAC Table 삭제는 장애 원인 분석 후 제한적으로 사용한다. 삭제 후 Unknown Unicast Flooding이 일시적으로 증가할 수 있다.

---

## 11. 필수 검증 명령어

VLAN:

```cisco
show vlan brief
show vlan id <VLAN-ID>
```

Switchport:

```cisco
show interfaces <INTERFACE> switchport
show interfaces status
show running-config interface <INTERFACE>
```

Trunk:

```cisco
show interfaces trunk
show dtp interface <INTERFACE>
```

MAC:

```cisco
show mac address-table
show mac address-table vlan <VLAN-ID>
```

STP:

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree interface <INTERFACE>
```

Router-on-a-Stick:

```cisco
show ip interface brief
show ip route connected
show arp
show interfaces <PHYSICAL-INTERFACE.SUBINTERFACE>
```

Log와 Err-disabled:

```cisco
show logging
show interfaces status err-disabled
show errdisable recovery
```

---

## 12. 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| `switchport mode access` | Access Mode로 고정 |
| `switchport access vlan 10` | Untagged 단말 Frame을 VLAN 10으로 분류 |
| `switchport mode trunk` | Trunk Mode로 고정 |
| `switchport nonegotiate` | DTP 협상 Frame 전송 억제 |
| `dynamic auto + dynamic auto` | Trunk 형성 안 됨 |
| `trunk + dynamic auto` | 일반적으로 Trunk 형성 |
| `allowed vlan 10,20` | 허용 목록을 10, 20으로 교체 |
| `allowed vlan add 30` | 기존 목록에 VLAN 30 추가 |
| Native VLAN 불일치 | Untagged Frame이 서로 다른 VLAN으로 분류될 수 있음 |
| VLAN 삭제 | Port의 Access VLAN 설정은 남을 수 있음 |
| VLAN이 없음 | 해당 VLAN을 참조하는 Port가 Inactive 가능 |
| Trunk Up | 모든 VLAN 통신 보장을 의미하지 않음 |
| Same VLAN 통신 | Router가 필요하지 않음 |
| Different VLAN 통신 | Router 또는 L3 Switch 필요 |
| Sub-interface Down | 물리 Interface 상태부터 확인 |
| MAC Table에 없음 | 아직 Traffic이 없거나 L2 경로 문제 |
| Broadcast | 같은 VLAN 내부에서 Flooding |
| Unknown Unicast | 목적지 MAC을 모르면 같은 VLAN에 Flooding |

---

## 13. 빠른 장애 진단 순서

```properties
Port Up 확인
    ↓
VLAN 생성 확인
    ↓
Access VLAN 확인
    ↓
Trunk Status 확인
    ↓
Allowed VLAN 확인
    ↓
Native VLAN 확인
    ↓
STP Forwarding 확인
    ↓
MAC 학습 확인
    ↓
IP/Mask/Gateway 확인
    ↓
Sub-interface/SVI 및 Routing 확인
    ↓
ACL·Port Security 확인
```

---

## 14. 안전한 정책 변경

변경 전:

```cisco
show vlan brief
show interfaces trunk
show interfaces <INTERFACE> switchport
show spanning-tree vlan <VLAN-ID>
show running-config interface <INTERFACE>
```

Allowed VLAN 추가:

```cisco
interface <TRUNK-INTERFACE>
 switchport trunk allowed vlan add <VLAN-ID>
```

변경 후:

```cisco
show interfaces trunk
show vlan brief
show mac address-table vlan <VLAN-ID>
ping <DESTINATION>
```

Rollback 예시:

```cisco
interface <TRUNK-INTERFACE>
 switchport trunk allowed vlan remove <VLAN-ID>
```

---

> 📌 **핵심 요약**
> - VLAN 생성: `vlan <ID>`
> - Access 설정: `switchport mode access`
> - Access VLAN: `switchport access vlan <ID>`
> - Trunk 설정: `switchport mode trunk`
> - Allowed VLAN 추가: `switchport trunk allowed vlan add <ID>`
> - Native VLAN: `switchport trunk native vlan <ID>`
> - VLAN 확인: `show vlan brief`
> - Trunk 확인: `show interfaces trunk`
> - Port Mode 확인: `show interfaces <IF> switchport`
> - MAC 확인: `show mac address-table`
> - Inter-VLAN Routing: Router Sub-interface 또는 L3 Switch SVI
> - 관련: 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 1-3. 🧩 스위치 기본 & VLAN·Trunk 통합 정리 · 1-4. 🚑 스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트 · 1-5. ⚡ 스위치 기본 & VLAN·Trunk 퀵 레퍼런스
