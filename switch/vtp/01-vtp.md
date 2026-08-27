# VTP (VLAN Trunking Protocol)

> **Tag:** #network #switch #vtp #vlan #cisco
> **핵심 요약:** VLAN Database를 스위치 간 자동 동기화하는 Cisco 전용 프로토콜. Revision Number와 Mode 이해가 핵심

---

## 1. 핵심 기술 개념 (Concept)

### VTP가 실무에서 위험한 이유

VTP는 **Revision Number가 가장 높은 스위치의 VLAN DB를 전체에 강제 동기화**하기 때문에, 실수로 Revision이 높은 구형/타 도메인 스위치를 연결하면 **운영망 전체 VLAN이 순식간에 덮어써지거나 삭제**되는 대형 장애가 발생한다. 이 때문에 실무에서는 VTP를 Transparent로 두거나 아예 쓰지 않는 경우가 많다.

Revision Number는 VLAN을 생성/수정/삭제할 때마다 **1씩 증가**하며, 리셋하려면 VTP Domain을 바꿨다가 되돌리거나 Transparent 모드로 전환하면 0으로 초기화된다. 스위치를 재투입할 때는 **반드시 Revision을 0으로 리셋**하고 연결하는 것이 안전 수칙이다. VTP 동작을 위한 3대 조건은 ① 스위치 간 **Trunk 연결**, ② **VTP Domain 일치**, ③ **VTP Password 일치**이며, 하나라도 다르면 동기화되지 않는다.

### VTP Server / Client / Transparent 모드의 실무 운용 방식

VLAN을 관리할 소수의 스위치만 **Server**, 이를 받기만 하는 대부분의 스위치는 **Client**, VTP 도메인에서 독립적으로 로컬 VLAN을 쓰고 싶은 스위치는 **Transparent**로 둔다. Transparent는 VTP 광고를 **자기 DB엔 반영하지 않되 하위로 중계(pass-through)**만 한다.

Server/Client는 **Standard VLAN(1~1005)**만 다룰 수 있고, **Extended VLAN(1006~4094)**은 오직 **Transparent 모드**에서만 생성 가능하다. Transparent에서 만든 VLAN은 자기 DB에만 존재하므로, 상위 Server가 VLAN을 삭제해도 **Transparent 스위치엔 그대로 남아** 독립성이 보장된다.

### VTP Mode별 권한 매트릭스 (반드시 암기)

| Mode | 생성 | 수정 | 삭제 | 전파 | 일치 | 중계 |
|------|:----:|:----:|:----:|:----:|:----:|:----:|
| **Server** (기본) | O | O | O | O | O | O |
| **Client** | X | X | X | X | O | O |
| **Transparent** | O | O | O | X | X | O |

> - **전파:** 자신의 VLAN 변경을 Trunk 연결 스위치와 동기화하는 권한
> - **일치:** 수신한 VLAN 정보를 자신의 DB에 반영하는 권한
> - **중계:** 수신한 VLAN 정보를 다른 Trunk 포트로 넘겨주는 권한
> - ※ Revision Number 증가는 **전파 또는 일치 권한**이 있어야 발생 → Client도 일치 권한이 있어 Revision이 증가함

---

## 2. 표준 설정 템플릿 (Configuration)

### VTP 기본 3종 설정 (Domain / Password / Mode)

```cisco
! === Domain : 한 스위치에서 설정하면 Trunk를 통해 자동 전파됨 ===
Switch(config)# vtp domain GIT_NET

! === Password : 전파되지 않으므로 모든 스위치에 개별 설정 필수 ===
Switch(config)# vtp password cisco1234

! === Mode 지정 ===
Switch(config)# vtp mode server        ! VLAN 관리 주체 (기본값)
Switch(config)# vtp mode client        ! VLAN 수신 전용
Switch(config)# vtp mode transparent   ! 독립 운영 + 중계만
```

### 실전 시나리오 (SW1=Server, SW2/SW4=Client, SW3=Transparent)

**요구사항:** SW1/SW3만 VLAN 생성 가능 · SW2/SW4는 SW1의 VLAN만 공유 · SW3은 VLAN 공유 안 함 · Domain=GIT_NET, Password=cisco1234 · 관리주소 192.168.1.X/24

```cisco
! ================= SW1 (Server) =================
interface vlan 1
 no shutdown
 ip address 192.168.1.1 255.255.255.0     ! 스위치 원격관리용 관리 IP
!
interface gigabitethernet 0/0
 switchport trunk encapsulation dot1q      ! VTP는 Trunk 위에서만 동작
 switchport mode trunk
!
vtp domain GIT_NET
vtp password cisco1234
vtp mode server                            ! VLAN 생성/전파의 주체
!
vlan 10
 name VLAN_AA
vlan 20
 name VLAN_BB

! ================= SW2 (Client) =================
interface vlan 1
 no shutdown
 ip address 192.168.1.2 255.255.255.0
!
interface range gi0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp domain GIT_NET
vtp password cisco1234
vtp mode client                            ! VLAN 생성 불가, 수신/일치만

! ============== SW3 (Transparent) ==============
interface vlan 1
 no shutdown
 ip address 192.168.1.3 255.255.255.0
!
interface range gi0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp domain GIT_NET
vtp password cisco1234
vtp mode transparent                       ! 독립 DB 운영 + 하위로 중계만

! ================= SW4 (Client) =================
interface vlan 1
 no shutdown
 ip address 192.168.1.4 255.255.255.0
!
interface gigabitethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp domain GIT_NET
vtp password cisco1234
vtp mode client
```

### Extended VLAN(1006~4094) 생성 (Transparent 전용)

```cisco
! Server 모드에서는 Extended VLAN 생성 불가 → Transparent로 전환 필요
Switch(config)# vtp mode transparent
Switch(config)# vlan 3001                  ! Extended VLAN은 Transparent에서만 가능
!
! 주의: Extended VLAN이 존재하면 Server 모드로 되돌릴 수 없음
!       → 되돌리려면 반드시 Extended VLAN을 먼저 삭제해야 함
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Switch# show vtp status         ! Mode, Domain, Revision Number, VLAN 개수 확인 (최우선)
Switch# show vtp password       ! Password 일치 여부 확인
Switch# show vlan brief         ! 동기화된 VLAN 목록 확인
Switch# show interface trunk    ! VTP 전제조건인 Trunk 형성 여부 확인
Switch# ping 192.168.1.X        ! VLAN 1 관리망 소통 확인
```

### `show vtp status` 판독 핵심 포인트

- **VTP Operating Mode** : Server / Client / Transparent 확인
- **VTP Domain Name** : 모든 스위치가 동일해야 함 (한 글자만 달라도 동기화 실패)
- **Configuration Revision** : 동기화 기준값. Client/Server는 서로 같아야 정상
- **Number of existing VLANs** : 동기화된 VLAN 총 개수 비교

### 장애 시나리오 1: VLAN이 동기화되지 않음

- **증상:** SW1에서 VLAN을 만들었는데 SW2에 안 나타남.
- **원인 파악 (순서대로):**
    1. `show interface trunk` — **Trunk가 형성됐는지**부터 확인 (VTP는 Trunk 필수).
    2. `show vtp status` — 양쪽 **Domain Name 철자 일치** 여부 확인.
    3. `show vtp password` — **Password 일치** 여부 확인.
    4. SW2가 실수로 **Transparent 모드**면 VTP 광고를 자기 DB에 반영하지 않음.
- **해결:** 불일치 항목(Trunk/Domain/Password/Mode)을 맞춰준다.

### 장애 시나리오 2: 신규 스위치 투입 후 전체 VLAN이 사라짐 (가장 치명적)

- **증상:** 새로 넣은 스위치 때문에 운영망 VLAN이 통째로 삭제/변경됨.
- **원인 파악:** 신규 스위치의 **Revision Number가 기존 Server보다 높았음.** VTP는 높은 Revision을 "최신"으로 간주해 전체를 덮어씀. (Domain/Password가 우연히 일치했을 때 발생)
- **해결:**
    1. 스위치 투입 전 **반드시 Revision을 0으로 리셋** — `vtp mode transparent` 후 다시 원하는 모드로 전환하거나, Domain을 임의값으로 바꿨다 되돌린다.
    2. 투입 전 `show vtp status`로 Revision이 0인지 확인 후 연결.

### 장애 시나리오 3: Client에서 VLAN 삭제/생성이 거부됨

- **증상:** `no vlan 30` 실행 시 `VTP VLAN configuration not allowed when device is in CLIENT mode.` 출력.
- **원인 파악:** Client 모드는 VLAN 생성/수정/삭제 권한이 없음 (정상 동작).
- **해결:** VLAN 변경은 **Server 모드 스위치에서** 수행. 굳이 로컬에서 관리하려면 Transparent로 전환.

---

> 💡 **면접 한 줄 요약:** VTP의 본질은 편의성이지만, Revision Number 기반 자동 덮어쓰기 특성 때문에 운영 리스크가 커서 실무에서는 신규 장비 투입 시 Revision 리셋이 필수이며, 최근에는 VTP 대신 수동 VLAN 관리나 Transparent 운영을 선호한다.


---

]