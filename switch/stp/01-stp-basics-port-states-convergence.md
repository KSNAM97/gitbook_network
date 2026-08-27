# 🌳 STP 기본 원리 (Spanning-Tree Protocol - PVST)

> **Tag:** #network #stp #pvst #layer2 #loop-prevention #cisco
> **핵심 요약:** L2 이중화 시 필연적으로 발생하는 Loop를 차단하는 알고리즘. Root-bridge 선출과 Port 역할이 핵심

---

## 1. 🎯 핵심 기술 개념 (Concept)

### L2 Loop가 라우팅 루프보다 치명적인 이유

L2 프레임에는 **TTL이 없기 때문**이다. L3 패킷은 TTL이 0이 되면 폐기되지만, 이더넷 프레임은 Loop에 갇히면 무한 순환하며 **Broadcast Storm, 중복 프레임 수신, MAC Table 불안정(플래핑)** 을 유발해 순식간에 스위치 CPU와 대역폭을 마비시킨다.

STP는 물리적 이중화 링크를 유지하되, 일부 포트를 **논리적으로 Blocking**하여 트리(Tree) 구조를 만들어 Loop를 끊는다. 장애 시 Block 포트를 활성화해 자동 복구한다. PVST(IEEE 802.1D 기반 Cisco)는 **VLAN 1개당 STP 인스턴스 1개**를 돌려 VLAN별 트래픽 로드밸런싱이 가능하다.

### Root-bridge와 Port 역할 선출 기준

모든 결정의 기준은 **BPDU** 안의 값들이다. ① **Bridge-ID(Priority + MAC)** 가 가장 낮은 스위치가 Root-bridge, ② 각 스위치에서 Root로 가는 최단경로(최저 Cost) 포트가 **RP(Root Port)**, ③ 세그먼트당 BPDU를 내보내는 포트가 **DP(Designated Port)**, ④ 나머지는 **AP(Blocking)** 이다.

Priority 기본값은 **32768**이며 실제로는 `32768 + VLAN번호`로 계산된다(sys-id-ext). Priority가 같으면 **MAC 주소가 낮은 쪽**이 Root가 된다. Blocking 포트 선출 기준(순서): ① **Cost가 큰 포트**, ② Sender **Bridge-ID가 큰 포트**, ③ **Port-priority가 큰 포트**(Root-bridge의 Port-priority 기준).

### 📌 BPDU 핵심 필드 & Cost 값

| 항목 | 값 |
|------|-----|
| Bridge-ID | Priority(32768) + MAC |
| Cost | 10M=100, 100M=19, 1G=4, 10G=2 |
| Port-priority | 128.x (x=포트번호) |
| Hello-time | 2초 |
| Max-age | 20초 |
| Forward-delay | 15초 |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Root-bridge / Backup Root-bridge 지정

```cisco
! === 방법1: Priority 직접 지정 (반드시 4096의 배수) ===
SW1(config)# spanning-tree vlan 1 priority 4096      ! 낮을수록 우선 → Root-bridge
SW2(config)# spanning-tree vlan 1 priority 16384     ! 두번째로 낮게 → Backup Root

! === 방법2: 매크로 명령 (권장, 자동 계산) ===
SW1(config)# spanning-tree vlan 1 root primary        ! Root-bridge 자동 지정
SW2(config)# spanning-tree vlan 1 root secondary      ! Backup Root 자동 지정
```

### 📌 Cost를 이용한 Block Port 변경 (BPDU 수신 스위치에서 설정)

```cisco
! Cost는 BPDU 수신 시 더해지므로, 경로를 바꾸려면 '수신하는 쪽'에서 조정
! VLAN 11,12는 Fa0/23 우선, 장애 시 Fa0/24로 전환되도록 구성
SW2(config)# interface fastethernet 0/23
SW2(config-if)# spanning-tree vlan 11,12 cost 100     ! 이 포트 Cost를 높여 Block 유도
```

### 📌 Port-priority를 이용한 Block Port 변경 (BPDU 송신 포트에서 설정)

```cisco
! Port-priority는 BPDU를 '송신하는' 상위(Root쪽) 포트에서 설정 (16의 배수)
SW1(config)# interface fastethernet 0/23
SW1(config-if)# spanning-tree vlan 13,14 port-priority 192   ! 값이 크면 상대 포트가 Block
```

### 📌 STP 타이머 조정 (Convergence 단축)

```cisco
Switch(config)# spanning-tree vlan 1 hello-time 2       ! 기본 2초
Switch(config)# spanning-tree vlan 1 forward-time 15    ! 기본 15초 (Listening+Learning 각 단계)
Switch(config)# spanning-tree vlan 1 max-age 20         ! 기본 20초
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 📌 STP 5가지 Port 상태 (반드시 암기)

|상태|BPDU 수신|MAC 학습|데이터 전송|소요시간|
|---|---|---|---|---|
|**Disable**|X|X|X|shutdown/No Cable|
|**Blocking**|O|X|X|Loop 차단 유지|
|**Listening**|O|X|X|15초 (Loop 감지)|
|**Learning**|O|O|X|15초 (MAC 학습)|
|**Forwarding**|O|O|O|통신 가능|

### ✅ 필수 검증 명령어

```cisco
Switch# show spanning-tree               ! 전체 STP 상태 (Root/Cost/Port역할)
Switch# show spanning-tree vlan 1        ! 특정 VLAN의 STP 상세
Switch# show spanning-tree root          ! Root-bridge 정보만 요약
Switch# debug spanning-tree events       ! 실시간 상태 전환 추적 (실습 필수)
```

### 🚨 장애 시나리오 1: Root-bridge ↔ Non Root-bridge 구간 장애 (총 30초)

- **동작 과정:** RP 경로 장애 발생 → Block 포트가 `Listening(15초) → Learning(15초) → Forwarding` 으로 전환.
- **debug 로그 흐름:**
    
    ```
    new root port Fa0/20, cost 38    ! 새 RP 선정
    Fa0/20 -> listening              ! Loop 체크
    Fa0/20 -> learning               ! MAC 학습
    Fa0/20 -> forwarding             ! 통신 재개
    ```
    

### 🚨 장애 시나리오 2: Root-bridge ↔ Backup Root-bridge 구간 장애 (총 50초, 더 느림)

- **동작 과정:** Block 포트가 **Max-age(20초) 대기** → `Listening(15초) → Learning(15초) → Forwarding`.
- **원인 파악:** Block 포트가 상위 BPDU를 계속 듣다가(`heard root...`), Max-age가 만료되어야 다음 단계로 진행하기 때문에 30초가 아닌 **50초**가 소요됨.
- **해결/개선:** 이 50초 지연을 줄이기 위해 **BackboneFast** 또는 **RSTP** 도입 (별도 문서 참조).

> 💡 **실무 포인트:** Root-bridge는 반드시 **관리자가 수동 지정**해야 한다. 기본값(MAC 최저)에 맡기면 오래된 저사양 스위치가 Root가 되어 백본이 성능 병목이 되는 사고가 흔하다.


---

]