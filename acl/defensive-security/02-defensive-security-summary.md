# 🧩 스위치 & 방어 보안 통합 정리 — Port-security·BPDU Guard·TCP Intercept

> **Tag:** #Network #Security #PortSecurity #BPDUGuard #ErrDisable #TCPIntercept #SYNFlood #DefensiveSecurity
> **핵심 요약:** Port-security는 Access Port에서 허용할 Source MAC을 제한하는 L2 방어 기술이고, BPDU Guard는 사용자 포트로 들어오는 BPDU를 감지해 STP Loop를 차단한다. TCP Intercept는 보호 대상 서버로 향하는 TCP SYN을 감시하거나 Proxy 처리하여 SYN Flood를 완화한다. 각 기술은 방어 계층과 목적이 다르며, Port-security는 802.1X를, TCP Intercept는 ZBF·방화벽·DDoS Scrubbing을 대체하지 않는다.

---

## 1. 🎯 핵심 기술 개념

### Q1. Port-security와 TCP Intercept의 방어 계층은?

👉 **핵심 답변:** Port-security는 Access Port에서 Source MAC을 제한하는 **L2 접근 제어**, TCP Intercept는 서버로 향하는 TCP 연결 요청을 검사하는 **L4 세션 방어**입니다.

```properties
[L2 Access Edge]
Port-security
BPDU Guard
DTP Disable
Storm-control

        ↓

[L3/L4 Network Edge]
ACL
uRPF
TCP Intercept
ZBF/Firewall

        ↓

[Application/Internet Edge]
WAF
CDN
DDoS Scrubbing
IPS/NGFW
```

| 기술 | 계층 | 보호 대상 | 주요 위협 |
|---|---|---|---|
| Port-security | L2 | Access Port | 무단 단말·MAC 이동 |
| BPDU Guard | L2/STP | Access Port | 비인가 스위치·STP Loop |
| Storm-control | L2 | Switch/Segment | Broadcast·Multicast 폭주 |
| TCP Intercept | L4 | TCP Server | SYN Flood |
| ACL/ZBF | L3~L7 | Network/Session | 비인가 트래픽 |
| 802.1X | 인증/NAC | 사용자·장비 | 비인가 접속 |

---

### Q2. Port-security와 802.1X는 어떻게 다른가?

👉 **핵심 답변:** Port-security는 MAC 주소를 허용 기준으로 사용하지만, 802.1X는 사용자 또는 장비 자격 증명을 AAA 서버로 검증합니다.

| 구분 | Port-security | 802.1X |
|---|---|---|
| 식별 기준 | Source MAC | 사용자·장비 인증 |
| 중앙 인증 | 없음 | RADIUS/AAA |
| 동적 VLAN/ACL | 제한적 | 가능 |
| MAC Spoofing 저항성 | 낮음 | 상대적으로 높음 |
| 구축 난이도 | 낮음 | 높음 |
| 적합 환경 | 고정 장비·단순 서버 포트 | 사무실·BYOD·NAC |

- 서버, 프린터, 고정형 IoT 등 단순 환경: Port-security 고려
- 사용자 노트북, BYOD, 정책 기반 VLAN: 802.1X/MAB/NAC 고려
- 현대 Catalyst에서는 Port-security와 802.1X 조합 시 플랫폼 제한과 세션 삭제 문제가 있을 수 있으므로 Feature Compatibility를 확인
- Port-security를 802.1X의 대체 기술로 보지 않음

---

### Q3. Sticky MAC은 어떻게 동작하는가?

👉 **핵심 답변:** Sticky Learning을 활성화하면 동적으로 학습한 Secure MAC이 Running-config에 추가됩니다.

```cisco
switchport port-security mac-address sticky
```

```properties
Dynamic Secure MAC
    ↓ Sticky Learning
Sticky Secure MAC
    ↓ copy running-config startup-config
재부팅 후에도 유지
```

- 저장하지 않으면 재부팅 후 사라질 수 있음
- Sticky를 끄면 Sticky MAC이 Dynamic Secure MAC으로 변환되고 Running-config에서 제거될 수 있음
- 일부 최신 Catalyst 플랫폼에서는 Sticky Secure MAC Aging을 지원하지 않으므로 모델·IOS XE 확인 필요

---

### Q4. Violation Mode는 어떻게 선택하는가?

| 모드 | 위반 Source 프레임 | 포트 | 알림/카운터 | 적합 환경 |
|---|---|---|---|---|
| `protect` | Drop | Up | 거의 없음 | 제한적 임시 환경 |
| `restrict` | Drop | Up | Syslog/SNMP/Count | 일반 사용자 포트 |
| `shutdown` | Drop | Err-disabled | Log/Count | 보안 우선 환경 |
| `shutdown vlan` | Drop | 해당 VLAN 제한 | 플랫폼 의존 | Secure Trunk 등 |

#### Protect

```cisco
switchport port-security violation protect
```

- 위반 프레임만 폐기
- 포트는 Up
- 사고 흔적이 부족해 운영 가시성이 낮음

#### Restrict

```cisco
switchport port-security violation restrict
```

- 위반 프레임 폐기
- 포트 Up 유지
- Violation Count 증가
- Syslog/SNMP 알림 지원

#### Shutdown

```cisco
switchport port-security violation shutdown
```

- 포트 전체를 Err-disabled로 전환
- 기본 Violation Mode
- 보안 수준은 높지만 사용자 영향과 운영 부담이 큼

> `restrict`가 일반 사용자 포트에 유용하지만 모든 환경의 절대 표준은 아닙니다. 서버룸·보안망에서는 `shutdown`, 가용성 우선 단말 포트에서는 `restrict`를 선택할 수 있습니다.

---

### Q5. BPDU Guard는 Port-security와 무엇이 다른가?

👉 **핵심 답변:** Port-security는 Source MAC 개수를 제한하고, BPDU Guard는 사용자 포트에서 BPDU를 수신했을 때 Port를 Err-disable하여 STP Loop를 방지합니다.

```cisco
interface FastEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
```

- PortFast: End Host Port를 빠르게 Forwarding 상태로 전환
- BPDU Guard: BPDU 유입 시 비인가 스위치 연결로 판단
- Switch, Bridge, Hypervisor Bridge 등이 연결될 포트에는 신중히 적용
- Uplink/Trunk에는 일반적인 PortFast Edge + BPDU Guard 조합을 적용하지 않음

---

### Q6. TCP Intercept의 Watch와 Intercept Mode 차이는?

| 모드 | 동작 | 서버가 SYN 수신 | Router 부하 |
|---|---|---|---|
| Watch | 연결을 통과시키고 감시 | 수신함 | 상대적으로 낮음 |
| Intercept | Router가 Client/Server Handshake 대행 | 검증 전 직접 수신하지 않음 | 높음 |

#### Watch Mode

```properties
Client SYN → Server로 전달
Router가 세션 성립 여부 감시
지정 시간 내 미완료 시 Server에 RST 전송
```

#### Intercept Mode

```properties
Client SYN → Router가 SYN-ACK 응답
Client 검증 성공
Router가 Server와 별도 Handshake
두 Half-Connection 연결
```

> Intercept Mode는 SYN Flood 방어가 강하지만 Router CPU와 Memory를 사용합니다. 대규모 환경에서는 전용 DDoS 방어 장비, CDN 또는 Scrubbing Service가 우선입니다.

---

### Q7. TCP Intercept ACL의 Permit/Deny는 무엇을 의미하는가?

👉 **핵심 답변:** TCP Intercept에 연결된 ACL은 트래픽을 통과·차단하는 필터가 아니라 **TCP Intercept 처리 대상을 선택하는 Classification ACL**입니다.

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2
!
ip tcp intercept list TCP_PROTECT
```

```properties
ACL permit 매칭 = TCP Intercept 처리
ACL deny/미매칭 = Intercept하지 않고 정상 전달
```

> 일반 Interface ACL의 `permit/deny` 의미와 혼동하면 안 됩니다.

---

### Q8. TCP Intercept를 NAT/ZBF와 같이 사용할 수 있는가?

👉 **핵심 답변:** Cisco 문서에서는 TCP Intercept를 NAT, Zone-Based Firewall 또는 CBAC와 함께 구성하지 않도록 제한합니다.

```properties
TCP Intercept + NAT  → 비권장/제한
TCP Intercept + ZBF  → 비권장/제한
TCP Intercept + CBAC → 비권장/제한
```

신규 설계에서는 다음 우선순위를 권장합니다.

```properties
ZBF/NGFW/DDoS Appliance
        ↓
플랫폼 기능이 제한된 Legacy/Branch 환경에서 TCP Intercept 검토
```

---

## 2. 🛠️ Port-security 설정

### 2-1. Sticky 방식

```cisco
interface range FastEthernet0/1 - 20
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
!
end
copy running-config startup-config
```

> `switchport port-security`가 반드시 마지막일 필요는 없지만, 필요한 세부 정책을 먼저 구성한 후 활성화하면 중간 상태에서 발생할 수 있는 장애를 줄일 수 있습니다.

---

### 2-2. 정적 MAC 방식

```cisco
interface FastEthernet0/1
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security maximum 1
 switchport port-security mac-address 00D0.BA58.EE9A
 switchport port-security violation shutdown
 switchport port-security
```

---

### 2-3. IP Phone + PC

```cisco
interface FastEthernet0/10
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

> 기본적으로 Phone MAC 1개와 PC MAC 1개를 고려한 값입니다. Dock, Hypervisor, 추가 단말이 존재하면 실제 Source MAC 수를 조사해야 합니다.

VLAN별 Maximum을 지원하는 플랫폼:

```cisco
interface FastEthernet0/10
 switchport port-security maximum 1 vlan access
 switchport port-security maximum 1 vlan voice
```

---

### 2-4. Secure MAC Aging

```cisco
interface FastEthernet0/10
 switchport port-security aging time 60
 switchport port-security aging type inactivity
```

| Aging Type | 동작 |
|---|---|
| `absolute` | 지정 시간이 지나면 삭제 |
| `inactivity` | 지정 시간 동안 사용하지 않으면 삭제 |

> 일부 Catalyst 플랫폼은 Sticky Secure MAC Aging을 지원하지 않습니다. Sticky와 Aging을 함께 사용할 경우 반드시 해당 플랫폼 문서를 확인합니다.

---

### 2-5. Err-disable 자동 복구

```cisco
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```

BPDU Guard 자동 복구:

```cisco
errdisable recovery cause bpduguard
```

> BPDU를 발생시키는 비인가 스위치가 계속 연결돼 있으면 자동 복구 후 다시 Err-disable됩니다. BPDU Guard 자동 복구는 보안 정책과 현장 대응 절차를 고려해 선택합니다.

---

## 3. 🧱 Access Layer 표준 템플릿

### 3-1. 관리 접근 제어

```cisco
ip access-list standard ACL_MGMT
 10 permit 10.10.10.0 0.0.0.255
 20 deny any log
!
line vty 0 4
 access-class ACL_MGMT in
 transport input ssh
 login local
```

---

### 3-2. 일반 PC 포트 프로파일

```cisco
interface range FastEthernet0/1 - 12
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

---

### 3-3. IP Phone + PC 포트 프로파일

```cisco
interface range FastEthernet0/13 - 20
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

---

### 3-4. Uplink 포트

```cisco
interface GigabitEthernet0/1
 description UPLINK_TO_DISTRIBUTION
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,100
 switchport nonegotiate
```

> 일부 플랫폼은 Trunk Port-security를 지원하지만 다수의 MAC이 통과하는 일반 Uplink에는 부적합합니다. Port-security를 무조건 설정하지 말고 STP 보호, Allowed VLAN, DHCP Snooping Trust 등 Uplink 전용 정책을 사용합니다.

---

### 3-5. 적용 제외 또는 별도 검토 대상

다음 포트에 일반 PC용 Sticky Maximum 1 정책을 그대로 적용하면 안 됩니다.

- Switch/Router Uplink
- Access Point
- IP Phone + PC
- Hypervisor/Virtualization Host
- 서버 Bonding/Teaming
- Docking Station
- 802.1X/MAB 인증 포트
- SPAN Destination
- EtherChannel Member
- Private VLAN Port
- 다수의 MAC이 정상적으로 통과하는 장비

---

## 4. 🛡️ TCP Intercept 설정

### 4-1. Watch Mode

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2
!
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode watch
ip tcp intercept watch-timeout 20
!
ip tcp intercept max-incomplete low 900 high 1100
ip tcp intercept one-minute low 900 high 1100
ip tcp intercept drop-mode oldest
```

구형 IOS 문법:

```cisco
ip tcp intercept max-incomplete high 1100
ip tcp intercept max-incomplete low 900
ip tcp intercept one-minute high 1100
ip tcp intercept one-minute low 900
```

> 플랫폼과 IOS 버전에 따라 Threshold 문법이 다르므로 `?` 및 Command Reference로 확인합니다.

---

### 4-2. Intercept Mode

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2
!
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode intercept
ip tcp intercept connection-timeout 30
```

> `connection-timeout`은 TCP Intercept가 관리하는 연결에서 트래픽이 없는 상태가 지속될 때 제거하는 Idle Timeout입니다. Client와 Server의 3-Way Handshake 완료 제한 시간을 뜻하지 않습니다.

---

### 4-3. 특정 서비스만 보호

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2 eq 80
 permit tcp any host 211.241.228.2 eq 443
!
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode watch
```

---

### 4-4. Threshold 동작

```properties
Incomplete High 초과
또는
One-minute High 초과
        ↓
Aggressive Mode 진입
        ↓
Partial Connection 적극 제거
        ↓
수치가 Low 아래로 감소
        ↓
Normal Mode 복귀
```

```cisco
ip tcp intercept max-incomplete low 900 high 1100
ip tcp intercept one-minute low 900 high 1100
ip tcp intercept drop-mode oldest
```

- `oldest`: 가장 오래된 Half-open Connection 삭제
- `random`: 임의 Half-open Connection 삭제

---

## 5. 🔍 검증 명령어

### Port-security

```cisco
show port-security
show port-security interface FastEthernet0/1
show port-security address
show port-security interface FastEthernet0/1 address
show running-config interface FastEthernet0/1
show mac address-table interface FastEthernet0/1
```

### Err-disable/BPDU Guard

```cisco
show interfaces status err-disabled
show errdisable recovery
show errdisable detect
show logging | include ERR_DISABLE|PSECURE|BPDU
show spanning-tree interface FastEthernet0/1 detail
```

### TCP Intercept

```cisco
show tcp intercept connections
show tcp intercept statistics
show running-config | include ip tcp intercept
show ip access-lists TCP_PROTECT
show processes cpu sorted
show processes memory sorted
```

운영망 주의:

```cisco
debug ip tcp intercept
undebug all
```

---

## 6. ⚠️ 실무 체크리스트

- [ ] 포트 역할별로 PC, Phone, AP, Server, Uplink 프로파일을 분리했는가?
- [ ] 실제 정상 Source MAC 개수를 확인했는가?
- [ ] Sticky MAC 저장 정책과 장비 교체 절차가 있는가?
- [ ] Violation Mode를 보안성과 가용성 기준으로 선택했는가?
- [ ] PortFast/BPDU Guard가 End Host Port에만 적용됐는가?
- [ ] 802.1X Port에 Port-security를 함께 적용하기 전 호환성을 확인했는가?
- [ ] Sticky Secure MAC Aging 지원 여부를 확인했는가?
- [ ] Err-disable 자동 복구 전에 Root Cause를 제거하는 절차가 있는가?
- [ ] TCP Intercept ACL의 Permit이 처리 대상 선택임을 이해했는가?
- [ ] TCP Intercept Threshold를 정상 Traffic Baseline 기준으로 설정했는가?
- [ ] NAT/ZBF/CBAC와 TCP Intercept를 혼합하지 않았는가?
- [ ] Router CPU/Memory를 지속적으로 감시하는가?
- [ ] 대규모 SYN Flood는 전용 DDoS 방어 체계로 처리하는가?

---

> 📌 **핵심 요약**
> - Port-security = L2 Source MAC 제한
> - BPDU Guard = 사용자 포트 BPDU 수신 시 Loop 방어
> - Sticky MAC은 Running-config에 기록되며 저장해야 재부팅 후 유지
> - Restrict는 가시성·가용성, Shutdown은 보안 우선
> - TCP Intercept Watch = 통과 후 감시
> - TCP Intercept Intercept = Router가 Handshake Proxy 수행
> - TCP Intercept ACL Permit = 검사 대상 선택
> - 관련: 3-3-1. 🔒  스위치 & 방어 보안 - Port-security & TCP Intercept · 6-1. 🔐 Port-security & Storm-control · 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리
