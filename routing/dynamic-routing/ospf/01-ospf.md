# OSPF (Open Shortest Path First)

> **Tag:** #Network #Cisco #OSPF #Routing #IGP #LinkState
> **핵심 요약:** **Link-State 기반의 표준 IGP** — Dijkstra(SPF) 알고리즘으로 Loop-Free 경로를 계산하며, Area 기반 계층 설계로 대규모 엔터프라이즈/데이터센터 환경에서 사실상 표준으로 자리잡은 라우팅 프로토콜.

---

## 1. 핵심 기술 개념 (Concept)

### 엔터프라이즈 코어에서 OSPF를 선택하는 이유

OSPF는 **개방형 표준(RFC 2328)** 이면서 **Area 기반 계층 설계**로 LSDB 부하와 SPF 계산 범위를 통제할 수 있어, 멀티벤더 환경에서 확장성과 예측 가능성을 동시에 확보할 수 있기 때문이다.

Cisco 종속인 EIGRP와 달리 OSPF는 Cisco/Juniper/Arista/Huawei가 혼재된 실무 환경에서 유일하게 신뢰 가능한 IGP 선택지다. M&A나 벤더 교체 시 리스크가 없다는 점도 중요하다. 또한 Area 0(Backbone) + 위성 Area 구조로 특정 Area 내부의 링크 플랩(Flap)이 전체 도메인에 SPF 재계산을 유발하지 않도록 격리된다. LSA Type 3/4/5 필터링과 Stub/NSSA를 조합해 LSDB 크기를 세밀하게 제어할 수 있다.

### NBMA 구간 OSPF 인접성 문제의 원인과 해결책

NBMA는 Broadcast/Multicast가 물리적으로 전달되지 않아 224.0.0.5 Hello가 Spoke에 도달하지 않기 때문에 인접성이 맺어지지 않는다. **Network Type을 point-to-multipoint로 변경**하거나 Hub에서 `neighbor` 명령으로 명시적 유니캐스트 지정하는 것이 표준 해법이다.

DR/BDR이 필요 없는 Hub-Spoke 3대 이상 구성에서는 `ip ospf network point-to-multipoint`를 적용하고, DR을 반드시 Hub로 강제해야 하는 경우에는 Hub에 `priority 255` + Spoke에 `priority 0` + `neighbor` 명령을 조합한다. 추가로 Broadcast(10/40)와 NBMA(30/120)의 Hello/Dead Timer가 서로 다르므로 양단 Network Type을 반드시 동일하게 맞춰야 한다. `show ip ospf interface`의 Hello/Dead 값이 불일치하면 인접성은 절대 맺어지지 않는다.

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. OSPF 기본 구성 (Single Area / Multi-Area)

```cisco
! ========================================
! [R1] Backbone Router - Area 0
! ========================================
router ospf 100
 router-id 1.1.1.1                                    ! 명시적 Router-ID 지정 (Loopback IP 사용 관례, 재부팅 후에도 고정)
 passive-interface default                            ! 서버/LAN 세그먼트 Hello 차단 (보안 표면 최소화 - 실무 필수)
 no passive-interface Serial1/0                       ! WAN 인접성 구간만 명시적 활성화
 network 13.13.12.1 0.0.0.0 area 0                    ! Wildcard로 인터페이스 단위 정밀 매칭 (Classful 금지)
 network 172.16.1.1 0.0.0.0 area 0                    ! Loopback도 개별 지정
 auto-cost reference-bandwidth 10000                  ! 10G 기준 Cost 재계산 (GE/10GE 혼재 시 필수)
!
interface Loopback172
 ip address 172.16.1.1 255.255.255.0
 ip ospf network point-to-point                       ! /32 Host 광고 방지 → 원래 SubnetMask로 광고
!
interface Serial1/0
 ip ospf authentication message-digest                ! WAN 구간 MD5 인증 (라우팅 스푸핑 방지)
 ip ospf message-digest-key 100 md5 <STRONG_KEY>      ! 실무에서는 반드시 강력한 Key 사용
 ip ospf hello-interval 10                            ! Hello 주기 (양단 동일 필수)
 ip ospf dead-interval 40                             ! Dead 주기 (Hello × 4 관례)
!

! ========================================
! [R2] ABR - Area 0 + Area 13
! ========================================
router ospf 100
 router-id 2.2.2.2
 passive-interface default
 no passive-interface Serial1/1                       ! Area 0 인접성
 no passive-interface FastEthernet0/1                 ! Area 13 인접성
 network 13.13.12.2 0.0.0.0 area 0                    ! Backbone 구간
 network 13.13.23.2 0.0.0.0 area 13                   ! 위성 Area
 network 172.16.2.2 0.0.0.0 area 0
 area 13 range 100.100.0.0 255.255.248.0              ! ABR Internal 요약 → LSA Type-3 축약 (LSDB 축소)
 area 0 authentication message-digest                 ! Area 전체 인증 강제 (개별 인터페이스 인증 대신 일괄 적용)
!

! ========================================
! [R3] ASBR - 외부 재분배 담당
! ========================================
router ospf 100
 router-id 3.3.3.3
 redistribute eigrp 100 metric-type 1 subnets         ! Type-1: ASBR까지 누적 Metric (경로 다중화 환경)
 summary-address 200.200.8.0 255.255.248.0            ! ASBR External 요약 → LSA Type-5 축약
!
```
### 2-2. NBMA (Frame-Relay) 구간 OSPF - Hub & Spoke
```cisco
! ========================================
! [Hub Router] - DR 강제
! ========================================
interface Serial1/0.123 multipoint
 ip address 13.13.9.2 255.255.255.0
 ip ospf priority 255                                 ! Hub를 DR로 강제 (최고 우선순위)
 frame-relay map ip 13.13.9.1 201 broadcast           ! broadcast 키워드 필수 (Multicast Hello 전달)
 frame-relay map ip 13.13.9.3 203 broadcast
!
router ospf 1
 neighbor 13.13.9.1                                   ! NBMA는 Multicast 안 되므로 유니캐스트 Hello 강제
 neighbor 13.13.9.3

! ========================================
! [Spoke Router]
! ========================================
interface Serial1/0.123 multipoint
 ip ospf priority 0                                   ! Spoke는 DR 후보에서 제외 (Hub 강제 선출용)
!
! [대안] DR 아예 사용 안 함 → 3대 이상 Hub-Spoke 권장 방식
interface Serial1/0.123 multipoint
 ip ospf network point-to-multipoint                  ! DR 선출 X, 논리적 P2P 링크 집합으로 처리
!
```
### 2-3. Stub / Totally Stub / NSSA 요약
```cisco
! [일반 Stub] LSA Type 4,5 차단 + Default 자동 생성 (ASBR 없는 Area)
router ospf 100
 area 14 stub                                         ! Area 내 모든 라우터에 동일 설정 필수

! [Totally Stub] LSA Type 3,4,5 차단 (ABR에만 no-summary)
router ospf 100
 area 14 stub no-summary                              ! ABR에서만 no-summary, 내부 라우터는 stub만

! [NSSA] ASBR이 존재하는 Area의 Stub 대체 (Type-7 재분배 허용)
router ospf 100
 area 14 nssa default-information-originate           ! NSSA는 Default를 수동 생성해야 함

! [Totally NSSA] 강화된 NSSA
router ospf 100
 area 14 nssa no-summary                              ! Default 자동 생성 + Type 3,4,5 차단
```
### 2-4. Virtual-Link (Backbone 단절 시 응급 조치)
```cisco
! Area 13이 Area 0과 물리적으로 단절된 경우
! [R2 - ABR] Transit Area 13을 통과하여 R3까지 논리적 Area 0 연장
router ospf 100
 area 13 virtual-link 3.3.3.3 message-digest-key 100 md5 <KEY>   ! 인증도 함께 설정

! [R3 - 반대편 ABR]
router ospf 100
 area 13 virtual-link 2.2.2.2 message-digest-key 100 md5 <KEY>

! ⚠️ 실무 원칙: Virtual-Link는 "설계 실패의 임시 조치"이며, 영구 운영 구조가 되면 안 됨
```
## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

|명령어|확인 목적|
|---|---|
|`show ip ospf neighbor`|인접성 상태(FULL/2WAY) 및 DR/BDR 확인 — 가장 먼저 봐야 할 명령|
|`show ip ospf interface <intf>`|Area, Network Type, Hello/Dead, Cost, Authentication 확인|
|`show ip ospf database`|LSDB 전체 확인 (Type 1~5, 7) — 광고/수신 여부 판단|
|`show ip ospf database router`|Type-1 상세 — 각 라우터의 링크 정보 및 Cost|
|`show ip route ospf`|O / O IA / O E1 / O E2 / O N1 / O N2 구분 확인|
|`show ip ospf border-routers`|ABR/ASBR 경로 확인|
|`show ip protocols`|프로세스 ID, Router-ID, Passive-interface 목록|
|`debug ip ospf adj`|인접성 수립 State 흐름 추적 (Down→Init→2WAY→Exstart→Exchange→Loading→Full)|
|`debug ip ospf packet`|인증 방식(aut:0/1/2), 패킷 송수신 확인|
|`clear ip ospf process`|Router-ID 변경 등 재적용 (⚠️ 운영 중 실행 금지)|

### 3-2. State 진행 흐름 (Neighbor 안 맺힐 때 어디서 멈추는지가 힌트)
```cisco
Down → Init → 2Way → Exstart → Exchange → Loading → Full
                ↑        ↑          ↑          ↑
             Hello    DBD 교환   MTU 불일치   LSR/LSU
             수신     Master/Slave 결정    문제 발생 지점

```
### 3-3. 대표 장애 시나리오 & 트러블슈팅

#### 시나리오 A. **인접성이 2WAY에서 멈추고 FULL로 진행되지 않음** (Multi-access 환경)

- **원인:** Broadcast/NBMA 세그먼트에서 DR/BDR이 아닌 DROTHER끼리는 정상적으로 2WAY 상태에서 멈춤 (정상 동작). 문제는 **DR과의 관계도 2WAY에서 멈추는 경우**.
- **1단계 - Priority 확인:** `show ip ospf interface fa0/0` → 해당 세그먼트의 모든 라우터가 `priority 0`으로 설정되어 DR/BDR 후보가 없는 상황인지 확인.
- **2단계 - Network Type 확인:** Hub-Spoke NBMA인데 Spoke끼리 인접성을 요구하는 잘못된 설계인지 검토. 정상 동작이면 `point-to-multipoint`로 변경.
- **3단계 - 해결:** 최소 1대에는 `priority 1` 이상 부여하거나, Network Type을 P2P/P2MP로 전환.

#### 시나리오 B. **인접성이 Exstart/Exchange에서 반복적으로 Flap** (MTU Mismatch)

- **원인:** 양단 인터페이스의 IP MTU가 다르면 DBD 교환 단계에서 실패. 특히 GRE/DMVPN/MPLS 구간에서 자주 발생.
- **1단계 - 로그 확인:** `debug ip ospf adj` → `Nbr X.X.X.X has larger interface MTU` 메시지 확인.
- **2단계 - MTU 비교:** 양단 `show interface <intf>` 로 MTU 값 비교.
- **3단계 - 해결 옵션:**
    - 근본 해결: 양단 MTU를 동일하게 재설정 (`ip mtu 1500`).
    - 임시 우회: `ip ospf mtu-ignore` (⚠️ Fragmentation 리스크로 임시 조치 한정, 문서화 필수).

#### 시나리오 C. **인접성은 FULL인데 특정 경로가 Routing Table에 없음**

- **원인 1 - Area 불일치:** 양단이 다른 Area 번호로 설정됨 → 인접성은 맺어져도 LSA 종류가 어긋남.
- **원인 2 - Stub 설정으로 필터링:** Area가 Stub/Totally Stub로 설정되어 O E1/E2가 차단됨.
- **1단계:** `show ip ospf database external` → 광고 자체가 안 되는지, 수신은 하는데 설치가 안 되는지 판별.
- **2단계:** `show ip ospf | include Area` → Area의 Stub/NSSA 상태 확인.
- **3단계 - 해결:** Area 설계 재검토. External 경로가 필요하면 Stub 해제 또는 NSSA로 전환.

### 3-4. 실무 Best Practice 체크리스트

- ✅ **`network` 명령은 반드시 Wildcard `0.0.0.0`으로 인터페이스 단위 매칭** — Classful 매칭 시 의도치 않은 인터페이스까지 OSPF 활성화되어 보안/성능 이슈 발생.
- ✅ **`passive-interface default` 선언 후 필요한 WAN 링크만 `no passive-interface`** — 서버 세그먼트에서 Hello 트래픽 및 인접성 공격 표면 최소화.
- ✅ **모든 WAN/Backbone 구간은 MD5 이상 인증 필수** — Plain-text 인증은 감사에서 감점 요인.
- ✅ **Router-ID는 Loopback IP로 명시적 지정** — 물리 인터페이스 변경/셧다운 시 Router-ID 변동 방지.
- ✅ **`auto-cost reference-bandwidth`는 도메인 전체에 동일하게 설정** — 한 대만 다르면 경로 계산 왜곡, SPF 이상 동작.
- ✅ **Area 요약(`area range`, `summary-address`)은 LSDB 축소가 아닌 SPF 안정성 관점에서 접근** — 링크 플랩 격리가 본질.
- ✅ **Virtual-Link는 응급 조치일 뿐, 영구 설계에 포함시키지 말 것** — 반드시 물리적 Backbone 연결로 재설계.
- ✅ **Loopback은 `ip ospf network point-to-point`로 원래 SubnetMask 광고** — /32 Host 광고로 인한 경로 관리 혼란 방지.

```cisco
interface fastethernet 0/0
 no shutdown
 ip address 198.210.10.30  255.255.255.224
!
interface serial 1/0
 no shutdown
interface fastethernet 0/0
 no shutdown
 ip address 100.10.1.254  255.255.255.0
!
interface serial 1/3
 no shutdown
interface fastethernet 0/0
 no shutdown
 ip address 200.20.2.254  255.255.255.0
!
interface serial 1/1
 no shutdown
interface fastethernet 0/0
 no shutdown
 ip address 210.116.41.57  255.255.255.252
!
interface serial 1/1
 no shutdown
interface serial 1/0
 no shutdown
 encapsulation hdlc
 bandwidth 64
 ip address 210.116.41.37  255.255.255.252
!
interface fastethernet 0/0
 no shutdown
 ip address  210.116.41.65  255.255.255.248
 ip ospf priority 0
!
interface fastethernet 0/1
interface fastethernet 0/1
 no shutdown
 ip address 210.116.41.58  255.255.255.252
!
```


---

]