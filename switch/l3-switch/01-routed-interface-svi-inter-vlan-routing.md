# 🏢 Multi-Layer Switch (L3 Switch)

> **Tag:** #network #l3switch #svi #routed-interface #inter-vlan #ospf #cisco
> **핵심 요약:** L2/L3 기능을 겸비한 스위치로 하드웨어 기반 고속 라우팅과 VLAN 간 통신을 담당

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Router-on-a-Stick 대신 L3 스위치를 선택하는 이유

L3 스위치는 라우팅을 **소프트웨어가 아닌 ASIC(하드웨어)에서 처리**하므로, 라우터의 서브인터페이스 방식(단일 물리 링크 병목)과 달리 **와이어스피드로 VLAN 간 라우팅**을 수행한다. 캠퍼스망의 코어/디스트리뷰션 계층에서 대량의 Inter-VLAN 트래픽을 감당하는 표준이다.

L3 스위치를 라우팅 장비로 쓰려면 반드시 전역에서 `ip routing`을 켜야 한다. 이게 빠지면 SVI에 IP를 줘도 라우팅이 동작하지 않는다. 라우터 온 어 스틱은 단일 트렁크 링크에 모든 VLAN 트래픽이 몰려 병목이 생기지만, L3 스위치는 내부 백플레인으로 처리해 병목이 없다.

### Routed Interface와 SVI의 사용 구분

**Routed Interface**는 물리 포트 하나를 라우터 포트처럼 만들어 **스위치 간 P2P 라우팅 구간(업링크)** 에 쓴다. **SVI**는 VLAN에 IP를 부여하는 가상 인터페이스로, **여러 단말이 속한 VLAN의 Default Gateway** 역할에 쓴다.

Routed Interface(`no switchport`)는 VLAN에 속하지 않고 포트 자체에 IP를 가지므로 하나의 세그먼트(P2P)에 적합하다. SVI(`interface vlan X`)는 하나의 VLAN에 연결된 다수 PC의 게이트웨이를 담당하며 단말이 많은 액세스 VLAN에 적합하다. L2 스위치는 관리용으로 **VLAN 1 SVI 하나만** 쓰지만, L3(Multi-Layer) 스위치는 **다수의 SVI**를 게이트웨이로 운용한다.

### 📊 Routed Interface vs SVI 비교

| 구분 | Routed Interface | SVI |
|------|------------------|-----|
| 설정 | 물리포트에 `no switchport` | `interface vlan X` |
| 소속 | VLAN에 속하지 않음 | 특정 VLAN에 종속 |
| 용도 | 스위치 간 P2P 라우팅 링크 | VLAN Default Gateway |
| 대상 | 단일 세그먼트 | VLAN 내 다수 단말 |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Routed Interface 구성 (물리 포트를 L3로)

```cisco
! === L3 스위치 라우팅 활성화 (필수 선행) ===
SW1(config)# ip routing

! === 물리 포트를 Routed Interface로 전환 ===
SW1(config)# interface fastethernet 0/24
SW1(config-if)# no switchport                  ! L2 → L3 포트로 전환 (VLAN 소속 해제)
SW1(config-if)# ip address 13.13.12.1 255.255.255.0
SW1(config-if)# no shutdown

! === Loopback (라우팅 테스트/식별용) ===
SW1(config)# interface loopback 0
SW1(config-if)# ip address 13.13.10.1 255.255.255.0
```

### 📌 SVI 구성 (VLAN Gateway)

```cisco
SW1(config)# ip routing                        ! Inter-VLAN 라우팅을 위해 필수
!
SW1(config)# interface vlan 11
SW1(config-if)# ip address 13.13.11.1 255.255.255.0   ! VLAN 11 단말의 Gateway
SW1(config-if)# no shutdown
!
SW1(config)# interface vlan 12
SW1(config-if)# ip address 13.13.12.1 255.255.255.0
SW1(config-if)# no shutdown

! === 단말 포트를 해당 VLAN에 Access 할당 (게이트웨이 통신 전제) ===
SW1(config)# interface range fa0/1 - 2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 11      ! PC의 Gateway = int vlan 11
```

### 📌 동적 라우팅 연동 (EIGRP 예시)

```cisco
SW2(config)# ip routing
!
SW2(config)# router eigrp 100
SW2(config-router)# no auto-summary            ! 자동 요약 비활성 (VLSM 환경 필수)
SW2(config-router)# network 13.13.12.0 0.0.0.255
SW2(config-router)# network 13.13.23.0 0.0.0.255
SW2(config-router)# network 13.13.22.0 0.0.0.255
```

### 📌 동적 라우팅 연동 (OSPF + Passive-interface)

```cisco
SW1(config)# router ospf 1
SW1(config-router)# router-id 1.1.1.1
SW1(config-router)# passive-interface default              ! 모든 IF에서 Hello 차단 (기본 봉쇄)
SW1(config-router)# no passive-interface vlan 12           ! 실제 인접 맺을 IF만 개방
SW1(config-router)# network 13.13.1.0 0.0.0.255 area 0
SW1(config-router)# network 13.13.11.0 0.0.0.255 area 0
SW1(config-router)# network 13.13.12.0 0.0.0.255 area 0
```

> 💡 `passive-interface default` 후 필요한 인터페이스만 `no passive-interface`로 여는 것이 **보안·안정성 관점의 실무 정석**입니다. 단말만 있는 VLAN(예: vlan 11)으로 불필요한 OSPF Hello를 뿌리지 않아 공격 표면과 CPU 낭비를 줄입니다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
Switch# show ip route                     ! 라우팅 테이블 (C=직접연결, O=OSPF, D=EIGRP)
Switch# show ip interface brief           ! 각 IF의 IP/상태 요약
Switch# show ip ospf neighbor             ! OSPF 인접성(FULL) 확인
Switch# show ip ospf interface brief      ! IF별 OSPF 상태/Cost/DR-BDR
Switch# show run | section router ospf    ! 라우팅 프로토콜 설정 확인
Switch# show vlan brief                    ! 단말 포트의 VLAN 할당 확인
```

### 🚨 장애 시나리오 1: SVI에 IP를 줬는데 VLAN 간 통신 불가

- **증상:** `interface vlan X`에 IP를 설정했으나 라우팅이 안 됨.
- **원인 파악 (순서대로):**
    1. **`ip routing`이 빠짐** — L3 스위치의 라우팅 스위치가 꺼져 있으면 SVI는 관리용으로만 동작.
    2. SVI가 `down` 상태 — 해당 VLAN에 **활성 Access 포트가 하나도 없거나** 트렁크로 VLAN이 안 넘어오면 SVI가 up되지 않음.
    3. 단말이 여전히 **VLAN 1(기본)** 에 있는데 게이트웨이는 VLAN 11 — 포트를 `switchport access vlan 11`로 할당해야 함.
- **해결:** `ip routing` 확인 → 해당 VLAN에 up 상태 포트 확보 → 단말 포트 VLAN 재할당.

### 🚨 장애 시나리오 2: 같은 스위치의 두 VLAN PC는 되는데 원격 스위치 VLAN과 통신 불가

- **증상:** SW1 내부 PC → SW1 게이트웨이(O), 그러나 SW3 내부망(13.13.14.0/24)으로 Ping 실패.
- **원인 파악:** SW1의 라우팅 테이블에 **원격 네트워크 정보가 없음.** SVI로 로컬 게이트웨이는 만들었으나, 스위치 간 **동적 라우팅(OSPF/EIGRP)이 미구성**되어 서로의 대역을 학습하지 못함.
- **해결:** 스위치 간 라우팅 프로토콜을 구성하고 `show ip route`로 원격 네트워크(O/D 표시)가 학습됐는지 확인. `show ip ospf neighbor`가 **FULL**인지 점검.

### 🚨 장애 시나리오 3: OSPF 인접(Neighbor)이 안 맺힘

- **원인 파악:**
    1. `passive-interface default`는 걸었는데 정작 **인접 맺을 인터페이스를 `no passive-interface`로 열지 않음** (가장 흔함).
    2. 양쪽 서브넷/마스크 불일치, Area 번호 불일치, Hello/Dead 타이머 불일치.
- **해결:** `show ip ospf interface brief`로 해당 IF가 passive가 아닌지 확인하고, 양쪽 network 문/Area를 일치시킴.

> 💡 **면접 한 줄 요약:** L3 스위치의 핵심은 `ip routing` 활성화 + SVI를 각 VLAN 게이트웨이로 두고 + 스위치 간은 Routed Interface 또는 SVI로 라우팅 프로토콜을 돌려 전체 네트워크를 하드웨어 속도로 수렴시키는 것이다.


---

]