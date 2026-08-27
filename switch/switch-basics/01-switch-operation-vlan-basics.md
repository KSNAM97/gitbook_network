# 스위치 기본 & VLAN (Switch Basics & VLAN)

> **Tag:** #network #switch #layer2 #vlan #cisco
> **핵심 요약:** 계층별 장비의 역할, 스위치 동작원리(L.F.F.F), VLAN을 통한 논리적 분할

---

## 1. 핵심 기술 개념 (Concept)

### 스위치의 Collision Domain 분리만으로 VLAN이 필요한 이유

스위치는 포트별로 Collision Domain은 분리하지만 **Broadcast Domain은 하나로 공유**하기 때문에, 호스트가 늘어날수록 ARP/DHCP 등 브로드캐스트 트래픽이 전체로 확산되어 성능을 갉아먹는다. VLAN은 물리적 배선을 건드리지 않고 **브로드캐스트 도메인을 논리적으로 쪼개는** 유일한 L2 수단이다.

기본적으로 모든 포트는 `VLAN 1`에 속하므로, VLAN 설계를 하지 않으면 스위치 24포트 전체가 하나의 거대한 방송망이 된다. 이것이 대규모 환경에서 BUM(Broadcast, Unknown-unicast, Multicast) 트래픽 폭증의 근본 원인이다. VLAN은 물리적 위치와 무관하게 그룹핑이 가능하므로 "3층에 있는 서버와 5층에 있는 관리자 PC를 같은 논리망"으로 묶는 유연한 설계가 가능하다.

### 스위치의 Unknown Unicast Flooding 동작과 실무적 위험

목적지 MAC이 MAC Table에 없는 **Unknown Unicast**, 그리고 **Broadcast/Multicast** 프레임은 수신 포트를 제외한 모든 포트로 **Flooding**된다. 이 Flooding은 Loop 환경과 결합되면 Broadcast Storm으로 직결되는 트리거가 된다.

MAC Table의 기본 Aging Time은 **300초(5분)**이며, 이 시간 동안 트래픽이 없으면 엔트리가 삭제되어 이후 통신 시 다시 Flooding이 발생한다. 잦은 MAC 이동/삭제는 대규모 Flooding을 유발할 수 있다. 스위치의 핵심 5대 동작은 **Learning(출발지 MAC 학습) → Flooding(모를 때 전파) → Forwarding(알 때 해당 포트로만) → Filtering(같은 포트면 차단) → Aging(오래된 정보 삭제)**이다.

---

## 2. 표준 설정 템플릿 (Configuration)

### 계층별 장비 요약 (면접 단골)

| 구분 | 계층 | 주소 체계 | 도메인 처리 | 기반 |
|------|------|-----------|-------------|------|
| **Hub** | L1 | 전기 신호 | Collision 공유 | - |
| **Switch** | L2 | MAC (48bit) | Collision 분할 / Broadcast 공유 | 하드웨어(ASIC) |
| **Router** | L3 | IP (32bit) | Broadcast 분할 | 소프트웨어 |

### VLAN 생성 / 수정 / 삭제

```cisco
! === 단일 VLAN 생성 ===
Switch(config)# vlan 10

! === 다중 VLAN 개별 생성 (콤마) ===
Switch(config)# vlan 10,20,30

! === 다중 VLAN 범위 생성 (하이픈) ===
Switch(config)# vlan 11-30

! === VLAN에 이름(Name) 부여 : 운영 시 식별성 확보를 위해 필수 ===
Switch(config)# vlan 11
Switch(config-vlan)# name CISCO-CCNA    ! 용도가 드러나도록 명명 권장
!
Switch(config)# vlan 13
Switch(config-vlan)# name LINUX-Server

! === VLAN 삭제 (개별/다중/범위 동일 문법에 no) ===
Switch(config)# no vlan 10
Switch(config)# no vlan 10,20,30
Switch(config)# no vlan 11-30
Copy
````

### Access Mode 포트 할당 (단말 장비용)

```cisco
! PC, Server 등 VLAN을 모르는 단말이 붙는 포트에 적용
Switch(config)# interface fastethernet 0/1
Switch(config-if)# switchport mode access          ! 모드를 Access로 고정 (DTP 협상 방지)
Switch(config-if)# switchport access vlan 10        ! 해당 포트를 VLAN 10에 소속
```

### Range 명령어 (다수 포트 일괄 설정 = 운영 효율의 핵심)

```cisco
Switch(config)# vlan 10,20,30,40
!
! 연속 범위 : 하이픈(-) 사용
Switch(config)# interface range fa0/1 - 8
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
!
! 비연속 포트 : 콤마(,) 사용 (콤마 앞뒤 공백 주의)
Switch(config)# interface range fa0/13, fa0/15, fa0/17, fa0/19
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 20
!
! 범위 + 비연속 혼합 사용 가능
Switch(config)# interface range fa0/9 - 12, fa0/21 - 24
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 40
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Switch# show vlan brief          ! VLAN 목록 + 각 VLAN에 소속된 포트 확인 (가장 많이 사용)
Switch# show mac address-table   ! 학습된 MAC과 포트 매핑 확인
Switch# show interface status    ! 포트의 VLAN/Duplex/Speed 상태 한눈에 확인
Switch# show running-config interface fa0/1   ! 특정 포트 설정 검증
```

### 장애 시나리오 1: 같은 VLAN 인데 통신이 안 됨

- **증상:** VLAN 10에 할당한 두 PC 간 Ping 실패.
- **원인 파악:**
    1. `show vlan brief`로 두 포트가 **정말 VLAN 10에 있는지** 확인. (오타로 다른 VLAN에 들어간 경우 다발)
    2. 할당하려던 VLAN이 실제로 **생성되지 않은** 경우, 포트는 inactive 상태가 됨.
    3. 두 PC가 다른 스위치에 있다면 스위치 간 **Trunk 구성 여부** 확인.
- **해결:** 누락된 VLAN을 `vlan 10`으로 생성하고, `switchport access vlan 10`을 재적용.

### 장애 시나리오 2: VLAN을 삭제했더니 포트가 먹통

- **증상:** `no vlan 10` 실행 후 해당 VLAN에 속했던 포트들이 통신 불가.
- **원인 파악:** VLAN을 삭제해도 포트의 `switchport access vlan 10` 설정은 남아있어, 존재하지 않는 VLAN을 가리키며 **inactive**가 됨. (`show vlan brief`에서 해당 포트가 사라짐)
- **해결:** VLAN을 다시 생성하거나, 포트를 다른 유효한 VLAN(`switchport access vlan 1` 등)으로 재할당.


---

]