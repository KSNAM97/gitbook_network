# 🚀 RSTP & MSTP (고속 수렴 & 초대형망 STP)

> **Tag:** #network #rstp #mstp #802.1w #802.1s #convergence
> **핵심 요약:** PVST의 느린 수렴을 해결한 RSTP, 128개 STP 한계를 넘는 MSTP

---

## 1. 🎯 핵심 기술 개념 (Concept)

### RSTP(802.1w)가 기존 PVST보다 빠른 이유

RSTP는 Listening/Learning 대기를 없애고, **Proposal/Agreement 핸드셰이크**로 인접 스위치와 즉시 합의하여 Forwarding으로 전환한다. 타이머(Forward-delay) 대신 **명시적 메시지 교환**으로 수렴하기 때문에 초 단위가 아닌 **거의 즉시** 수렴한다.

Proposal은 "내가 DP로 동작하겠다"는 제안(Bridge-ID 낮은 쪽이 송신)이고, Agreement는 그에 대한 응답이다. RSTP는 PVST의 5상태를 **Discarding / Learning / Forwarding**으로 단순화하고, Alternate/Backup 포트로 백업 경로를 미리 준비해 장애 시 즉시 전환한다.

### VLAN이 128개를 초과할 때 MSTP가 필요한 이유

PVST는 **VLAN당 STP 인스턴스 1개**를 돌려 최대 128개까지만 지원하고 CPU 부하가 크다. MSTP(802.1s)는 **동일 경로를 쓰는 여러 VLAN을 하나의 Instance(MST)로 묶어** 인스턴스 수를 대폭 줄여 초대형망에서 효율적이다.

MSTP는 CST(단일 STP, 부하 적음)와 PVST(유연함)의 장점을 결합한다. STP Mode를 MST로 바꾸면 기본 동작은 **RSTP**가 된다. **MST0(IST)** 는 별도 매핑하지 않은 모든 VLAN을 관리하는 기본 인스턴스다.

### 📊 PVST vs RSTP Port 상태 매핑

| PVST | RSTP |
|------|------|
| Disable | Discarding |
| Listening | Discarding |
| Learning | Learning |
| Forwarding | Forwarding |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 RSTP (Rapid-PVST) 구성

```cisco
! === 모든 스위치를 RSTP 모드로 전환 ===
SW1(config)# spanning-tree mode rapid-pvst
SW2(config)# spanning-tree mode rapid-pvst
SW3(config)# spanning-tree mode rapid-pvst
!
! Root/Backup 지정은 PVST와 동일
SW2(config)# spanning-tree vlan 1 priority 4096     ! Root-bridge
SW3(config)# spanning-tree vlan 1 priority 16384    ! Backup Root
```
### 📌 MST Region 구성 (VLAN-to-Instance 매핑)

```cisco
! === MST Region 설정 (모든 스위치에서 이름·Revision·매핑이 완전히 동일해야 함) ===
SW1(config)# spanning-tree mode mst
SW1(config)# spanning-tree mst configuration
SW1(config-mst)# name CAMPUS_REGION
SW1(config-mst)# revision 1
SW1(config-mst)# instance 1 vlan 10,20,30      ! VLAN 10/20/30 → MST Instance 1
SW1(config-mst)# instance 2 vlan 40,50,60      ! VLAN 40/50/60 → MST Instance 2
SW1(config-mst)# end
!
! Root Bridge 지정 (Instance별)
SW1(config)# spanning-tree mst 1 priority 4096   ! Instance 1 Root
SW2(config)# spanning-tree mst 2 priority 4096   ! Instance 2 Root
```
### 📌 MST Cost 값 (PVST와 다름 - 주의)

```properties
# MST는 별도의 Long-path Cost 체계 사용
10M   = 2,000,000
100M  = 200,000
1G    = 20,000
10G   = 2,000
100G  = 200
1T    = 20
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
Switch# show spanning-tree                        ! RSTP 동작 및 Port Role 확인
Switch# show spanning-tree mst 0                   ! IST(기본 인스턴스) 확인
Switch# show spanning-tree mst 1                   ! 특정 Instance별 Root/Block 확인
Switch# show spanning-tree mst configuration       ! Region 이름/VLAN 매핑 확인
Switch# debug spanning-tree events                 ! Proposal/Agreement 로그 확인
```

### 🚨 장애 시나리오 1: RSTP인데 수렴이 느림 (PVST처럼 동작)

- **증상:** RSTP 설정 후에도 30초 지연 발생.
- **원인 파악:** 상대 스위치가 **PVST(802.1D)** 로 동작 중이면, RSTP는 호환을 위해 해당 포트를 **레거시 STP로 fallback**함.
- **해결:** 모든 스위치를 `spanning-tree mode rapid-pvst`로 통일. `show spanning-tree`에서 protocol이 rstp인지 확인.

### 🚨 장애 시나리오 2: MSTP 스위치 간 VLAN 매핑이 동기화 안 됨

- **증상:** 일부 스위치가 다른 Region으로 인식되어 MST0(IST)로만 통신.
- **원인 파악:** MSTP는 **Region Name + Revision + VLAN-to-Instance 매핑**이 **완전히 일치**해야 같은 Region으로 묶임. 한 글자라도 다르면 별도 Region.
- **해결:** `show spanning-tree mst configuration`으로 세 값을 모든 스위치에서 비교해 완전히 동일하게 맞춤.

> 💡 **면접 한 줄 요약:** 현대 캠퍼스망은 대부분 **RSTP(Rapid-PVST)** 를 기본으로 쓰고, VLAN 수백 개 이상의 대규모 환경에서만 **MSTP**로 인스턴스를 통합해 CPU 부하를 관리한다.


---

]