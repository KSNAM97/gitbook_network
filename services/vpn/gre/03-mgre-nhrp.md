# 🌐 mGRE & NHRP (Multipoint GRE)

> **Tag:** #network #vpn #mgre #nhrp #dmvpn #hub-spoke #cisco
> **핵심 요약:** 하나의 터널로 다수 지사를 연결하는 멀티포인트 GRE. NHRP로 터널IP↔공인IP를 매핑 (DMVPN의 기반)

---

## 1. 🎯 핵심 기술 개념 (Concept)

### 지사가 많아질 때 일반 GRE 대신 mGRE를 쓰는 이유

일반 GRE는 P2P라서 지사 N개면 본사에 터널 N개를 각각 만들어야 하고, 터널마다 대역폭·설정 부담이 커진다. mGRE는 본사에 터널 1개만 두고 `tunnel mode gre multipoint`로 여러 지사를 동시에 수용해 확장성이 크게 향상된다.

mGRE의 기본 네트워크 타입은 NBMA(Non-Broadcast Multi-Access)다. 프레임릴레이처럼 브로드캐스트가 기본 안 되는 구조다. 지사 추가 시 본사(Hub) 설정 변경 없이 지사만 붙이면 되는 것이 DMVPN 확장성의 핵심이다.

### mGRE에서 NHRP가 필수인 이유

터널 하나에 여러 지사가 물려 있으면, 특정 목적지 터널 IP가 실제 어느 공인 IP(NBMA)로 가야 하는지 알 수 없다. NHRP는 이 "터널 IP ↔ 공인(NBMA) IP" 매핑을 등록·조회하는 프로토콜로, Hub가 NHRP Server, 지사가 NHRP Client가 된다.

동작 방식은 지사(Client)가 부팅 시 자신의 (터널IP, 공인IP)를 Hub(Server)에 등록하면 Hub는 이를 동적으로 학습한다. 지사 간 직접 통신 시에는 목적지 터널IP로 NHRP 조회 후 목적지 공인IP를 확인해 직접 전송한다(DMVPN Phase 2/3).

### mGRE + OSPF에서 인접이 1개만 맺히는 문제와 해결책

터널의 OSPF 네트워크 타입이 기본 point-to-point라 다수 인접을 못 맺는다. `ip ospf network point-to-multipoint`(또는 non-broadcast + DR 고정)로 변경해 멀티 인접을 형성해야 한다.

EIGRP를 쓸 경우, Hub는 한 지사에서 배운 경로를 같은 터널로 다시 광고해야 하므로 `no ip split-horizon eigrp`가 필요하다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 mGRE + NHRP (Hub-Spoke)

```cisco
! === Hub (NHRP Server) ===
Hub(config)# interface tunnel 123
Hub(config-if)# ip address 172.16.123.1 255.255.255.0
Hub(config-if)# tunnel source 121.160.14.1
Hub(config-if)# tunnel mode gre multipoint            ! mGRE 활성화 (destination 지정 안 함)
Hub(config-if)# ip nhrp network-id 1234               ! NHRP 도메인 식별자
Hub(config-if)# ip nhrp authentication SOLnet         ! NHRP 인증
Hub(config-if)# ip nhrp map multicast dynamic         ! 지사 등록을 동적으로 수용

! === Spoke (NHRP Client) ===
Spoke(config)# interface tunnel 123
Spoke(config-if)# ip address 172.16.123.2 255.255.255.0
Spoke(config-if)# tunnel source 121.160.25.2
Spoke(config-if)# tunnel mode gre multipoint
Spoke(config-if)# ip nhrp network-id 1234
Spoke(config-if)# ip nhrp authentication SOLnet
Spoke(config-if)# ip nhrp nhs 172.16.123.1            ! NHRP 서버(Hub) 터널 IP
Spoke(config-if)# ip nhrp map multicast 121.160.14.1  ! Hub 공인 IP로 멀티캐스트 매핑
Spoke(config-if)# ip nhrp map 172.16.123.1 121.160.14.1  ! Hub 터널IP↔공인IP 정적 매핑
Copy
````

### 📌 라우팅 프로토콜 연동 (터널 타입 조정)

```cisco
! === EIGRP : Hub에서 split-horizon 해제 ===
Hub(config)# interface tunnel 123
Hub(config-if)# no ip split-horizon eigrp 100         ! 배운 경로를 같은 터널로 재광고 허용
!
! === OSPF : 다중 인접을 위해 network type 변경 ===
Router(config)# interface tunnel 123
Router(config-if)# ip ospf network point-to-multipoint  ! P2P → 멀티포인트
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
Router# show ip nhrp                     ! NHRP 매핑 테이블 (터널IP↔NBMA)
Router# show ip nhrp brief               ! NHRP 매핑 요약
Hub#    show ip nhrp dynamic             ! Hub가 동적 학습한 지사 목록
Router# show ip route                     ! 터널 경로 + 사설망 경로 확인
Router# show ip ospf neighbor             ! 멀티 인접 형성 확인
```

### 📊 `show ip nhrp` 판독

- **static** : 관리자가 수동 매핑한 항목 (지사→Hub)
- **dynamic** : Hub가 지사로부터 자동 학습한 항목 (Hub에서 확인)
- **NBMA address** : 해당 터널 IP에 대응하는 실제 공인 IP

### 🚨 장애 시나리오 1: 지사가 NHRP 등록이 안 됨

- **원인 파악:**
    1. **network-id 또는 authentication 불일치** — Hub/지사가 동일해야 함.
    2. 지사에 `ip nhrp nhs`(Hub 터널IP) 또는 Hub 매핑 누락.
    3. Hub에 `ip nhrp map multicast dynamic` 누락.
- **해결:** network-id/인증 일치 확인 → 지사의 nhs/map 설정 확인 → `show ip nhrp dynamic`(Hub)에 지사가 등록됐는지 확인.

### 🚨 장애 시나리오 2: OSPF 인접이 1개만 맺힘 (다수 지사인데)

- **원인 파악:** 터널 OSPF 타입이 기본 **point-to-point**라 멀티 인접 불가.
- **해결:** 모든 터널에 `ip ospf network point-to-multipoint` 적용. (또는 non-broadcast + Hub를 DR로 고정, 지사 priority 0)

### 🚨 장애 시나리오 3: EIGRP에서 지사끼리 경로를 못 배움

- **원인 파악:** Hub에서 **split-horizon**이 켜져 있어, 한 지사에서 배운 경로를 같은 터널의 다른 지사로 재광고하지 못함.
- **해결:** Hub 터널에 `no ip split-horizon eigrp [AS]` 설정.

> 💡 **면접 한 줄 요약:** mGRE는 터널 1개로 다수 지사를 수용하는 멀티포인트 구조이고, NHRP가 터널IP↔공인IP 매핑을 담당하며, 라우팅 연동 시 EIGRP는 split-horizon 해제, OSPF는 network type 변경이 필수다. 이 조합이 곧 DMVPN의 토대다.

c, eigrp neighbor

```text
ISP-1# show ip route
C    211.241.14.0/24 is directly connected, FastEthernet0/0
C    198.210.12.0/24 is directly connected, Serial1/0.12
C    198.210.13.0/24 is directly connected, Serial1/0.13
      110.0.0.0/24 is subnetted, 4 subnets
C        110.110.1.0 is directly connected, Loopback0
O        110.110.3.0 [110/65] via 198.210.13.3, 00:00:30, Serial1/0.13
C        110.110.10.0 is directly connected, FastEthernet0/1
O    198.210.23.0/24 [110/128] via 198.210.13.3, 00:00:30, Serial1/0.13
                           [110/128] via 198.210.12.2, 00:00:30, Serial1/0.12

ISP-2# show ip route
C    211.241.25.0/24 is directly connected, FastEthernet0/0
C    198.210.12.0/24 is directly connected, Serial1/0.12
O    198.210.13.0/24 [110/128] via 198.210.23.3, 00:00:40, Serial1/0.23
                           [110/128] via 198.210.12.1, 00:00:40, Serial1/0.12
      110.0.0.0/24 is subnetted, 4 subnets
O        110.110.3.0 [110/65] via 198.210.23.3, 00:00:40, Serial1/0.23
C        110.110.20.0 is directly connected, FastEthernet0/1
C    198.210.23.0/24 is directly connected, Serial1/0.23

ISP-3# show ip route
C    198.210.13.0/24 is directly connected, Serial1/0.13
C    211.241.36.0/24 is directly connected, FastEthernet0/0
      110.0.0.0/24 is subnetted, 4 subnets
O        110.110.1.0 [110/65] via 198.210.13.1, 00:00:54, Serial1/0.13
O        110.110.2.0 [110/65] via 198.210.23.2, 00:00:54, Serial1/0.23
C        110.110.3.0 is directly connected, Loopback0
C        110.110.3.0 is directly connected, Loopback0
C        110.110.30.0 is directly connected, FastEthernet0/1
C    198.210.23.0/24 is directly connected, Serial1/0.23

ISP-1# show ip route eigrp 
D    220.220.1.0/24 [90/409600] via 211.241.14.1, 00:00:02, FastEthernet0/0
D    192.168.1.0/24 [90/307200] via 211.241.14.1, 00:00:03, FastEthernet0/0	<--- 서울 본사 사설 네트워크 확인
ISP-1# ping 192.168.1.1
```

> `2) MGRE (GIT_Tunnel).txt`

```cisco

EX) GIT-HQ , GIT-A , GIT-B에서 OSPF routing protocol을 구성하시오
 .Process = 626 , Area = 123 , Router-ID = 개별설정
 .GIT-HQ , GIT-A , GIT-B의 Loopback 0 네트워크와 사설 네트워크는 OSPF 에 포함되어야한다.
 .OSPF 라우팅 업데이트가 필요한 Interface로만 OSPF  Packet이 송신되어야 한다.
 .OSPF에 포함된 모든 네트워크는 Interface에 할당된 Subnetmask로 확인되어야 한다.

   # GIT-HQ (R4)
router ospf 626
 router-id 1.1.1.1
 passive-interface default
 network 192.168.1.0  0.0.0.255 area 123

EX2) GIT-HQ , GIT-A , GIT-B에서 직접 연결된 ISP로 Default-route를 생성해야한다.

-MGRE를 구성해도 Next-hop이 연결되지 않는다.
# GIT-HQ의 Routing Table에 GIT-A와 ISP-2의 연결 구간 네트워크 정보가 확인되지 않는다.
# GIT-HQ의 Routing Table에 GIT-B와 ISP-3의 연결 구간 네트워크 정보가 확인되지 않는다.

# GIT-A의 Routing Table에 GIT-HQ와 ISP-1의 연결 구간 네트워크 정보가 확인되지 않는다.
# GIT-A의 Routing Table에 GIT-B와 ISP-3의 연결 구간 네트워크 정보가 확인되지 않는다.

# GIT-B의 Routing Table에 GIT-HQ와 ISP-1의 연결 구간 네트워크 정보가 확인되지 않는다.
# GIT-B의 Routing Table에 GIT-A와 ISP-2의 연결 구간 네트워크 정보가 확인되지 않는다.

   # GIT-HQ  (R4)
ip route 0.0.0.0 0.0.0.0 211.241.14.11
!

   # GIT-A  (R5)

GIT-HQ# show ip route
C    220.220.1.0/24 is directly connected, Loopback0
C    211.241.14.0/24 is directly connected, FastEthernet0/0
C    192.168.1.0/24 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 211.241.14.11		<------- Default-route 확인
S*   0.0.0.0/0 [1/0] via 211.241.14.11		<------- Default-route 확인
GIT-HQ# ping 211.241.25.2
GIT-HQ# ping 211.241.36.3

```

> `1) MGRE (ISP_Tunnel).txt`

```cisco
router eigrp 456
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 211.241.14.0
 network 192.168.1.0
router eigrp 456
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 211.241.14.0
 network 110.110.10.0  0.0.0.255
router eigrp 456
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 110.110.20.0  0.0.0.255
 network 211.241.25.0
router eigrp 456
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 220.220.2.0
 network 211.241.25.0
router eigrp 456
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 110.110.30.0  0.0.0.255
 network 211.241.36.0
router eigrp 456
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 220.220.3.0
 network 211.241.36.0
```

> `2) MGRE (GIT_Tunnel).txt`

```cisco
  Process ID 626, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 11111
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:02
  Supports Link-local Signaling (LLS)
GIT-HQ# show ip ospf neighbor 
Neighbor ID     Pri   State           	Dead Time     Address    	Interface
2.2.2.2              0   FULL/  -        	00:01:59        172.16.100.2	Tunnel123
3.3.3.3              0   FULL/  -        	00:01:52        172.16.100.3	Tunnel123

GIT-A# show ip ospf neighbor 
Neighbor ID     Pri   State           	Dead Time     Address         	Interface
1.1.1.1              0   FULL/  -        	00:01:55        172.16.100.1	Tunnel123

GIT-B# show ip ospf neighbor 
GIT-B# show ip ospf neighbor 
Neighbor ID     Pri   State           	Dead Time     Address         	Interface
1.1.1.1              0   FULL/  -        	00:01:55        172.16.100.1	Tunnel123

-GIT-HQ , GIT-A , GIT-B의 Routing Table에 각 사설 IP 주소가 모두 확인되어야 한다.
  Process ID 626, Router ID 1.1.1.1, Network Type NON_BROADCAST, Cost: 11111	<---- Network-type = NON_BROADCAST
  Transmit Delay is 1 sec, State WAITING, Priority 1
  No designated router on this network
  No backup designated router on this network
  Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5
    ~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
GIT-HQ# show ip ospf neighbor 
Neighbor ID     Pri   State           	Dead Time     Address    	Interface
2.2.2.2              0   FULL/DROTHER	00:01:59        172.16.100.2	Tunnel123
3.3.3.3              0   FULL/DROTHER	00:01:52        172.16.100.3	Tunnel123

GIT-A# show ip ospf neighbor 
Neighbor ID     Pri   State           	Dead Time     Address         	Interface
1.1.1.1              0   FULL/DR        	00:01:55        172.16.100.1	Tunnel123

```

---

> 관련: 5-3-1-1. 🚇 GRE Static · 5-3-1-2. 🔄 GRE Dynamic