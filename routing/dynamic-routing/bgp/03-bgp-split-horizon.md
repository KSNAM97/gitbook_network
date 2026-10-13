# BGP Split-horizon 문제 해결 (Full-mesh · Route-reflector · Confederation)

> **Tag:** #Network #Cisco #BGP #Routing #iBGP #SplitHorizon #FullMesh #RouteReflector #Confederation
> **핵심 요약:** iBGP로 수신한 네트워크는 다른 iBGP Peer에게 다시 광고할 수 없다(BGP Split-horizon). 동일 AS 내 3대 이상 Router에서 BGP 구성 시 라우팅 업데이트가 끊기는 이 문제를 **Full-mesh(모든 Router Peer) / Route-reflector(RR 경유 재광고) / Confederation(AS를 사설 Sub-AS로 분할)** 3가지 방식으로 해결한다.

---

## 1. 핵심 기술 개념 (Concept)

### BGP Split-horizon 문제 확인

```cisco
R2# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.1.1                   0                    0 100 i
*> 172.16.1.0/24	13.13.1.1                   0                    0 100 i
*> 192.168.1.0 	13.13.1.1                   0                    0 100 i

R3# show ip bgp
   Network          	Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1                   0      100         0 100 i
*>i172.16.1.0/24    	13.13.1.1                   0      100         0 100 i
*>i192.168.1.0      	13.13.1.1                   0      100         0 100 i

R4# show ip route bgp
R4# show ip bgp

R5# show ip route bgp
R5# show ip bgp
```

- R1이 R2에게 eBGP로 라우팅 업데이트하게 되면 R2는 R1에게 받은 네트워크 정보를 R3에게 iBGP로 라우팅 업데이트하게 된다. (Routing Table 및 BGP Table에서 확인 가능)
- R3은 R2에게 수신한 iBGP 네트워크 정보를 R4에게 iBGP로 라우팅 업데이트할 수 없다. (원인 : BGP Split-horizon)
- 위 출력에서 R4, R5는 `show ip route bgp`, `show ip bgp` 결과가 **비어 있다.**

### BGP Split-horizon 동작 원리

- BGP는 광역망을 연결 시 사용하는 Protocol이므로 **강력한 Split-horizon**이 적용된다.
- BGP는 Loop를 방지하기 위해서 **iBGP로 수신한 네트워크 정보는 iBGP로 다시 송신할 수 없다.** 즉 하나의 AS 내에서 3대 이상의 Router에서 BGP 구성 시 라우팅 업데이트되지 않는 문제가 발생한다. (**iBGP로는 한 번만 라우팅 업데이트가 가능하다.**)
- eBGP는 거쳐온 AS를 기록하기 때문에 `AS_PATH`에 자신의 AS 정보가 포함 시 해당 네트워크를 수신하지 않아 Loop가 방지된다.
- iBGP Peer에서는 `AS_PATH`가 동일하기 때문에 `AS_PATH`로는 Loop를 방지할 수 없다. 동일 AS에서 Loop를 방지하기 위해서 iBGP로는 한 번만 라우팅이 가능하다.

### 해결 방식 3가지 비교

| 방식 | 원리 | 장점 | 단점 / 주의 |
|------|------|------|-------------|
| **Full-mesh** | 동일 AS 내 **모든 BGP Router 간 iBGP Peer** 연결 → 모든 Router가 iBGP로 "한 번에" 라우팅 | 단순, 개념이 명확 | Router 수가 늘수록 Peer 수 급증 |
| **Route-reflector** | RR Router가 iBGP로 수신한 정보를 **RR-client에게 한 번 더 iBGP로 라우팅** | Peer 수 감소 | RR이 단일 장애점(SPOF)이 될 수 있음 |
| **Confederation** | 하나의 AS 내에 **가상의 사설 AS(Sub-AS)** 를 만들어 iBGP Peer 관계를 **논리적인 eBGP Peer**로 변경 | 대규모 AS 분할 | 설정이 복잡, Sub-AS 경계 설계 필요 |

> **사전 조건:** 이 문서의 실습은 **BGP Peer 연결**의 `EX2) R1(AS 100) ↔ R2(AS 200) Loopback eBGP Peer (ebgp-multihop 3)` 및 `EX3) R1 Loopback 10/172/192 network 광고` 구성이 완료된 상태에서 진행한다.

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. Full-mesh 방식

```text
       Loopback 0                         Loopback 0                       Loopback 0                       Loopback 0                         Loopback 0
       13.13.1.0/24                        13.13.2.0/24                      13.13.3.0/24                      13.13.4.0/24                        13.13.5.0/24

                           13.13.12.0/24                    13.13.23.0/24                     13.13.34.0/24                    13.13.45.0/24
               R1------------------------R2------------------------R3------------------------R4------------------------R5
                           Serial 1/0.12 P                 Serial 1/0.23 P                  Serial 1/0.34 P                    Serial 1/0.45 P

   |--- BGP AS 100 ---|        |--------------------------------------- BGP AS 200 ---------------------------------------|
```

- 동일 AS 내의 모든 BGP Router 간 BGP Peer를 연결하는 기능
- 동일 AS 내의 모든 BGP Router 간 iBGP로 한 번에 라우팅하기 때문에 Split-horizon 문제를 해결할 수 있다.

**1단계: R2를 중심으로 R3, R4, R5와 iBGP Peer 연결**

```cisco
! === R2 ===
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source loopback 0
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source loopback 0
!

! === R3 ===
router bgp 200
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!

! === R4 ===
router bgp 200
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!

! === R5 ===
router bgp 200
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!
```

**정보 확인**

```cisco
R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.3.3        4    200          6          7           4     0      0  00:01:27           0
13.13.4.4        4    200          5          6           4     0      0  00:01:25           0
13.13.5.5        4    200          5          6           4     0      0  00:01:23           0
13.13.1.1        4    100          9          9           4     0      0  00:44:52           3	<--- eBGP로 3개의 네트워크를 수신

R3# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          17         16          4     0       0 00:02:17           3	<--- iBGP로 3개의 네트워크를 수신
13.13.4.4       4    200          51         52          4     0       0 00:48:33           0
13.13.5.5       4    200          14         14          4     0       0 00:00:35           0

R4# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          17         16          4     0       0 00:02:17           3	<--- iBGP로 3개의 네트워크를 수신
13.13.3.3        4   200          51         52          4     0       0 00:48:33           0
13.13.5.5        4   200          14         14          4     0       0 00:00:35           0

R5# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          17         16          4     0       0 00:02:17           3	<--- iBGP로 3개의 네트워크를 수신
13.13.3.3        4   200          51         52          4     0       0 00:48:33           0
13.13.4.4        4   200          14         14          4     0       0 00:00:35           0
```

```cisco
R2# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.1.1                  0                   0 100 i	: *>	= i가 표기되지 않음 (eBGP로 수신한 네트워크)
*> 172.16.1.0/24    	13.13.1.1                  0                   0 100 i	: *>	= i가 표기되지 않음 (eBGP로 수신한 네트워크)
*> 192.168.1.0      	13.13.1.1                  0                   0 100 i	: *>	= i가 표기되지 않음 (eBGP로 수신한 네트워크)

R3# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1                  0     100         0 100 i	: *>i	= i가 표기  (iBGP로 수신한 네트워크)
*>i172.16.1.0/24    	13.13.1.1                  0     100         0 100 i	: *>i	= i가 표기  (iBGP로 수신한 네트워크)
*>i192.168.1.0      	13.13.1.1                  0     100         0 100 i	: *>i	= i가 표기  (iBGP로 수신한 네트워크)

R4# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1                  0     100         0 100 i
*>i172.16.1.0/24    	13.13.1.1                  0     100         0 100 i
*>i192.168.1.0      	13.13.1.1                  0     100         0 100 i

R5# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1                  0     100         0 100 i
*>i172.16.1.0/24    	13.13.1.1                  0     100         0 100 i
*>i192.168.1.0      	13.13.1.1                  0     100         0 100 i
```

---

#### EX) 아래의 조건에 맞게 BGP AS 200을 구성하시오

- BGP AS 200 내의 Router는 Loopback 192를 생성한 후 `192.168.X.X/24` IP 주소를 할당해야 한다.
- Loopback 192 네트워크는 BGP AS 200으로 라우팅 업데이트 되어야 한다.

```cisco
! === R2 ===
interface loopback 192
 ip address 192.168.2.2  255.255.255.0
!
router bgp 200
 network 192.168.2.0
!

! === R3 ===
interface loopback 192
 ip address 192.168.3.3  255.255.255.0
!
router bgp 200
 network 192.168.3.0
!

! === R4 ===
interface loopback 192
 ip address 192.168.4.4  255.255.255.0
!
router bgp 200
 network 192.168.4.0
!

! === R5 ===
interface loopback 192
 ip address 192.168.5.5  255.255.255.0
!
router bgp 200
 network 192.168.5.0
!
```

**정보 확인**

```cisco
R2# show ip bgp
R3# show ip bgp
R4# show ip bgp
R5# show ip bgp
```

- R5가 `192.168.5.0/24` 네트워크 정보를 R2에게 iBGP로 라우팅 업데이트하게 되면 R2는 `192.168.5.0/24` 네트워크 정보를 Routing Table 및 BGP Table에 등록한 후 **R3, R4에게 iBGP로 라우팅 업데이트할 수 없다.** (BGP Split-horizon)
- BGP **Full-mesh 방식은 동일 AS에 포함된 모든 BGP Router 간 iBGP Peer를 연결**해야 한다.

**2단계: 나머지 Router 간 iBGP Peer를 모두 연결 (Full-mesh 완성)**

```cisco
! === R3 ===
router bgp 200
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source loopback 0
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source loopback 0
!

! === R4 ===
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source loopback 0
!

! === R5 ===
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source loopback 0
!
```

**정보 확인**

```cisco
R1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2      4   200           77         71           8     0      0  01:06:36           4	       <--- eBGP Peer에게 4개의 네트워크 정보 수신

R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4   100          64          68         11     0      0  01:03:01           3	       <--- eBGP Peer (iBGP 네트워크 3개수신)
13.13.3.3        4   200          22          24         11     0      0  00:19:36           1	       <--- iBGP Peer (iBGP 네트워크 1개수신)
13.13.4.4        4   200          22          24         11     0      0  00:19:34           1	       <--- iBGP Peer (iBGP 네트워크 1개수신)
13.13.5.5        4   200          22          24         11     0      0  00:19:31           1	       <--- iBGP Peer (iBGP 네트워크 1개수신)

R3# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          25          22           5     0      0 00:19:00           4	       <--- iBGP Peer (eBGP 3개 , iBGP 1개 수신)
13.13.4.4        4   200          14          14           5     0      0 00:00:58           1	       <--- iBGP Peer (iBGP 네트워크 1개수신)
13.13.5.5        4   200          16          15           5     0      0 00:01:04           1	       <--- iBGP Peer (iBGP 네트워크 1개수신)

R4# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          25          23           5     0      0 00:19:05           4	       <--- iBGP Peer
13.13.3.3        4   200          15          15           5     0      0 00:01:13           1	       <--- iBGP Peer
13.13.5.5        4   200          16          15           5     0      0 00:01:07           1	       <--- iBGP Peer

R5# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          25          24           5     0      0 00:19:09           4	       <--- iBGP Peer
13.13.3.3        4   200          15          16           5     0      0 00:01:37           1	       <--- iBGP Peer
13.13.4.4        4   200          15          16           5     0      0 00:01:25           1	       <--- iBGP Peer
```

**다음 실습을 위해 BGP 설정 삭제**

```cisco
! === R1 ===
no router bgp 100
!
! === R2 ~ R5 ===
no router bgp 200
!
```

---

### 2-2. Route-reflector 방식

- **BGP Split-horizon** : iBGP로 수신한 네트워크 정보를 iBGP로 라우팅할 수 없다.
- **Route-reflector** : Route-reflector를 생성한 후 Route-reflector Router를 사용하여 iBGP로 수신한 네트워크 정보를 iBGP로 라우팅하여 BGP Split-horizon 문제를 해결할 수 있다.
- Route-reflector는 **Route-reflector-client** Router에게 iBGP로 한 번 더 라우팅할 수 있다.

#### EX1) 아래의 조건에 맞게 BGP를 구성하시오

- R1의 BGP AS 100과 R2의 BGP AS 200 구간에 eBGP Peer를 연결해야 한다.
- eBGP Peer 연결 시 Loopback IP 주소를 사용해야 한다.
- BGP Router-ID = X.X.X.X (X = Router 번호)

```cisco
! === R1 ===
router bgp 100
 no auto-summary
 no synchronization
 bgp router-id 1.1.1.1
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
 neighbor 13.13.2.2 ebgp-multihop 3
!

! === R2 ===
router bgp 200
 no auto-summary
 no synchronization
 bgp router-id 2.2.2.2
 neighbor 13.13.1.1 remote-as 100
 neighbor 13.13.1.1 update-source loopback 0
 neighbor 13.13.1.1 ebgp-multihop 3
!
```

```cisco
R1# show ip bgp summary
Neighbor        V	AS   MsgRcvd   MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4    200	   7	   7	  1     0      0  00:03:19          0

R2# show ip bgp summary
Neighbor        V	AS   MsgRcvd   MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4    100	   7	   7	  1     0      0  00:03:19          0
```

#### EX2) BGP AS 200에서 아래의 조건에 맞게 BGP Peer를 연결하시오

- BGP Router-ID = X.X.X.X (X = Router 번호)
- R2와 R3 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 Loopback Interface를 사용해야 한다.
- R2와 R4 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 Loopback Interface를 사용해야 한다.
- R2와 R5 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 Loopback Interface를 사용해야 한다.

> 원본 지시문에는 "물리적인 Interface"로 기재되어 있으나 설정은 `update-source loopback 0` 기반의 Loopback Peer 연결이다.

```cisco
! === R2 ===
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source loopback 0
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source loopback 0
!

! === R3 ===
router bgp 200
 no auto-summary
 no synchronization
 bgp router-id 3.3.3.3
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!

! === R4 ===
router bgp 200
 no auto-summary
 no synchronization
 bgp router-id 4.4.4.4
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!

! === R5 ===
router bgp 200
 no auto-summary
 no synchronization
 bgp router-id 5.5.5.5
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!
```

**정보 확인**

```cisco
R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.3.3        4    200          6          7           4     0      0  00:01:27           0
13.13.4.4        4    200          5          6           4     0      0  00:01:25           0
13.13.5.5        4    200          5          6           4     0      0  00:01:23           0
13.13.1.1        4    100          9          9           4     0      0  00:44:52           0

R3# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200           7           6          4     0       0 00:02:17           0

R4# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200           7           6           4    0       0 00:02:17           0

R5# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200           7           6          4     0       0 00:02:17           0
```

#### EX3) 아래의 조건에 맞게 R1을 구성하시오

- R1에 Loopback 10을 생성한 후 `10.1.1.0/24` IP 주소를 할당하시오
- R1에 Loopback 172를 생성한 후 `172.16.1.0/24` IP 주소를 할당하시오
- R1에 Loopback 192를 생성한 후 `192.168.1.0/24` IP 주소를 할당하시오
- Lo 10, Lo 172, Lo 192 네트워크를 BGP로 라우팅 업데이트해야 한다.

```cisco
! === R1 ===
interface loopback 10
 ip address 10.1.1.1  255.255.255.0
!
interface loopback 172
 ip address 172.16.1.1  255.255.255.0
!
interface loopback 192
 ip address 192.168.1.1  255.255.255.0
!
router bgp 100
 network 10.1.1.0 mask 255.255.255.0
 network 172.16.1.0 mask 255.255.255.0
 network 192.168.1.0
!
```

**정보 확인**

```cisco
R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4   100            9          8          4      0      0 00:09:50           3		<---- R1에게 eBGP로 3개의 네트워크 정보 수신
13.13.3.3        4   200            4          5          4      0      0 00:00:40           0
13.13.4.4        4   200            8          9          4      0      0 00:00:03           0
13.13.5.5        4   200            4          5          4      0      0 00:00:09           0

R3# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200           6          5           4     0      0  00:01:52           3		<---- R2에게 iBGP로 3개의 네트워크 정보 수신

R4# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200           6          5           4     0      0  00:01:52           3		<---- R2에게 iBGP로 3개의 네트워크 정보 수신

R5# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200           6          5           4     0      0  00:01:52           3		<---- R2에게 iBGP로 3개의 네트워크 정보 수신
```

```cisco
R2# show ip route bgp
R2# show ip bgp

R3# show ip route bgp
R3# show ip bgp

R4# show ip route bgp
R4# show ip bgp

R5# show ip route bgp
R5# show ip bgp
```

#### EX4) 아래의 조건에 맞게 BGP AS 200을 구성하시오

- BGP AS 200 내의 Router는 Loopback 192를 생성한 후 `192.168.X.X/24` IP 주소를 할당해야 한다.
- Loopback 192 네트워크는 BGP AS 200으로 라우팅 업데이트 되어야 하며 BGP Table에서 확인되어야 한다.

```cisco
! === R2 ===
interface loopback 192
 ip address 192.168.2.2  255.255.255.0
!
router bgp 200
 network 192.168.2.0
!

! === R3 ===
interface loopback 192
 ip address 192.168.3.3  255.255.255.0
!
router bgp 200
 network 192.168.3.0
!

! === R4 ===
interface loopback 192
 ip address 192.168.4.4  255.255.255.0
!
router bgp 200
 network 192.168.4.0
!

! === R5 ===
interface loopback 192
 ip address 192.168.5.5  255.255.255.0
!
router bgp 200
 network 192.168.5.0
!
```

**정보 확인 (아직 Route-reflector 미적용)**

```cisco
R2# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.1.1		        0             	 0 100 i
*> 172.16.1.0/24    	13.13.1.1		        0             	 0 100 i
*> 192.168.1.0      	13.13.1.1		        0             	 0 100 i
*> 192.168.2.0      	0.0.0.0    		        0             32768 i	<--- 해당 Router가 라우팅한 네트워크 정보
*>i192.168.3.0      	13.13.3.3		        0     100      	 0 i	<--- BGP AS 200의 R3 네트워크 정보
*>i192.168.4.0      	13.13.4.4		        0     100      	 0 i	<--- BGP AS 200의 R4 네트워크 정보
*>i192.168.5.0      	13.13.5.5		        0     100      	 0 i	<--- BGP AS 200의 R5 네트워크 정보

R3# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1   	        0     100      	 0 100 i
*>i172.16.1.0/24    	13.13.1.1    	        0     100      	 0 100 i
*>i192.168.1.0      	13.13.1.1   	        0     100      	 0 100 i
*>i192.168.2.0      	13.13.2.2    	        0     100      	 0 i	<--- BGP AS 200의 R2 네트워크 정보
*> 192.168.3.0      	0.0.0.0     	        0             32768 i	<--- 해당 Router가 라우팅한 네트워크 정보

R4# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1   	        0     100      	 0 100 i
*>i172.16.1.0/24    	13.13.1.1    	        0     100      	 0 100 i
*>i192.168.1.0      	13.13.1.1   	        0     100      	 0 100 i
*>i192.168.2.0      	13.13.2.2    	        0     100      	 0 i	<--- BGP AS 200의 R2 네트워크 정보
*> 192.168.4.0      	0.0.0.0     	        0             32768 i	<--- 해당 Router가 라우팅한 네트워크 정보

R5# show ip bgp
    Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1   	        0     100      	 0 100 i
*>i172.16.1.0/24    	13.13.1.1    	        0     100      	 0 100 i
*>i192.168.1.0      	13.13.1.1   	        0     100      	 0 100 i
*>i192.168.2.0      	13.13.2.2    	        0     100      	 0 i	<--- BGP AS 200의 R2 네트워크 정보
*> 192.168.5.0      	0.0.0.0     	        0             32768 i	<--- 해당 Router가 라우팅한 네트워크 정보
```

- BGP AS 200 내의 R3, R4, R5가 Loopback 192 네트워크를 R2에게 라우팅하게 되면 R2는 iBGP로 수신한 네트워크 정보를 iBGP Peer인 R3, R4, R5에게 라우팅할 수 없다. (R3 ↔ R4 ↔ R5 간 `192.168.X.0`이 서로 보이지 않음)

#### EX5) 아래의 조건에 맞게 BGP AS 200을 구성하시오

- BGP AS 200 내의 모든 네트워크는 BGP AS 200 내의 모든 Router의 BGP Table, Routing Table에서 확인되어야 한다.
- R2가 **BGP Route-reflector**로 동작하며 R3, R4, R5가 **Route-reflector-client**로 동작해야 한다.

```cisco
! === R2 (Route-reflector) ===
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source Loopback0
 neighbor 13.13.3.3 route-reflector-client	<---- R3을 Route-reflector-client로 지정
!
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source Loopback0
 neighbor 13.13.4.4 route-reflector-client	<---- R4을 Route-reflector-client로 지정
!
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source Loopback0
 neighbor 13.13.5.5 route-reflector-client	<---- R5을 Route-reflector-client로 지정
!
```

`route-reflector-client` 설정이 바뀌면 해당 Peer가 Reset 되는 로그가 발생한다.

```text
*Mar  1 03:13:00.775: %BGP-5-ADJCHANGE: neighbor 13.13.3.3 Down RR client config change
*Mar  1 03:13:00.783: %BGP-5-ADJCHANGE: neighbor 13.13.4.4 Down RR client config change
*Mar  1 03:13:00.787: %BGP-5-ADJCHANGE: neighbor 13.13.5.5 Down RR client config change

*Mar  1 03:13:03.231: %BGP-5-ADJCHANGE: neighbor 13.13.3.3 Up
*Mar  1 03:13:03.427: %BGP-5-ADJCHANGE: neighbor 13.13.4.4 Up
*Mar  1 03:13:03.803: %BGP-5-ADJCHANGE: neighbor 13.13.5.5 Up
```

**정보 확인**

```cisco
R3# show ip bgp
   Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1         	       0      100      	0 100 i
*>i172.16.1.0/24    	13.13.1.1         	       0      100      	0 100 i
*>i192.168.1.0      	13.13.1.1        	       0      100      	0 100 i
*>i192.168.2.0      	13.13.2.2        	       0      100      	0 i	<--- BGP AS 200 네트워크
*> 192.168.3.0      	0.0.0.0         	       0             32768 i	<--- BGP AS 200 네트워크
*>i192.168.4.0      	13.13.4.4        	       0      100      	0 i	<--- BGP AS 200 네트워크
*>i192.168.5.0      	13.13.5.5         	       0      100      	0 i	<--- BGP AS 200 네트워크

R4# show ip bgp
   Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1         	       0      100      	0 100 i
*>i172.16.1.0/24    	13.13.1.1         	       0      100      	0 100 i
*>i192.168.1.0      	13.13.1.1        	       0      100      	0 100 i
*>i192.168.2.0      	13.13.2.2        	       0      100      	0 i	<--- BGP AS 200 네트워크
*> 192.168.3.0      	13.13.3.3        	       0      100      	0 i	<--- BGP AS 200 네트워크
*>i192.168.4.0      	0.0.0.0         	       0             32768 i	<--- BGP AS 200 네트워크
*>i192.168.5.0      	13.13.5.5         	       0      100      	0 i	<--- BGP AS 200 네트워크

R5# show ip bgp
   Network          	Next Hop                Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1         	       0      100      	0 100 i
*>i172.16.1.0/24    	13.13.1.1         	       0      100      	0 100 i
*>i192.168.1.0      	13.13.1.1        	       0      100      	0 100 i
*>i192.168.2.0      	13.13.2.2        	       0      100      	0 i	<--- BGP AS 200 네트워크
*> 192.168.3.0      	13.13.3.3        	       0      100      	0 i	<--- BGP AS 200 네트워크
*>i192.168.4.0      	13.13.4.4        	       0      100      	0 i	<--- BGP AS 200 네트워크
*>i192.168.5.0      	0.0.0.0         	       0             32768 i	<--- BGP AS 200 네트워크
```

> 출력의 `*>i` / `*>` 표기 일부는 원문 그대로이며, 핵심은 **R2(RR)가 R3·R4·R5의 `192.168.X.0`을 서로에게 반사(reflect)해 주기 때문에 모든 Router의 BGP Table에 AS 200 네트워크가 보인다**는 점이다.

**다음 실습을 위해 BGP 설정 삭제**

```cisco
! === R1 ===
no router bgp 100
!
! === R2 ~ R5 ===
no router bgp 200
!
```

---

### 2-3. Confederation 방식

- 하나의 BGP AS 내에서 **가상의 사설 AS를 생성**하여 iBGP Peer 관계를 **논리적인 eBGP Peer로 변경**하여 라우팅하는 방식

```text
   |--- BGP AS 100 ---|     |------------------------------- BGP AS 200 -------------------------------|

                      13.13.12.0/24                    13.13.23.0/24                       13.13.34.0/24                   13.13.45.0/24
          R1--------------------R2--------------------R3--------------------R4--------------------R5
                       Serial 1/0.12 P               Serial 1/0.23 P                 Serial 1/0.34 P                      Serial 1/0.45 P

                                        |----------- BGP AS 64512 -----------|     |----------- BGP AS 64513 -----------|
```

- 실제 AS 200 = Confederation Identifier, 내부 Sub-AS는 **64512 (R2·R3)** 와 **64513 (R4·R5)** 로 분할한다.

#### EX1) 아래의 조건에 맞게 R1, R2에서 BGP를 구성하시오

- R1의 BGP AS 100과 R2의 BGP AS 200 간 eBGP Peer를 연결하시오
- BGP Peer 연결 시 Loopback 0의 IP 주소를 사용해야 한다.
- R2에서는 BGP Confederation을 사용하기 위해서 가상의 사설 AS인 64512를 사용해야 한다.

```cisco
! === R1 ===
router bgp 100
 no auto-summary
 no synchronization
 bgp router-id 1.1.1.1
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
 neighbor 13.13.2.2 ebgp-multihop 3
!
*Mar  1 03:40:04.291: %BGP-3-NOTIFICATION: sent to neighbor 13.13.2.2 2/2 (peer in wrong AS) 2 bytes FC00
FFFF FFFF FFFF FFFF FFFF FFFF FFFF FFFF 002D 0104 FC00 00B4 0202 0202 1002 0601 0400 0100 0102 0280 0002 0202 00

! === R2 ===
router bgp 64512
 no auto-summary
 no synchronization
 bgp router-id 2.2.2.2
 neighbor 13.13.1.1 remote-as 100
 neighbor 13.13.1.1 update-source loopback 0
 neighbor 13.13.1.1 ebgp-multihop 3
!
*Mar  1 04:13:53.534: %BGP-3-NOTIFICATION: received from neighbor 13.13.1.1 2/2 (peer in wrong AS) 2 bytes FC00
```

- BGP AS 100의 R1은 BGP Peer 연결 시 AS 번호를 **200**으로 설정
- BGP AS 200의 R2는 BGP Peer 연결 시 AS 번호를 **64512**를 사용했기 때문에 AS 번호 불일치로 인해 `peer in wrong AS` 에러 메시지가 확인된다.
- BGP AS 200의 R2에서 **`bgp confederation identifier`** 명령어를 사용하여 실제 사용할 AS 번호를 설정해야 한다.

```cisco
! === R2 ===
router bgp 64512
 bgp confederation identifier 200	<--- 실제 사용할 AS 번호 설정
!
```

**정보 확인**

```cisco
R1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200          12         12           1     0      0 00:00:37           0

R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4   100          12         12           1     0      0 00:00:44           0
```

#### EX2) 아래의 조건에 맞게 BGP AS 200 내의 Router 간 iBGP Peer를 연결하시오

- BGP 구성 시 Confederation을 사용한 iBGP Peer를 연결해야 한다.
- R2와 R3 구간 iBGP Peer 연결 시 BGP 사설 AS 64512를 사용해야 한다.
- R4와 R5 구간 iBGP Peer 연결 시 BGP 사설 AS 64513을 사용해야 한다.
- R3와 R4 구간 BGP Peer 연결 시 가상의 eBGP Peer를 연결해야 한다.

**Confederation iBGP Peer 연결 (R2 ↔ R3)**

```cisco
! === R2 ===
router bgp 64512
 neighbor 13.13.3.3  remote-as 64512
 neighbor 13.13.3.3  update-source loopback 0
!

! === R3 ===
router bgp 64512
 no auto-summary
 no synchronization
 bgp router-id 3.3.3.3
 bgp confederation identifier 200
 neighbor 13.13.2.2  remote-as 64512
 neighbor 13.13.2.2  update-source loopback 0
!
```

```cisco
R2# show ip bgp summary
Neighbor        V      AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4     100	18         18           1     0      0  00:06:56           0
13.13.3.3        4  64512     	15         15           1     0      0  00:01:26           0

R3# show ip bgp summary
Neighbor        V      AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4  64512     	15         15           1     0      0  00:01:26           0
```

**Confederation iBGP Peer 연결 (R4 ↔ R5)**

```cisco
! === R4 ===
router bgp 64513
 no auto-summary
 no synchronization
 bgp router-id 4.4.4.4
 bgp confederation identifier 200
 neighbor 13.13.5.5 remote-as 64513
 neighbor 13.13.5.5 update-source loopback 0
!

! === R5 ===
router bgp 64513
 no auto-summary
 no synchronization
 bgp router-id 5.5.5.5
 bgp confederation identifier 200
 neighbor 13.13.4.4 remote-as 64513
 neighbor 13.13.4.4 update-source loopback 0
!
```

```cisco
R4# show ip bgp summary
Neighbor        V     AS  MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.5.5        4  64513	 2           2           0      0      0 00:00:16           0

R5# show ip bgp summary
Neighbor        V     AS  MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.4.4        4  64513	 2           2           0      0      0 00:00:16           0
```

**Confederation eBGP Peer 연결 (R3 ↔ R4)**

```cisco
! === R3 ===
router bgp 64512
 bgp confederation peers 64513
 neighbor 13.13.4.4 remote-as 64513
 neighbor 13.13.4.4 update-source loopback 0
 neighbor 13.13.4.4 ebgp-multihop 3
!

! === R4 ===
router bgp 64513
 bgp confederation peers 64512
 neighbor 13.13.3.3 remote-as 64512
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.3.3 ebgp-multihop 3
!
```

> 원문에는 `bgp confederation peer 64513`으로 입력되어 있으나 IOS의 정식 명령어는 **`bgp confederation peers <Sub-AS>`** 이다. (약어 입력 시에도 동작)

```cisco
R3# show ip bgp summary
Neighbor        V     AS   MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4 64512	20          21           1     0      0  00:17:34           0
13.13.4.4        4 64513	16          16           1     0      0  00:00:03           0

R4# show ip bgp summary
Neighbor        V     AS   MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.3.3        4 64512	16          16           1     0      0  00:00:03           0
13.13.5.5        4 64513	16          18           1     0      0  00:14:05           0
```

#### EX3) 아래의 조건에 맞게 R1을 구성하시오

- R1에 Loopback 10을 생성한 후 `10.1.1.0/24` IP 주소를 할당하시오
- R1에 Loopback 172를 생성한 후 `172.16.1.0/24` IP 주소를 할당하시오
- R1에 Loopback 192를 생성한 후 `192.168.1.0/24` IP 주소를 할당하시오
- Lo 10, Lo 172, Lo 192 네트워크를 BGP로 라우팅 업데이트해야 한다.

```cisco
! === R1 ===
interface loopback 10
 ip address 10.1.1.1  255.255.255.0
!
interface loopback 172
 ip address 172.16.1.1  255.255.255.0
!
interface loopback 192
 ip address 192.168.1.1  255.255.255.0
!
router bgp 100
 network 10.1.1.0 mask 255.255.255.0
 network 172.16.1.0 mask 255.255.255.0
 network 192.168.1.0
!
```

**정보 확인**

```cisco
R2# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.1.1                   0                    0 100 i	<--- eBGP로 수신
*> 172.16.1.0/24    	13.13.1.1                   0                    0 100 i	<--- eBGP로 수신
*> 192.168.1.0      	13.13.1.1                   0                    0 100 i	<--- eBGP로 수신

R3# show ip bgp
    Network          	Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1                   0      100         0 100 i	<--- iBGP로 수신
*>i172.16.1.0/24    	13.13.1.1                   0      100         0 100 i	<--- iBGP로 수신
*>i192.168.1.0      	13.13.1.1                   0      100         0 100 i	<--- iBGP로 수신

R4# show ip bgp
     Network          	Next Hop            Metric LocPrf Weight Path
*> 10.1.1.0/24      	13.13.1.1                   0      100         0 (64512) 100 i	<--- Confederation eBGP로 수신
*> 172.16.1.0/24    	13.13.1.1                   0      100         0 (64512) 100 i	<--- Confederation eBGP로 수신
*> 192.168.1.0      	13.13.1.1                   0      100         0 (64512) 100 i	<--- Confederation eBGP로 수신

R5# show ip bgp
     Network          	Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      	13.13.1.1                   0      100         0 (64512) 100 i	<--- iBGP로 수신
*>i172.16.1.0/24    	13.13.1.1                   0      100         0 (64512) 100 i	<--- iBGP로 수신
*>i192.168.1.0      	13.13.1.1                   0      100         0 (64512) 100 i	<--- iBGP로 수신
```

- Confederation 내부 Sub-AS 경계를 넘어 수신한 경로는 `AS_PATH`에 `(64512)`처럼 **괄호로 Sub-AS 번호**가 기록된다. 이 값은 AS 200 밖으로 광고될 때 제거된다.
- R3 ↔ R4 구간은 논리적으로 eBGP 취급이므로 R3가 R2에게서 iBGP로 받은 경로를 R4에게 전달할 수 있다. 즉 Split-horizon이 해결된다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```cisco
Router# show ip bgp summary          ! Peer 상태 및 수신 Prefix 수 (iBGP/eBGP 구분은 AS 컬럼으로 확인)
Router# show ip bgp                  ! BGP Table (*>i = iBGP로 수신, *> = eBGP/로컬)
Router# show ip route bgp            ! Routing Table에 등록된 BGP 경로
```

### 3-2. Case별 트러블슈팅 가이드

#### Case A. **R3까지는 보이는데 R4·R5에는 BGP 네트워크가 없음**

- **원인 파악:** iBGP로 수신한 Route를 다른 iBGP Peer에게 재광고하지 못하는 BGP Split-horizon.
- **해결:** Full-mesh(모든 Router 간 iBGP Peer), Route-reflector(`route-reflector-client`), Confederation(`bgp confederation identifier` / `peers`) 중 하나를 적용한다.

#### Case B. **Route-reflector 설정 후 Peer가 잠깐 Down/Up 됨**

- **원인 파악:** `route-reflector-client` 설정 변경 시 `%BGP-5-ADJCHANGE ... Down RR client config change`가 발생하며 해당 Peer가 Reset되는 것은 정상 동작이다.
- **확인:** 이후 `Up` 메시지가 출력되고 `show ip bgp`에 AS 200 내부 네트워크가 모두 학습되는지 확인한다.

#### Case C. **Confederation 구성 시 `peer in wrong AS` Notification 발생**

- **원인 파악:** R2가 `router bgp 64512`(Sub-AS)로 동작하면서 외부 AS(R1)에게는 여전히 64512로 광고 → R1은 `remote-as 200`을 기대하므로 AS 번호 불일치.
- **해결:** Sub-AS Router에 `bgp confederation identifier 200`을 설정해 외부에는 실제 AS 번호(200)로 보이게 한다. 모든 Confederation Router(R3, R4, R5 포함)에 동일 Identifier를 구성한다.

#### Case D. **Confederation Sub-AS 간 Peer(R3 ↔ R4)가 맺어지지 않음**

- **원인 파악:** Sub-AS 간 연결은 eBGP 취급이므로 `bgp confederation peers`와 Loopback Peer 사용 시 `ebgp-multihop`이 필요하다.
- **해결:** 양쪽에 `bgp confederation peers <상대 Sub-AS>`, `neighbor ... ebgp-multihop 3`을 구성한다.
