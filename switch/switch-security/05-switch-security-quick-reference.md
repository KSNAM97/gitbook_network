# ⚡ Switch 보안 명령어 퀵 레퍼런스

> **Tag:** #Network #Switch #Security #PortSecurity #SPAN #Cisco  
> **핵심 요약:** Port-security, Storm-control, SPAN/RSPAN, 미사용 Port·VLAN 격리 명령을 검증과 함정 위주로 정리한 암기 카드. Port-security는 반드시 Access Port에서, SPAN Destination은 반드시 미사용 전용 Port에서 사용하며 두 기능은 같은 Port에 병행할 수 없다.

---

## 1. 🧠 요구사항별 기능 선택

| 요구사항 | 사용 기능 |
|---|---|
| Access Port에 붙는 장비 수 제한 | Port-security `maximum` |
| 허용 MAC 수동 지정 | `mac-address <MAC-ADDRESS>` |
| 허용 MAC 자동 학습 후 고정 | `mac-address sticky` |
| 위반 시 조용히 폐기 | Violation `protect` |
| 위반 시 폐기 + 기록 | Violation `restrict` |
| 위반 시 Port 격리 | Violation `shutdown` |
| Broadcast 폭주 억제 | Storm-control `broadcast` |
| 폭주 시 알림만 | `storm-control action trap` |
| 폭주 시 Port 격리 | `storm-control action shutdown` |
| 같은 Switch 내 Traffic 복제 | Local SPAN |
| 다른 Switch로 Traffic 복제 | RSPAN + `remote-span` VLAN |
| Trunk에서 특정 VLAN만 복제 | `monitor session ... filter vlan` |
| Err-disable Port 자동 복구 | `errdisable recovery cause ...` |
| 미사용 Port 진입점 축소 | `shutdown` + 격리 VLAN |
| 비인가 Switch 연결 차단 | PortFast + BPDU Guard |
| Trunk VLAN 경계 명확화 | `switchport trunk native vlan` + `nonegotiate` |

---

## 2. 📊 Violation Mode 비교

| Violation Mode | 위반 Frame 폐기 | Syslog 로그 | SNMP Trap | Violation Counter 증가 | Port 상태 |
|---|:---:|:---:|:---:|:---:|---|
| `protect` | O | X | X | X | Up 유지 |
| `restrict` | O | O | O | O | Up 유지 |
| `shutdown` | O | O | O | O | Err-disabled (Secure-shutdown) |
| `shutdown vlan` | O | O | O | O | 해당 VLAN만 차단 (플랫폼별 지원 여부 확인 필요) |

```properties
기본 Violation Mode = shutdown
기본 Maximum        = 1
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 3. 📊 Secure MAC 학습 방식 비교

| 학습 방식 | 등록 주체 | Running-config 기록 | 재부팅 후 유지 | 기본 Aging 대상 |
|---|---|:---:|:---:|:---:|
| Static | 관리자 | O | 저장 시 O | X |
| Dynamic | 자동 학습 | X | X | O |
| Sticky | 자동 학습 후 고정 | O | 저장 시 O | X |

---

## 4. 🔧 Port-security 기본 명령

```cisco
interface <INTERFACE>
 switchport mode access
 switchport port-security maximum 1
 switchport port-security mac-address <MAC-ADDRESS>
 switchport port-security mac-address sticky
 switchport port-security violation protect
 switchport port-security violation restrict
 switchport port-security violation shutdown
 switchport port-security
```

검증:

```cisco
show port-security
show port-security interface <INTERFACE>
show port-security address
show running-config interface <INTERFACE>
```

---

## 5. 🖥️ 표준 프로파일

### 5-1. 일반 사용자 Port

```cisco
configure terminal
!
interface <INTERFACE>
 description ACCESS_USER
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

검증:

```cisco
show port-security interface <INTERFACE>
show interfaces <INTERFACE> switchport
```

---

### 5-2. 고정 장비 Port

```cisco
configure terminal
!
interface <INTERFACE>
 description SERVER_FIXED
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport nonegotiate
 !
 switchport port-security maximum 1
 switchport port-security mac-address <MAC-ADDRESS>
 switchport port-security violation shutdown
 switchport port-security
```

검증:

```cisco
show port-security interface <INTERFACE>
show port-security address
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 5-3. IP Phone + PC Port

```cisco
configure terminal
!
interface <INTERFACE>
 description PHONE_PLUS_PC
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport voice vlan <VLAN-ID>
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 !
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

VLAN별 Maximum 지원 플랫폼:

```cisco
interface <INTERFACE>
 switchport port-security maximum 1 vlan access
 switchport port-security maximum 1 vlan voice
```

검증:

```cisco
show port-security interface <INTERFACE>
show interfaces <INTERFACE> switchport
```

---

## 6. ⏱️ Secure MAC Aging

```cisco
interface <INTERFACE>
 switchport port-security aging time 60
 switchport port-security aging type inactivity
 switchport port-security aging static
```

검증:

```cisco
show port-security interface <INTERFACE>
show port-security address
```

| Aging Type | 동작 |
|---|---|
| `absolute` | 등록 후 지정 시간 경과 시 삭제 |
| `inactivity` | 지정 시간 동안 Traffic 없으면 삭제 |

```properties
aging time 단위 = 분(Minute)
기본값          = 0 (Aging 없음)
aging static    = Static Secure MAC도 Aging 대상 포함
```

> Sticky Secure MAC의 Aging 지원 여부는 플랫폼별 지원 여부 확인 필요입니다.

---

## 7. 🧹 초기화 및 Err-disable 복구

```cisco
clear port-security sticky interface <INTERFACE>
clear port-security dynamic interface <INTERFACE>
clear port-security sticky address <MAC-ADDRESS>
!
configure terminal
interface <INTERFACE>
 shutdown
 no shutdown
```

자동 복구:

```cisco
configure terminal
!
errdisable recovery cause psecure-violation
errdisable recovery cause storm-control
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

검증:

```cisco
show interfaces status err-disabled
show errdisable recovery
show errdisable detect
show port-security address
```

```properties
errdisable recovery interval 단위 = 초(Second)
자동 복구 = Port를 다시 Up
자동 복구 ≠ 원인 제거
```

> 원인이 남아 있으면 복구 후 즉시 재차단됩니다. Interval을 지나치게 짧게 설정하면 Port Flapping과 STP 재수렴이 반복됩니다.

---

## 8. 🌊 Storm-control

```cisco
interface <INTERFACE>
 storm-control broadcast level 30.00 20.00
 storm-control multicast level pps 3k 2k
 storm-control unicast level bps 50m 40m
 storm-control action trap
 storm-control action shutdown
```

검증:

```cisco
show storm-control
show storm-control <INTERFACE>
show storm-control broadcast
show interfaces <INTERFACE> counters
```

| Action | Port 상태 | 복구 방식 |
|---|---|---|
| 미지정 (기본) | Up 유지 | Lower 이하 감소 시 자동 |
| `trap` | Up 유지 | 자동 (SNMP Trap 발생) |
| `shutdown` | Err-disabled | 수동 또는 `errdisable recovery cause storm-control` |

```properties
level <UPPER> <LOWER>      = 백분율 기준
level pps <UPPER> <LOWER>  = 초당 패킷 수
level bps <UPPER> <LOWER>  = 초당 비트 수

Upper만 지정     = Lower도 동일 값 → Flapping 가능
Upper/Lower 분리 = 임계치 부근 Flapping 억제
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> `pps`/`bps` 옵션, `k`/`m`/`g` 접미사, 소수점 허용 범위는 플랫폼과 IOS 버전에 따라 다릅니다. Packet Tracer는 백분율 기반만 정상 동작하거나 `pps`/`bps` 입력을 거부할 수 있고 `show storm-control` 출력도 실장비와 다를 수 있습니다.

---

## 9. 🔎 Local SPAN

```cisco
configure terminal
!
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> source interface <INTERFACE> rx
monitor session <NUMBER> source interface <INTERFACE> tx
monitor session <NUMBER> filter vlan <VLAN-ID>
monitor session <NUMBER> destination interface <INTERFACE>
```

VLAN SPAN:

```cisco
monitor session <NUMBER> source vlan <VLAN-ID> rx
monitor session <NUMBER> destination interface <INTERFACE>
```

Session 삭제:

```cisco
no monitor session <NUMBER>
no monitor session all
```

검증:

```cisco
show monitor
show monitor session <NUMBER>
show monitor session all
```

| 방향 | 복제 대상 |
|---|---|
| `rx` | Source로 들어오는 Traffic |
| `tx` | Source에서 나가는 Traffic |
| `both` | 양방향 (기본값) |

> `filter vlan`은 Trunk Source Port에서 특정 VLAN만 복제할 때 사용합니다. VLAN SPAN과 `filter vlan`의 지원 여부는 플랫폼별 지원 여부 확인 필요입니다.

---

## 10. 🛰️ RSPAN

양쪽 Switch 공통:

```cisco
configure terminal
!
vlan <VLAN-ID>
 remote-span
```

Source Switch:

```cisco
monitor session <NUMBER> source interface <INTERFACE> both
monitor session <NUMBER> destination remote vlan <VLAN-ID>
```

Destination Switch:

```cisco
monitor session <NUMBER> source remote vlan <VLAN-ID>
monitor session <NUMBER> destination interface <INTERFACE>
```

경로 Trunk 허용:

```cisco
interface <INTERFACE>
 switchport mode trunk
 switchport trunk allowed vlan add <VLAN-ID>
```

검증:

```cisco
show vlan remote-span
show monitor session <NUMBER>
show interfaces trunk
```

```properties
RSPAN 성립 조건
[1] 양쪽 Switch에 동일 VLAN-ID + remote-span 지정
[2] 경로상 모든 Trunk의 Allowed VLAN에 해당 VLAN 포함
[3] Source Switch      : destination remote vlan
[4] Destination Switch : source remote vlan
[5] RSPAN VLAN에 일반 Access Port·SVI 미할당
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> 일부 플랫폼과 Packet Tracer는 RSPAN Source Session에 `reflector-port <INTERFACE>` 지정을 요구합니다. Reflector Port는 미사용 Port여야 하며, 요구 여부는 플랫폼별 지원 여부 확인 필요입니다.

---

## 11. 🚫 SPAN Destination Port 특성

| 특성 | 동작 |
|---|---|
| 일반 Forwarding | 수행하지 않음 |
| MAC 학습 | 수행하지 않음 |
| STP 참여 | 참여하지 않음 |
| 수신 Traffic | 기본 폐기 |
| Port-security 병행 | 불가 |
| EtherChannel Member 병행 | 불가 |
| 다중 Session Destination | 불가 |

확인:

```cisco
show monitor session <NUMBER>
show interfaces <INTERFACE> switchport
show mac address-table interface <INTERFACE>
```

```properties
Destination Port Ping 실패        = 정상 동작
Destination Port에 서비스 단말 연결 = 즉시 통신 중단
```

---

## 12. 🧱 미사용 Port 및 VLAN 격리

```cisco
configure terminal
!
interface range <INTERFACE>
 description UNUSED_PORT
 switchport mode access
 switchport access vlan <VLAN-ID>
 switchport nonegotiate
 no cdp enable
 shutdown
```

검증:

```cisco
show interfaces status
show interfaces description
show vlan brief
```

```properties
[1] shutdown       = 물리적 활성화 차단
[2] 격리 VLAN 할당  = SVI·Gateway 없는 VLAN
[3] nonegotiate    = Trunk 자동 형성 차단
[4] no cdp enable  = 불필요한 정보 노출 축소
```

> 격리 VLAN을 Native VLAN이나 관리 VLAN으로 재사용하면 격리 효과가 사라집니다. VLAN은 Broadcast Domain 분리이며 완전한 보안 경계가 아닙니다.

---

## 13. 🚚 Native VLAN 및 Trunk 보안

```cisco
configure terminal
!
interface <INTERFACE>
 description TRUNK_TO_DISTRIBUTION
 switchport mode trunk
 switchport trunk native vlan <VLAN-ID>
 switchport trunk allowed vlan <VLAN-ID>
 switchport nonegotiate
```

검증:

```cisco
show interfaces trunk
show interfaces <INTERFACE> switchport
```

Rollback:

```cisco
interface <INTERFACE>
 no switchport trunk native vlan
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

> Native VLAN은 Trunk 양쪽을 같은 Maintenance Window에 동일하게 변경합니다. 한쪽만 변경하면 Untagged Frame이 서로 다른 VLAN으로 해석됩니다.

---

## 14. 🛡️ BPDU Guard 연계

```cisco
configure terminal
!
interface range <INTERFACE>
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

전역 기본값 적용:

```cisco
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

검증:

```cisco
show spanning-tree interface <INTERFACE> detail
show spanning-tree summary
show interfaces status err-disabled
```

```properties
PortFast   = End Device Port 전용
BPDU Guard = BPDU 수신 시 Err-disable

PortFast 단독         → 비인가 Switch 연결 시 Loop 위험
PortFast + BPDU Guard → 비인가 Switch 연결 즉시 차단
Uplink/Trunk          → 이 조합 적용하지 않음
```

> 상세 내용은 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard)에서 확인하십시오.

---

## 15. 🔄 안전한 변경 원칙

```properties
현재 확인 → 백업 → 관리 경로 확인 → 적용 → 검증 → 저장
```

변경 전:

```cisco
show running-config interface <INTERFACE>
show interfaces status
show port-security interface <INTERFACE>
show storm-control <INTERFACE>
show monitor session all
show interfaces trunk
```

Rollback:

```cisco
configure terminal
!
interface <INTERFACE>
 no switchport port-security
 no storm-control broadcast level
 no storm-control action shutdown
 no switchport trunk native vlan
!
no monitor session <NUMBER>
```

변경 후:

```cisco
show port-security interface <INTERFACE>
show storm-control <INTERFACE>
show monitor session <NUMBER>
show interfaces status err-disabled
show logging
!
copy running-config startup-config
```

---

## 16. ⚠️ 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| `switchport port-security`만 입력 | Maximum 1, Violation shutdown이 기본 적용 |
| Port가 Access Mode가 아님 | Port-security 적용이 거부되거나 동작하지 않음 |
| `protect` Mode | 위반은 막지만 로그·Counter가 없어 추적 불가 |
| `restrict` Mode | Port는 Up 유지, 위반 Frame만 폐기 |
| `shutdown` Mode | Port 전체가 Err-disabled (Secure-shutdown) |
| Sticky 학습 후 미저장 | 재부팅 시 Secure MAC 소실 |
| 배선 검증 전 Sticky 활성화 | 잘못된 MAC이 고정됨 |
| `aging time 60` | 60초가 아니라 60분 |
| `errdisable recovery interval 300` | 300분이 아니라 300초 |
| `errdisable recovery` 설정 | 원인 제거를 의미하지 않음 |
| Storm-control Upper만 지정 | Lower도 같은 값이므로 Flapping 가능 |
| `storm-control unicast` | 플랫폼에 따라 전체 Unicast에 적용될 수 있음 |
| `storm-control action shutdown` (Uplink) | Storm 발생 시 하위 Segment 전체 단절 |
| SPAN Destination Port | 일반 Forwarding·MAC 학습·STP 미참여 |
| Destination Port Ping 실패 | 장애가 아니라 정상 동작 |
| Port-security + SPAN Destination | 같은 Port에 병행 불가 |
| 같은 Session Number 재사용 | 기존 Session 설정이 대체됨 |
| RSPAN VLAN 생성만 함 | `remote-span` 미지정 시 동작하지 않음 |
| RSPAN VLAN Trunk 미허용 | 경로 중간에서 Traffic 소실 |
| RSPAN VLAN에 Access Port 할당 | MAC 학습이 없어 통신 불가 |
| VLAN SPAN과 Interface Source 혼용 | 같은 Session에서 불가 |
| SPAN Session 추가 | 플랫폼 Session 수 제한 초과 시 거부 |
| Native VLAN 한쪽만 변경 | Untagged Frame이 다른 VLAN으로 해석 |
| VLAN 분리 | Broadcast Domain 분리이며 완전한 보안 경계는 아님 |
| 미사용 Port 방치 | Shutdown·격리 VLAN 없이는 진입점으로 남음 |
| 격리 VLAN에 SVI 생성 | 격리 효과 상실 |
| PortFast를 Trunk에 적용 | Loop 위험 증가, End Device Port 전용 |
| PortFast 단독 사용 | BPDU Guard 없이는 비인가 Switch 차단 불가 |
| Packet Tracer 동작 성공 | SPAN Session 제약·Storm-control 단위가 실장비와 다를 수 있음 |

---

## 17. 🚑 빠른 장애 진단 순서

```properties
show interfaces status
    ↓
Err-disabled 여부 확인
    ↓
show logging으로 원인 분류 (psecure / storm-control / bpduguard)
    ↓
Violation Count 및 Last Source Address 확인
    ↓
Secure MAC 목록과 실제 장비 대조
    ↓
Storm-control Current 값과 임계치 비교
    ↓
SPAN Session Source·Destination·방향 확인
    ↓
RSPAN VLAN 일치 및 Trunk Allowed VLAN 확인
    ↓
Access VLAN 및 Trunk Native VLAN 확인
    ↓
SVI 및 L3 Gateway 상태 확인
```

명령 순서:

```cisco
show interfaces status
show interfaces status err-disabled
show logging
show port-security interface <INTERFACE>
show port-security address
show storm-control <INTERFACE>
show monitor session all
show vlan remote-span
show interfaces trunk
show ip interface brief
```

---

> 📌 **핵심 요약**
> - 활성화: `switchport port-security` (기본 Maximum 1, Violation shutdown)
> - 최대 MAC: `switchport port-security maximum <NUMBER>`
> - Static MAC: `switchport port-security mac-address <MAC-ADDRESS>`
> - Sticky: `switchport port-security mac-address sticky`
> - Violation: `switchport port-security violation protect|restrict|shutdown`
> - Aging: `switchport port-security aging time <MINUTES>` + `aging type absolute|inactivity`
> - Sticky 초기화: `clear port-security sticky interface <INTERFACE>`
> - 자동 복구: `errdisable recovery cause psecure-violation` + `errdisable recovery interval <SECONDS>`
> - Storm-control: `storm-control broadcast level <UPPER> <LOWER>`
> - Storm-control Action: `storm-control action trap|shutdown`
> - Local SPAN: `monitor session <NUMBER> source interface <INTERFACE> [both|rx|tx]` + `destination interface <INTERFACE>`
> - RSPAN: `vlan <VLAN-ID>` + `remote-span`, `destination remote vlan` / `source remote vlan`
> - 상태 확인: `show port-security`, `show storm-control`, `show monitor session all`
> - 미사용 Port: `shutdown` + 격리 VLAN + `nonegotiate`
> - 관련: 6-1. 🔐 Port-security & Storm-control · 6-2. 🔎 SPAN (Port Mirroring) · 6-3. 🧩 Switch 보안 통합 정리 · 6-4. 🚑 Switch 보안 트러블슈팅 치트시트 · 6-5. ⚡ Switch 보안 명령어 퀵 레퍼런스 · 3-3-2. 🧩 스위치 & 방어 보안 통합 정리
