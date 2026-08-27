# EIGRP (Enhanced Interior Gateway Routing Protocol)

>**Tags:** #EIGRP #Routing #IGP #Cisco #DUAL #HybridProtocol
> **핵심 요약:** **Category:** Routing Protocol / Fundamentals

---

## 1. 핵심 기술 개념 (Concept)

### Q1. OSPF라는 표준 프로토콜이 있는데, 왜 실무에서 여전히 EIGRP를 채택하나요? 각각의 아키텍처적 강점은 무엇인가요?

- 👉 **핵심 답변:** EIGRP는 **DUAL 알고리즘 기반의 "Hybrid 프로토콜"** 로, Link-State의 빠른 수렴 속도와 Distance-Vector의 낮은 리소스 사용량을 모두 갖췄기 때문입니다. 특히 **Feasible Successor(사전 계산된 백업 경로)** 덕분에 장애 발생 시 SPF 재계산 없이 즉시 Failover가 가능해, Cisco 단일 벤더 환경의 대규모 캠퍼스/WAN에서 여전히 강력합니다.

- **꼬리 질문 대비 설명:**
  - **OSPF vs EIGRP 선택 기준:** 멀티벤더 환경이면 OSPF, Cisco 단일 벤더면 EIGRP가 유리합니다. EIGRP는 2013년 RFC 7868로 공개되었지만 실무 구현은 여전히 Cisco 중심입니다.
  - **Metric 산정 방식:** OSPF는 Bandwidth만 보지만, EIGRP는 **Bandwidth + Delay(기본)** 조합으로 계산해 더 세밀한 경로 선택이 가능합니다.
  - **부분 업데이트:** 링크 변화 시 전체 토폴로지가 아닌 **변화된 경로만** 업데이트해 대역폭 소모가 적습니다.

### Q2. EIGRP의 3가지 테이블(Neighbor / Topology / Routing)은 왜 분리되어 있고, 각각 어떤 실무적 의미를 가지나요?

- 👉 **핵심 답변:** 이 3단계 분리 구조가 바로 **EIGRP의 빠른 수렴을 가능하게 하는 핵심 아키텍처**입니다. Topology Table에 이미 모든 대체 경로(FS)를 사전 계산해 두었기 때문에, 장애 발생 시 Routing Table 재계산 없이 즉시 백업 경로로 전환됩니다.

- **꼬리 질문 대비 설명:**
  - **Neighbor Table:** Hello로 형성된 인접성 관리 → `show ip eigrp neighbor`
  - **Topology Table:** 모든 가능한 경로(Successor + Feasible Successor) 저장 → `show ip eigrp topology`
  - **Routing Table:** Topology Table에서 최적 경로(Successor)만 선택 → `show ip route eigrp`
  - **SIA(Stuck-In-Active) 문제:** FS가 없을 때만 Query를 확산시키므로, 대규모 망에서는 **주소 요약/Stub 설정**으로 Query 범위를 제한하는 것이 필수입니다.

---

## 2. 표준 설정 템플릿 (Configuration)

### 기본 EIGRP 활성화 (실무 표준)

```cisco
! === 기본 EIGRP 설정 (모든 라우터 공통 템플릿) ===
!
router eigrp 100
 !
 ! [필수] 자동 요약 비활성화 - VLSM 환경에서 오작동 방지
 no auto-summary
 !
 ! [권장] Passive-interface default로 모든 인터페이스 차단 후,
 ! EIGRP가 필요한 인터페이스만 명시적으로 활성화 (Best Practice)
 passive-interface default
 no passive-interface serial 1/1
 no passive-interface fastethernet 0/0
 !
 ! [필수] EIGRP를 광고할 네트워크를 Wildcard로 정밀 지정
 ! - Classful network (예: network 13.0.0.0)는 의도치 않은 인터페이스까지 광고됨
 ! - Host 단위(0.0.0.0)로 지정하면 인터페이스 하나만 정확히 지정 가능
 network 13.13.1.1    0.0.0.0
 network 13.13.10.1   0.0.0.0
 network 150.1.13.1   0.0.0.0
!
```
### Frame-Relay Multipoint 환경 (Split-Horizon 이슈)
```cisco
! === Hub 라우터 (R2) - NBMA 환경 필수 설정 ===
!
interface serial 1/0.123 multipoint
 !
 ! NBMA(Frame-Relay Multipoint)에서 Spoke 간 통신을 위해 반드시 비활성화
 ! - 기본적으로 EIGRP는 수신 인터페이스로 재광고하지 않음 (Split-Horizon)
 ! - Hub-and-Spoke에서는 이 규칙이 Spoke 간 라우팅을 막는 원인이 됨
 no ip split-horizon eigrp 100
!

```
### EIGRP 인증 설정 (보안 필수)
```cisco
! === MD5 인증 (양단 동일하게 설정) ===
!
key chain EIGRP_AUTH
 key 20240101
  key-string CCNA1234
!
interface serial 1/0
 !
 ! MD5 해시 방식으로 EIGRP 패킷 인증
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_AUTH
!
```
### Hello/Hold Timer 조정 (수렴 속도 튜닝)
```cisco
! === NBMA 구간에서 Hello 주기 단축 ===
!
interface serial 1/0.123
 !
 ! 기본값: NBMA는 Hello 60초 / Hold 180초 → 장애 감지 지연 심각
 ! LAN 수준(5/15초)으로 낮춰 빠른 수렴 확보
 ip hello-interval eigrp 100 5
 ip hold-time     eigrp 100 15
!

```
### 주요 파라미터 정리

|항목|값|설명|
|---|---|---|
|**Protocol Number**|88|IP 헤더에서 EIGRP 식별|
|**Multicast Address**|224.0.0.10|Hello/Update 전송용|
|**AD (Internal)**|90|신뢰도 (RIP 120, OSPF 110보다 우선)|
|**AD (Summary)**|5|요약 경로 신뢰도|
|**AD (External)**|170|재분배된 경로|
|**Metric 공식**|(10⁷/BW + Delay) × 256|K1=1, K3=1 기본|
|**Hello / Hold (LAN)**|5초 / 15초|Ethernet, PPP, HDLC|
|**Hello / Hold (NBMA)**|60초 / 180초|Frame-Relay Multipoint|
|**최대 Load Balancing**|4 (기본) / 최대 16|`maximum-paths` 조정|

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어
```cisco
! [1] 인접성(Neighbor) 형성 확인 - 가장 먼저 볼 것
R1# show ip eigrp neighbor
!  → Uptime이 계속 리셋되면 링크 불안정 or 인증/K값 불일치 의심
!
! [2] EIGRP Process 전반 상태 확인
R1# show ip protocols
!  → K값, Auto-summary, Passive-interface, Router-ID 확인
!
! [3] Topology Table - Successor/FS 존재 여부 확인
R1# show ip eigrp topology
R1# show ip eigrp topology [network]/[prefix]
!  → "1 successors" 만 있고 FS가 없다면 장애 시 Query 확산 위험
!
! [4] 실제 라우팅 테이블 반영 확인
R1# show ip route eigrp
!
! [5] 인터페이스별 EIGRP 상태 (Hello 주기, 인증 등)
R1# show ip eigrp interfaces detail
!
! [6] 실시간 패킷 분석 (트러블슈팅 시)
R1# debug eigrp packet hello
R1# debug eigrp packet update ack
R1# debug eigrp packet query reply
```
### 대표 장애 시나리오 & 트러블슈팅

#### 시나리오 1. Neighbor 인접성이 형성되지 않음

|단계|원인 분석|해결 방법|
|---|---|---|
|①|**AS 번호 불일치**|양단 `router eigrp [번호]` 동일한지 확인|
|②|**K값 불일치**|`show ip protocols`로 K1~K5 값 확인 (기본 1,0,1,0,0)|
|③|**Passive-interface 설정**|해당 인터페이스가 Passive로 잡혀 Hello 미송신|
|④|**인증 불일치**|Key ID, Key-string, 인증 모드 대소문자 정확히 확인|
|⑤|**Subnet 불일치**|양단 IP가 같은 서브넷에 속해있는지 확인|
|⑥|**ACL 차단**|인터페이스 ACL이 EIGRP(Protocol 88) 또는 224.0.0.10을 차단하는지|

#### 시나리오 2. SIA (Stuck-In-Active) 발생으로 인접성 단절
```cisco
! 증상 로그
! %DUAL-3-SIA: Route x.x.x.x/24 stuck-in-active state in IP-EIGRP(0) 100
! %DUAL-5-NBRCHANGE: Neighbor x.x.x.x is down: stuck in active
!
! 원인: FS가 없는 경로에서 Query가 대규모로 확산되어 3분 내 Reply 미수신
!
! === 해결책 1: 주소 요약으로 Query 범위 축소 (가장 근본적) ===
interface fastethernet 0/0
 ip summary-address eigrp 100 13.13.0.0 255.255.0.0
!
!
! === 해결책 2: Stub 설정 (Spoke 라우터에 적용) ===
router eigrp 100
 eigrp stub connected summary
!  → Hub는 Stub에게 Query를 보내지 않아 확산 차단
!
!
! === 해결책 3: Active Timer 증가 (임시방편) ===
router eigrp 100
 timers active-time 3   ! 기본 3분 → 필요 시 증가
!
```
#### 시나리오 3. Frame-Relay에서 Spoke 간 통신 불가
```cisco
! 원인: Hub의 Multipoint 인터페이스에서 Split-Horizon이 활성화되어
!       Spoke A에서 온 광고를 Spoke B로 재광고하지 않음
!
! === Hub 라우터에서 해결 ===
interface serial 1/0.123 multipoint
 no ip split-horizon eigrp 100
!
! 검증
R2# show ip eigrp interfaces detail serial 1/0.123
!  → "Split-horizon is disabled" 확인
```
### 실무 Best Practice

- **`network` 명령어는 Wildcard로 정밀하게** — Classful로 넣으면 원하지 않는 인터페이스까지 EIGRP가 활성화되어 보안/성능 이슈 발생
- **`passive-interface default` 후 필요한 곳만 해제** — 서버 세그먼트 등 EIGRP 불필요 구간의 Hello 트래픽 및 공격 표면 최소화
- **모든 WAN 구간은 인증 필수** — MD5 인증으로 라우팅 스푸핑 방지
- **주소 요약은 Query 범위 제한의 핵심** — 단순 테이블 축소가 아닌 **SIA 예방책**으로 접근
- **Hello/Hold Timer 튜닝은 양단 동시에** — 한쪽만 변경 시 인접성 단절 위험 (엄밀히는 Hold만 맞추면 되지만 관례상 동일하게)

```cisco
     121.0.0.0/24 is subnetted, 8 subnets
D       121.160.9.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.8.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.11.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.10.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.13.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.13.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.12.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.15.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0
D       121.160.14.0 [90/2349056] via 150.3.13.3, 00:00:05, FastEthernet0/0

R5# show ip route eigrp
D    197.110.9.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.8.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.11.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.10.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.13.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.13.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.12.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.15.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
D    197.110.14.0/24 [90/2349056] via 150.3.13.3, 00:03:51, FastEthernet0/0
     13.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
D       13.13.0.0/16 [90/409600] via 150.3.13.3, 00:00:03, FastEthernet0/0	<---- 하나의 요약된 네트워크로 확인
     121.0.0.0/21 is subnetted, 1 subnets
D       121.160.8.0 [90/2349056] via 150.3.13.3, 00:00:03, FastEthernet0/0	<---- 하나의 요약된 네트워크로 확인
     150.1.0.0/24 is subnetted, 1 subnets
D       150.1.13.0 [90/2221056] via 150.3.13.3, 00:03:51, FastEthernet0/0

C       150.3.13.0 is directly connected, FastEthernet0/0
D*   0.0.0.0/0 [90/409600] via 150.3.13.3, 00:00:06, FastEthernet0/0	<---- EIGRP Default-route

R5# ping 13.13.1.1  source  13.13.5.5
R5# ping 13.13.2.2  source  13.13.5.5
D       13.13.14.0/24 [90/2221056] via 13.13.10.1, 00:16:16, Serial1/1		<----- 최적 경로 = Serial1/1
D       13.13.15.0/24 [90/307200] via 150.3.13.5, 01:29:38, FastEthernet0/0
      ~~~~~~~~ 중간 생략 ~~~~~~~~

```

> `EIGRP 실습.txt`

```cisco
EX1) 아래의 조건에 맞게 EIGRP를 구성하시오
 .EIGRP Routing protocol을 사용하여 모든 네트워크간 통신되도록 설정하시오
 .AS = 100 , 자동 요약 기능은 사용하지 않는다.
 .EIGRP 라우팅 업데이트가 필요한 Interface로만 EIGRP Packet이 송신되어야 한다.

EX2-1) 아래의 조건에 맞게 설정하시오
 .R4에 Loopback 121을 생성한 후 121.160.8.0/24 ~ 121.160.15.0/24 대역의 IP 주소를 할당해야한다.
 .R4는 Loopback 121에 포함된 네트워크를 EIGRP로 라우팅 업데이트해야한다. (단 "network" command는 한줄만 사용)

   # R4

EX2-2) 아래의 조건에 맞게 설정하시오
 # R3은  R5에게 '13.13.0.0/16'의 범위로 수동요약하여 업데이트가 가능해야한다.

   # R3
interface fastethernet 0/0

EX2-3) 아래의 조건에 맞게 설정하시오
 # R3은  R5에게 '121.160.8.0/24 ~ 121.160.15.0/24' 네트워크를 하나의 요약하여 라우팅 해야한다.

-121.160.00001 000.00000000

EX2) 아래의 조건에 맞게 Offset-list를 구성하시오
 .R4의 13.13.4.0/24 네트워크가 R5의 13.13.5.0/24  네트워크로 통신시 R2를 경유하여 통신하도록 설정하시오
 .R4의 150.1.13.0/24 네트워크가 R5의 150.3.13.0/24  네트워크로 통신시 R2를 경유하여 통신하도록 설정하시오

    # R1

EX5) 아래의 조건에 맞게 설정하시오
 .R1 , R2 , R3 Frame-relay 구간에서 Hello는 5초마다 교환하며 15초안에 Hello가 수신되지 않으면 인접성이 단절되어야 한다.
 .R1 , R3 Frame-relay 구간에서 Hello는 5초마다 교환하며 15초안에 Hello가 수신되지 않으면 인접성이 단절되어야 한다.

   # R1 , R3
IP routing table maximum-paths is 16
Route Source    	Networks    Subnets     Overhead    Memory (bytes)
connected       	14    	   8	     1584 	       2992
static          	0        	   0      	     0     	       0
eigrp 100       	8        	   19   	     1944 	       3672    <--- Memory 소모량 확인
```

---

> 관련: 2-2. 🐎 EIGRP 통합 정리 · 2-3. 🚑 EIGRP 트러블슈팅 치트시트 · 2-4. ⚡ EIGRP 명령어 퀵 레퍼런스