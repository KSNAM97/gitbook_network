# BGP Next-hop 문제 해결 (Static · Redistribute · Next-hop-self)

> **Tag:** #Network #Cisco #BGP #Routing #NextHop #DMZ #NextHopSelf #Redistribution #Static
> **핵심 요약:** BGP는 **동일 AS 내에서 Next-hop을 변경하지 않는다.** eBGP Peer 구간이 DMZ(IGP 미구성)이면 AS 내부 Router는 외부 AS로 가는 Next-hop IP를 알 수 없어 경로가 Routing Table에 등록되지 않는다. **① Static Route ② DMZ 구간 IGP 재분배 ③ `next-hop-self`** 3가지 방법으로 해결한다.

---

## 1. 핵심 기술 개념 (Concept)

### BGP Next-hop 문제란

- 동일한 AS 내의 BGP Router들은 수신한 네트워크의 Next-hop 정보가 **변경되지 않는다.** (서로 다른 AS 간에만 Next-hop이 변경된다.)
- 이 문제는 **eBGP Peer 연결 구간이 DMZ인 환경**에서 발생한다. (DMZ 구간 = IGP가 구성되지 않는 구간)
- **해결 방법 3가지**
  1. Static Route를 사용
  2. DMZ 구간의 네트워크를 IGP로 재분배
  3. BGP `next-hop-self`

### 왜 DMZ 구간에서는 Loopback eBGP Peer가 불가능한가

- eBGP 연결 구간이 DMZ가 되면 상대 Router의 Loopback 0 네트워크가 Routing Table에서 확인되지 않는다.
- 따라서 R1 ↔ R2 구간을 DMZ로 구성하게 되면 R1의 Routing Table에는 R2의 Loopback 0 네트워크 정보가, R2의 Routing Table에는 R1의 Loopback 0 네트워크 정보가 확인되지 않기 때문에 **Loopback 기반 eBGP Peer가 단절**된다.
- 따라서 eBGP 연결을 **물리적인 Interface**로 변경해야 한다.

### 문제 발생 흐름

```text
[R1 AS100] --- DMZ(13.13.12.0/24) --- [R2 AS200 / 64512] --- [R3] --- [R4] --- [R5]
                                          │
   R2: Next-hop 13.13.12.1 (Connected → 통신 가능)
   R3~R5: 동일 AS 내이므로 Next-hop 13.13.12.1 그대로 유지
          → 13.13.12.0/24 경로가 Routing Table에 없음 → 통신 불가 → BGP 경로 미등록
```

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. DMZ 구간 구성 (사전 작업)

BGP AS 100과 BGP AS 200 구간에 DMZ를 구성하기 위해 EIGRP를 삭제 및 수정한다.

```cisco
! === R1 ===
router eigrp 100
 no network 13.13.12.1  0.0.0.0
!

! === R2 ===
router eigrp 100
 no network 13.13.12.2  0.0.0.0
!
```

eBGP 연결을 Loopback 기반에서 물리적 Interface 기반으로 변경한다.

```cisco
! === R1 ===
router bgp 100
 no neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.12.2 remote-as 200
!

! === R2 ===
router bgp 64512
 no neighbor 13.13.1.1 remote-as 100
 neighbor 13.13.12.1 remote-as 100
!
```

**정보 확인**

```cisco
R1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.12.2      4   200          17          18           4     0      0  00:03:08           0

R2# show ip bgp summary
Neighbor        V     AS  MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.3.3        4 64512	61         67         10     0       0 00:59:23           0
13.13.12.1      4   100      	18         18         10     0       0 00:04:03           3
```

---

### 2-2. BGP Next-hop 문제 확인

```cisco
R2# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.12.1                     0             	0 100 i	<--- 해당 네트워크로 통신하기위한 Next-hop = 13.13.12.1
*> 172.16.1.0/24    	13.13.12.1                     0             	0 100 i	<--- 해당 네트워크로 통신하기위한 Next-hop = 13.13.12.1
*> 192.168.1.0      	13.13.12.1                     0             	0 100 i	<--- 해당 네트워크로 통신하기위한 Next-hop = 13.13.12.1
```

```cisco
R2# show ip route
     172.16.0.0/24 is subnetted, 1 subnets
B       172.16.1.0 [20/0] via 13.13.12.1, 00:02:15
     10.0.0.0/24 is subnetted, 1 subnets
B       10.1.1.0 [20/0] via 13.13.12.1, 00:02:15
B    192.168.1.0/24 [20/0] via 13.13.12.1, 00:02:15
     13.0.0.0/24 is subnetted, 8 subnets
C       13.13.2.0 is directly connected, Loopback0
D       13.13.3.0 [90/2297856] via 13.13.23.3, 01:06:11, Serial1/0.23
D       13.13.4.0 [90/2809856] via 13.13.23.3, 01:06:03, Serial1/0.23
D       13.13.5.0 [90/3321856] via 13.13.23.3, 01:05:57, Serial1/0.23
C       13.13.12.0 is directly connected, Serial1/0.12	<------------ Next-hop의 IP 주소가 Routing table에서 확인된다.
C       13.13.23.0 is directly connected, Serial1/0.23
D       13.13.34.0 [90/2681856] via 13.13.23.3, 01:06:11, Serial1/0.23
D       13.13.45.0 [90/3193856] via 13.13.23.3, 01:06:04, Serial1/0.23
```

> 원문에는 `show ip route connecte`로 기재되어 있으나 출력에 B(BGP)·D(EIGRP) 경로가 함께 표시되므로 `show ip route` 전체 출력이다.

- R2는 BGP AS 100의 네트워크 정보를 `13.13.12.1`에게 수신하며 해당 네트워크로 통신하기 위한 Next-hop 정보는 **R2의 Connected**이므로 통신이 가능하다.

```cisco
R3# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
* i10.1.1.0/24      	13.13.12.1                      0     100      	0 100 i	<--- 해당 네트워크로 통신하기위한 Next-hop = 13.13.12.1
* i172.16.1.0/24    	13.13.12.1                      0     100      	0 100 i	<--- 해당 네트워크로 통신하기위한 Next-hop = 13.13.12.1
* i192.168.1.0      	13.13.12.1                      0     100      	0 100 i	<--- 해당 네트워크로 통신하기위한 Next-hop = 13.13.12.1
```

```cisco
R3# show ip route
     13.0.0.0/24 is subnetted, 7 subnets
D       13.13.2.0 [90/2297856] via 13.13.23.2, 01:11:13, Serial1/0.23
C       13.13.3.0 is directly connected, Loopback0
D       13.13.4.0 [90/2297856] via 13.13.34.4, 01:11:05, Serial1/0.34
D       13.13.5.0 [90/2809856] via 13.13.34.4, 01:10:59, Serial1/0.34
C       13.13.23.0 is directly connected, Serial1/0.23
C       13.13.34.0 is directly connected, Serial1/0.34
D       13.13.45.0 [90/2681856] via 13.13.34.4, 01:11:05, Serial1/0.34
```

- R1과 R2의 eBGP 연결 구간이 DMZ로 구성되기 때문에 `13.13.12.0/24` 네트워크 정보는 BGP AS 200 내의 Router에게 라우팅되지 않는다. (DMZ 구간 = IGP가 구성되지 않는 구간을 의미)
- 동일 BGP AS 내에서는 Next-hop이 변경되지 않기 때문에 BGP AS 200 내의 Router들은 BGP AS 100으로 통신하기 위한 Next-hop IP 주소가 Routing Table에서 확인되지 않아 통신할 수 없다.
- BGP로 수신한 네트워크 정보에 대해서 **통신할 수 없는 경로는 Routing Table에 등록되지 않는다.** (위 R3 `show ip bgp`에서 `*>`가 아니라 `* i`로 Best 표시 없음)

---

### 2-3. 방법 1) Static Route를 사용한 BGP Next-hop 문제 해결

```cisco
! === R3 ===
ip route 13.13.12.0  255.255.255.0  13.13.23.2
!

! === R4 ===
ip route 13.13.12.0  255.255.255.0  13.13.34.3
!

! === R5 ===
ip route 13.13.12.0  255.255.255.0  13.13.45.4
!
```

**정보 확인**

```cisco
R3# show ip route
      ~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~
     13.0.0.0/24 is subnetted, 8 subnets
D       13.13.2.0 [90/2297856] via 13.13.23.2, 01:39:05, Serial1/0.23
C       13.13.3.0 is directly connected, Loopback0
D       13.13.4.0 [90/2297856] via 13.13.34.4, 01:39:05, Serial1/0.34
D       13.13.5.0 [90/2809856] via 13.13.34.4, 01:39:05, Serial1/0.34
S       13.13.12.0 [1/0] via 13.13.23.2		<-------------- eBGP 구간 네트워크 정보 확인 (BGP AS 100의 Next-hop 주소)
C       13.13.23.0 is directly connected, Serial1/0.23
C       13.13.34.0 is directly connected, Serial1/0.34
      ~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~

R3# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.12.1                      0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i172.16.1.0/24    	13.13.12.1                      0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i192.168.1.0      	13.13.12.1                      0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능

R4# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*> 172.16.1.0/24    	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*> 192.168.1.0      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능

R5# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i172.16.1.0/24    	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i192.168.1.0      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
```

**정보 확인 후 Static Route 삭제**

```cisco
! === R3 ===
no ip route 13.13.12.0  255.255.255.0  13.13.23.2
!
! === R4 ===
no ip route 13.13.12.0  255.255.255.0  13.13.34.3
!
! === R5 ===
no ip route 13.13.12.0  255.255.255.0  13.13.45.4
!
```

---

### 2-4. 방법 2) eBGP DMZ 구간 네트워크를 IGP 재분배(Redistribute)하여 해결

```cisco
! === R2 ===
router eigrp 100
 redistribute connected metric 1544 2000 255 1 1500
!
```

**정보 확인**

```cisco
R3# show ip route
      ~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~
     13.0.0.0/24 is subnetted, 8 subnets
D       13.13.2.0 [90/2297856] via 13.13.23.2, 01:39:05, Serial1/0.23
C       13.13.3.0 is directly connected, Loopback0
D       13.13.4.0 [90/2297856] via 13.13.34.4, 01:39:05, Serial1/0.34
D       13.13.5.0 [90/2809856] via 13.13.34.4, 01:39:05, Serial1/0.34
D EX  13.13.12.0 [170/2681856] via 13.13.23.2, 00:00:02, Serial1/0.23  <--- eBGP 구간 네트워크 정보 (BGP AS 100의 Next-hop 주소)
C       13.13.23.0 is directly connected, Serial1/0.23
C       13.13.34.0 is directly connected, Serial1/0.34
      ~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~

R3# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.12.1                      0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i172.16.1.0/24    	13.13.12.1                      0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i192.168.1.0      	13.13.12.1                      0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능

R4# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*> 172.16.1.0/24    	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*> 192.168.1.0      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능

R5# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i172.16.1.0/24    	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i192.168.1.0      	13.13.12.1                      0     100      	0 (64512) 100 i	<--- BGP AS 100 네트워크로 통신 가능
```

#### EX1) 아래의 조건에 맞게 R2를 구성하시오

- R2에 Loopback 100을 생성한 후 `100.10.1.2/24` IP 주소를 할당하시오
- R2에 Loopback 200을 생성한 후 `200.20.2.2/24` IP 주소를 할당하시오
- R2의 Loopback 100, Loopback 200 네트워크는 다른 Router의 Routing Table에서 확인되지 않아야 한다.

> 원문 지시문의 Loopback 100 주소는 `100.10.1.2/24`이나, 설정 예제는 `100.10.2.2`로 기재되어 있다. 아래 설정은 원문 설정 그대로 기재한다. (지시문과 설정의 3번째 옥텟 불일치)

```cisco
! === R2 ===
interface loopback 100
 ip address 100.10.2.2  255.255.255.0
!
interface loopback 200
 ip address 200.20.2.2  255.255.255.0
!
```

**방법 A) Serial 1/0.12만 재분배 허용 (Permit)**

```cisco
! === R2 ===
route-map EIGRP_REDI permit 10
 match interface serial 1/0.12
!
router eigrp 100
 redistribute connected metric 1544 2000 255 1 1500 route-map EIGRP_REDI
!
```

**~~~~~~~ OR ~~~~~~~**

**방법 B) Loopback 100, 200만 재분배 차단 (Deny) 후 나머지 허용**

```cisco
! === R2 ===
route-map EIGRP_REDI deny 10
 match interface loopback100  loopback200
!
route-map EIGRP_REDI permit 20
!
router eigrp 100
 redistribute connected metric 1544 2000 255 1 1500 route-map EIGRP_REDI
!
```

**정보 확인 후 재분배 삭제**

```cisco
! === R2 ===
router eigrp 100
 no redistribute connected
!
```

---

### 2-5. 방법 3) BGP `next-hop-self` 기능을 사용한 해결

BGP에는 DMZ로 인한 BGP Next-hop 문제 발생 시 해당 문제를 해결하는 기능이 포함되어 있다. (**Next-hop-self**)

```cisco
! === R2 ===
router bgp 64512
 neighbor 13.13.3.3  next-hop-self
!
```

**정보 확인**

```cisco
R3# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.2.2                       0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i172.16.1.0/24    	13.13.2.2                       0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능
*>i192.168.1.0      	13.13.2.2                       0     100      	0 100 i	<--- BGP AS 100 네트워크로 통신 가능

R4# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.2.2                       0     100      	0 (64512) 100 i	<--- 동일 AS내에서는 Next-hop이 변경되지 않는다.
*> 172.16.1.0/24    	13.13.2.2                       0     100      	0 (64512) 100 i	<--- 동일 AS내에서는 Next-hop이 변경되지 않는다.
*> 192.168.1.0      	13.13.2.2                       0     100      	0 (64512) 100 i	<--- 동일 AS내에서는 Next-hop이 변경되지 않는다.

R5# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.2.2                       0     100      	0 (64512) 100 i	<--- 동일 AS내에서는 Next-hop이 변경되지 않는다.
*>i172.16.1.0/24    	13.13.2.2                       0     100      	0 (64512) 100 i	<--- 동일 AS내에서는 Next-hop이 변경되지 않는다.
*>i192.168.1.0      	13.13.2.2                       0     100      	0 (64512) 100 i	<--- 동일 AS내에서는 Next-hop이 변경되지 않는다.
```

- R2가 `next-hop-self`를 지정한 Peer(R3)에게는 Next-hop이 R2 자신의 Loopback IP(`13.13.2.2`)로 바뀌어 IGP(EIGRP)로 도달 가능하다.
- R2는 R3에게만 `next-hop-self`를 적용했지만, 이후 R3 ↔ R4(Confederation eBGP)·R4 ↔ R5(iBGP) 구간에서는 이미 변경된 Next-hop(`13.13.2.2`)이 **그대로 유지**된다. (동일 AS 내에서는 Next-hop이 변경되지 않는다.)

### 2-6. 3가지 방법 비교

| 방법 | 설정 위치 | 핵심 명령어 | 장점 | 단점 |
|------|-----------|-------------|------|------|
| 1) Static Route | AS 내부 모든 Router (R3, R4, R5) | `ip route 13.13.12.0 255.255.255.0 <next-hop>` | 단순 | Router마다 수동 입력, 확장성 낮음 |
| 2) IGP 재분배 | DMZ와 연결된 Border Router (R2) | `redistribute connected ... [route-map]` | 한 곳만 설정 | DMZ 구간이 IGP에 노출됨, Route-map으로 범위 제어 필요 |
| 3) next-hop-self | Border Router (R2) | `neighbor <iBGP-Peer> next-hop-self` | BGP 자체 기능, 가장 깔끔 | iBGP Peer 별로 지정 필요 |

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```cisco
Router# show ip bgp                       ! Next Hop 컬럼 확인, Best(>) 표시 여부 확인
Router# show ip route <next-hop-network>  ! Next-hop IP가 Routing Table에 존재하는지 확인
Router# show ip route bgp                 ! BGP 경로가 실제 Routing Table에 등록됐는지 확인
```

### 3-2. Case별 트러블슈팅 가이드

#### Case A. **AS 내부 Router의 BGP Table에는 있지만 `show ip route bgp`에 안 보임**

- **원인 파악:** Next-hop(예: `13.13.12.1`)이 IGP로 도달 불가 — DMZ 구간이므로 AS 내부 Router의 Routing Table에 Next-hop 네트워크가 없다. BGP Table에서도 `*>`(Best) 대신 `* i`로 표시된다.
- **해결:** Static Route / DMZ IGP 재분배 / `next-hop-self` 중 하나를 적용한다.

#### Case B. **DMZ 구간 재분배 시 의도하지 않은 Loopback까지 다른 Router에 전파됨**

- **원인 파악:** `redistribute connected`는 모든 Connected 네트워크를 재분배한다.
- **해결:** Route-map(`match interface`)으로 재분배 대상을 제한한다. (Serial 1/0.12만 Permit, 또는 Loopback 100/200 Deny + 나머지 Permit)

#### Case C. **DMZ 구성 후 Loopback eBGP Peer가 끊김**

- **원인 파악:** DMZ 구간에는 IGP가 없어 서로의 Loopback 0 네트워크를 알 수 없다.
- **해결:** eBGP Peer를 물리적 Interface IP(13.13.12.X)로 재구성한다.
