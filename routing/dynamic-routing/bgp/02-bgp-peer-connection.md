# BGP Peer 연결 (Physical · Loopback · Multihop)

> **Tag:** #Network #Cisco #BGP #Routing #iBGP #eBGP #UpdateSource #Multihop #Loopback
> **핵심 요약:** 물리적 Interface와 논리적 Interface(Loopback)를 사용한 iBGP/eBGP Peer 연결 방법. **iBGP는 Multihop 기본값이 255**라 Loopback 연결이 바로 되지만, **eBGP는 기본값이 1**이라 Loopback 연결 시 `ebgp-multihop`을 반드시 조정해야 한다.

---

## 1. 핵심 기술 개념 (Concept)

### 실습 토폴로지

```text
       Loopback 0                         Loopback 0                       Loopback 0                       Loopback 0                         Loopback 0
       13.13.1.0/24                        13.13.2.0/24                      13.13.3.0/24                      13.13.4.0/24                        13.13.5.0/24

                           13.13.12.0/24                    13.13.23.0/24                     13.13.34.0/24                    13.13.45.0/24
               R1------------------------R2------------------------R3------------------------R4------------------------R5
                           Serial 1/0.12 P                 Serial 1/0.23 P                  Serial 1/0.34 P                    Serial 1/0.45 P

   |--- BGP AS 100 ---|        |--------------------------------------- BGP AS 200 ---------------------------------------|
```

- R1은 BGP AS 100, R2 ~ R5는 BGP AS 200에 포함된다.
- 모든 Router는 Frame-Relay Point-to-Point Subinterface(Serial 1/0.XY)로 연결되어 있고, 사전에 EIGRP 100으로 Loopback 0 및 연결 구간이 라우팅되어 있다. (Pre-config는 문서 하단 **부록** 참고)

### 물리적 Interface vs 논리적 Interface(Loopback) Peer 연결

| 구분 | 물리적 Interface | 논리적 Interface (Loopback) |
|------|------------------|------------------------------|
| neighbor 대상 | 직접 연결된 Serial IP (예: 13.13.12.2) | 상대 Loopback 0 IP (예: 13.13.2.2) |
| `update-source` | 불필요 | **필요** (`update-source loopback 0`) |
| iBGP | 바로 연결 | 별도 설정 없이 연결 (Multihop 기본값 255) |
| eBGP | 바로 연결 | `ebgp-multihop` 조정 필수 (Multihop 기본값 1) |
| 사전 조건 | 직접 연결 구간 | IGP 등으로 상대 Loopback IP에 대한 Routing이 확립되어야 함 |

### BGP Multihop

- BGP multihop은 최대 **255**까지 가능하다.
- IGP에서는 Connected가 0 hop으로 계산되지만 **BGP에서는 Connected가 1 hop**으로 계산된다.
- **iBGP는 Multihop의 기본값이 255**로 설정되어 있기 때문에 가상 Interface인 Loopback을 사용하여 BGP Peer 연결 시 별도의 설정 없이 연결이 가능하다.
- **eBGP는 Multihop의 기본값이 1**로 설정되어 있기 때문에 가상 Interface인 Loopback을 사용하여 BGP Peer 연결 시 Multihop을 변경해야 한다.

### BGP Routing Update (`network` command)

- BGP Routing Protocol을 사용하여 라우팅 시 IGP와 동일하게 `network` command를 사용한다.
- 단, Subnetting된 네트워크는 **`mask` 옵션**을 사용하여 Subnetting된 Subnet Mask를 설정해야 한다.
- eBGP로 수신한 네트워크는 AD **20**, iBGP로 수신한 네트워크는 AD **200** 으로 Routing Table에 등록되며, Metric 정보는 포함되지 않는다. (`[20/0]`, `[200/0]`)

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. 물리적 Interface를 사용한 BGP Peer 연결

#### EX1) R1과 R2 간 eBGP Peer를 연결하시오

- R1은 BGP AS 100, R2는 BGP AS 200에 포함되어야 한다.
- BGP Peer 연결 시 물리적인 Interface를 사용해야 한다.

```cisco
! === R1 ===
router bgp 100
 bgp router-id 1.1.1.1
 no auto-summary
 no synchronization
 neighbor 13.13.12.2 remote-as 200
!

! === R2 ===
router bgp 200
 bgp router-id 2.2.2.2
 no auto-summary
 no synchronization
 neighbor 13.13.12.1 remote-as 100
!
```

**정보 확인**

```cisco
R1# show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 100
BGP table version is 1, main routing table version 1

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
13.13.12.2      4     200          2          2          0      0       0  00:00:15          0
*Mar  1 00:51:07.839: %BGP-5-ADJCHANGE: neighbor 13.13.12.2 Up
```

```cisco
R2# show ip bgp summary
BGP router identifier 2.2.2.2, local AS number 200
BGP table version is 1, main routing table version 1

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
13.13.12.1      4     100          0          0          0      0       0  00:01:44           0
R2#
*Mar  1 00:43:00.971: %BGP-5-ADJCHANGE: neighbor 13.13.12.1 Up
```

---

#### EX2) BGP AS 200에서 아래의 조건에 맞게 BGP Peer를 연결하시오

- R2와 R3 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 물리적인 Interface를 사용해야 한다.
- R3과 R4 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 물리적인 Interface를 사용해야 한다.
- R4와 R5 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 물리적인 Interface를 사용해야 한다.

```cisco
! === R2 ===
router bgp 200
 bgp router-id 2.2.2.2
 no auto-summary
 no synchronization
 neighbor 13.13.12.1 remote-as 100
 neighbor 13.13.23.3 remote-as 200
!

! === R3 ===
router bgp 200
 bgp router-id 3.3.3.3
 no auto-summary
 no synchronization
 neighbor 13.13.23.2 remote-as 200
 neighbor 13.13.34.4 remote-as 200
!

! === R4 ===
router bgp 200
 bgp router-id 4.4.4.4
 no auto-summary
 no synchronization
 neighbor 13.13.34.3 remote-as 200
 neighbor 13.13.45.5 remote-as 200
!

! === R5 ===
router bgp 200
 bgp router-id 5.5.5.5
 no auto-summary
 no synchronization
 neighbor 13.13.45.4 remote-as 200
!
```

> ⚠️ **가장 흔한 실수 정정:** 원본 일부 버전의 R2 설정에는 `neighbor 13.13.23.2 remote-as 200`(자기 자신의 Serial IP)로 기재되어 있으나, R3의 Serial IP인 **`13.13.23.3`** 이 올바르다. (정보 확인 출력도 13.13.23.3 기준)

**정보 확인**

```cisco
R2# show ip bgp summary 	[eBGP Peer 1개 , iBGP Peer 1개 확인]
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.12.1      4    100           4          3           1     0      0 00:12:30           0
13.13.23.3      4    200           7          7           1     0      0 00:00:21           0

R3# show ip bgp summary  	[iBGP Peer 2개 확인]
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.23.2      4    200           7          7           1     0      0 00:07:32           0
13.13.34.4      4    200           9          9           1     0      0 00:00:21           0

R4# show ip bgp summary  	[iBGP Peer 2개 확인]
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.34.3      4    200           7          7           1     0      0 00:07:32           0
13.13.45.5      4    200           9          9           1     0      0 00:00:21           0

R5# show ip bgp summary   	[iBGP Peer 1개 확인]
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.45.4      4    200           9          9           1     0      0 00:00:21           0
```

---

### 2-2. BGP Routing Update (`network` command)

#### EX3) 아래의 조건에 맞게 R1을 구성하시오

- R1에 Loopback 10을 생성한 후 `10.1.1.0/24` IP 주소를 할당하시오
- R1에 Loopback 172를 생성한 후 `172.16.1.0/24` IP 주소를 할당하시오
- R1에 Loopback 192를 생성한 후 `192.168.1.0/24` IP 주소를 할당하시오
- Loopback 10, Loopback 172, Loopback 192 네트워크를 BGP로 라우팅 업데이트해야 한다.

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
 network 10.1.1.0  mask  255.255.255.0		<---- Subnetting된 네트워크 (원본 = A class)
 network 172.16.1.0  mask  255.255.255.0	<---- Subnetting된 네트워크 (원본 = B class)
 network 192.168.1.0				<---- 원본 C class 네트워크
!
```

**정보 확인**

```cisco
R1# show ip route connected
      172.16.0.0/24 is subnetted, 1 subnets
C         172.16.1.0 is directly connected, Loopback172	<---- R1에서 Connected로 확인
      10.0.0.0/24 is subnetted, 1 subnets
C         10.1.1.0 is directly connected, Loopback10     	<---- R1에서 Connected로 확인
C    192.168.1.0/24 is directly connected, Loopback192	<---- R1에서 Connected로 확인
      13.0.0.0/24 is subnetted, 9 subnets
C         13.13.1.0 is directly connected, Loopback0
C         13.13.12.0 is directly connected, Serial1/0.12
```

```cisco
R2# show ip route bgp
      172.16.0.0/24 is subnetted, 1 subnets
B         172.16.1.0 [20/0] via 13.13.12.1, 00:03:59		<---- eBGP로 수신한 네트워크  (eBGP AD = 20 , Metric정보는 포함되지 않는다.)
      10.0.0.0/24 is subnetted, 1 subnets
B         10.1.1.0 [20/0] via 13.13.12.1, 00:03:59		<---- eBGP로 수신한 네트워크  (eBGP AD = 20 , Metric정보는 포함되지 않는다.)
B    192.168.1.0/24 [20/0] via 13.13.12.1, 00:03:59		<---- eBGP로 수신한 네트워크  (eBGP AD = 20 , Metric정보는 포함되지 않는다.)
```

```cisco
R3# show ip route bgp
      172.16.0.0/24 is subnetted, 1 subnets
B         172.16.1.0 [200/0] via 13.13.12.1, 00:05:16		<---- iBGP로 수신한 네트워크  (iBGP AD = 200 , Metric정보는 포함되지 않는다.)
      10.0.0.0/24 is subnetted, 1 subnets
B         10.1.1.0 [200/0] via 13.13.12.1, 00:05:16		<---- iBGP로 수신한 네트워크  (iBGP AD = 200 , Metric정보는 포함되지 않는다.)
B    192.168.1.0/24 [200/0] via 13.13.12.1, 00:05:16	<---- iBGP로 수신한 네트워크  (iBGP AD = 200 , Metric정보는 포함되지 않는다.)
```

```cisco
R4# show ip route bgp	(BGP Split-horizon에 의해 BGP 네트워크를 수신하지 못한다.)

R5# show ip route bgp	(BGP Split-horizon에 의해 BGP 네트워크를 수신하지 못한다.)
```

> 💡 R4, R5가 BGP 네트워크를 수신하지 못하는 현상은 **BGP Split-horizon 문제 해결** 문서에서 3가지 방식(Full-mesh / Route-reflector / Confederation)으로 해결한다.

**정보 확인 후 BGP 설정 삭제**

```cisco
! === R1 ===
no router bgp 100
!
! === R2 ===
no router bgp 200
!
! === R3 ===
no router bgp 200
!
! === R4 ===
no router bgp 200
!
! === R5 ===
no router bgp 200
!
```

---

### 2-3. 논리적 Interface(Loopback)를 사용한 BGP Peer 연결

#### EX1) BGP AS 200에서 아래의 조건에 맞게 BGP Peer를 연결하시오

- BGP Router-ID = X.X.X.X (X = Router 번호)
- R2와 R3 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 **논리적인 Interface(Loopback 0)** 를 사용해야 한다.
- R3과 R4 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 **논리적인 Interface(Loopback 0)** 를 사용해야 한다.
- R4와 R5 간 iBGP Peer를 연결하며 iBGP Peer 연결 시 **논리적인 Interface(Loopback 0)** 를 사용해야 한다.

> 원본 지시문에는 "물리적인 Interface를 사용해야한다"로 기재되어 있으나, 이 실습은 Loopback 0 IP로 Peer를 연결하는 예제다. (설정의 `update-source loopback 0` 참고)

```cisco
! === R2 ===
router bgp 200
 bgp router-id 2.2.2.2
 no auto-summary
 no synchronization
 neighbor 13.13.3.3  remote-as 200
 neighbor 13.13.3.3  update-source  loopback 0
!

! === R3 ===
router bgp 200
 bgp router-id 3.3.3.3
 no auto-summary
 no synchronization
 neighbor 13.13.2.2  remote-as 200
 neighbor 13.13.2.2  update-source  loopback 0
 neighbor 13.13.4.4  remote-as 200
 neighbor 13.13.4.4  update-source  loopback 0
!

! === R4 ===
router bgp 200
 no auto-summary
 no synchronization
 bgp router-id 4.4.4.4
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source loopback 0
!

! === R5 ===
router bgp 200
 no auto-summary
 no synchronization
 bgp router-id 5.5.5.5
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source loopback 0
!
```

**정보 확인**

```cisco
R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.3.3        4    200         16         15           1    0      0  00:12:29           0

R3# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4    200         15         16           1    0      0  00:12:29           0
13.13.4.4        4    200         18         17           1    0      0  00:14:21           0

R4# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.3.3        4    200         17         18           1    0      0  00:14:43           0
13.13.5.5        4    200         22         23           1    0      0  00:15:12           0

R5# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.4.4        4    200         23         22           1    0      0  00:15:12           0
```

---

#### EX2) 아래의 조건에 맞게 BGP Peer를 연결하시오

- R1의 BGP AS 100과 R2의 BGP AS 200 구간 사이에서 eBGP Peer를 연결하시오
- eBGP Peer 연결 시 Loopback 0의 IP 주소를 사용하여 연결해야 한다.

```cisco
! === R1 ===
router bgp 100
 no auto-summary
 no synchronization
 bgp router-id 1.1.1.1
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
!

! === R2 ===
router bgp 200
 neighbor 13.13.1.1 remote-as 100
 neighbor 13.13.1.1 update-source loopback 0
!
```

설정 후 `show ip bgp summary` command를 사용하여 BGP Peer 연결 상태를 확인하면 **R1과 R2 eBGP Peer가 연결되지 않는다.**

```cisco
R1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200            0          0          0     0       0 never       Idle

R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4   100            0          0          0     0       0 never       Idle
13.13.3.3        4   200            9          9          1     0       0 00:09:44           0
```

**원인:** eBGP는 Multihop 기본값이 1이라, Loopback IP로 Peer를 연결하면 TTL 부족으로 연결되지 않는다. → `ebgp-multihop` 조정이 필요하다.

**해결: `ebgp-multihop 3` 적용** (default = 1)

```cisco
! === R1 ===
router bgp 100
 no auto-summary
 no synchronization
 bgp router-id 1.1.1.1
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
 neighbor 13.13.2.2 ebgp-multihop 3		<---- eBGP Peer 연결 시 BGP Multihop을 3으로 변경 (default = 1)
!

! === R2 ===
router bgp 200
 neighbor 13.13.1.1 remote-as 100
 neighbor 13.13.1.1 update-source loopback 0
 neighbor 13.13.1.1 ebgp-multihop 3		<---- eBGP Peer 연결 시 BGP Multihop을 3으로 변경 (default = 1)
!
```

**정보 확인**

```cisco
R1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.2.2        4   200         18         17           1     0      0  00:03:58           0

R2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.13.1.1        4   100         18         18           1     0       0 00:04:58           0
13.13.3.3        4   200         46         43           1     0       0 00:40:53           0
```

---

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
R2# show ip bgp

R3# show ip bgp summary
R3# show ip bgp

R4# show ip bgp summary
R4# show ip bgp

R5# show ip bgp summary
R5# show ip bgp
```

> 💡 R3까지는 iBGP로 네트워크가 전달되지만 R4, R5에서는 **BGP Split-horizon**으로 수신하지 못한다. 이후 실습은 **BGP Split-horizon 문제 해결** 문서에서 이어진다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```cisco
R1# show ip bgp summary          ! Peer 상태 (State/PfxRcd) 확인
R1# show ip bgp                  ! BGP Table 확인
R1# show ip route bgp            ! Routing Table의 BGP 경로 (B) 확인
R1# show ip route connected      ! Connected 네트워크 확인
```

### 3-2. Case별 트러블슈팅 가이드

#### Case A. **Loopback eBGP Peer가 Idle에서 올라오지 않음**

- **증상:** `show ip bgp summary`에서 State가 `Idle`, Up/Down이 `never`.
- **원인 파악:** eBGP Multihop 기본값이 1이라 Loopback IP(1 hop 초과)로는 Peer가 맺어지지 않는다.
- **해결:** 양쪽 모두 `neighbor <IP> ebgp-multihop <N>` 설정 (실습에서는 3).

#### Case B. **iBGP Loopback Peer는 되는데 AS 간(eBGP)만 안 됨**

- **원인 파악:** iBGP는 Multihop 기본값이 255라 `update-source loopback 0`만으로 연결되지만, eBGP는 별도 `ebgp-multihop` 설정이 필요하다.
- **해결:** eBGP 구간에만 `ebgp-multihop`을 추가한다.

#### Case C. **BGP로 수신한 네트워크가 R4·R5까지 전달되지 않음**

- **원인 파악:** iBGP로 수신한 Route는 다른 iBGP Peer에게 전달하지 않는 BGP Split-horizon 규칙.
- **해결:** Full-mesh / Route-reflector / Confederation 중 하나를 적용한다. (BGP Split-horizon 문제 해결 문서 참고)

---

## 부록. 실습 Pre-config

### Pre-config I (R1 ~ R5, EIGRP 100)

```cisco
     # R1

en
conf t
!
no ip domain-lookup
!
enable secret cisco
!
hostname R1
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.1.1 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.12 point-to-point
 ip add 13.13.12.1 255.255.255.0
 frame-relay interface-dlci 102
!
router eigrp 100
 no auto-summary
 network 13.13.1.1  0.0.0.0
 network 13.13.12.1  0.0.0.0
end
!

     # R2

en
conf t
!
no ip domain-lookup
!
enable secret cisco
!
hostname R2
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.2.2 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.12 point-to-point
 ip add 13.13.12.2 255.255.255.0
 frame-relay interface-dlci 201
!
interface serial 1/0.23 point-to-point
 ip add 13.13.23.2 255.255.255.0
 frame-relay interface-dlci 203
!
router eigrp 100
 no auto-summary
 network 13.13.2.2  0.0.0.0
 network 13.13.12.2  0.0.0.0
 network 13.13.23.2  0.0.0.0
!
end
!

     # R3

en
conf t
!
no ip domain-lookup
!
enable secret cisco
!
hostname R3
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.3.3 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.23 point-to-point
 ip add 13.13.23.3 255.255.255.0
 frame-relay interface-dlci 302
!
interface serial 1/0.34 point-to-point
 ip add 13.13.34.3 255.255.255.0
 frame-relay interface-dlci 304
!
router eigrp 100
 no auto-summary
 network 13.13.3.3  0.0.0.0
 network 13.13.23.3  0.0.0.0
 network 13.13.34.3  0.0.0.0
end
!

     # R4

en
conf t
!
no ip domain-lookup
!
hostname R4
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.4.4 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.34 point-to-point
 ip add 13.13.34.4 255.255.255.0
 frame-relay interface-dlci 403
!
interface serial 1/0.45 point-to-point
 ip add 13.13.45.4 255.255.255.0
 frame-relay interface-dlci 405
!
router eigrp 100
 no auto-summary
 network 13.13.4.4  0.0.0.0
 network 13.13.34.4  0.0.0.0
 network 13.13.45.4  0.0.0.0
end
!

     # R5

en
conf t
!
no ip domain-lookup
!
hostname R5
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.5.5 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.45 point-to-point
 ip add 13.13.45.5 255.255.255.0
 frame-relay interface-dlci 504
!
router eigrp 100
 no auto-summary
 network 13.13.5.5  0.0.0.0
 network 13.13.45.5  0.0.0.0
!
end
!
```

### Pre-config II (R1 ~ R7 확장 토폴로지, EIGRP 100)

- Pre-config I과 달리 R5에 `Serial 1/0.56`, R6, R7이 추가된 7대 구성이다.
- R1 ~ R4 구성은 Pre-config I과 동일하되 **R1은 EIGRP 설정이 없고**, **R2는 `13.13.12.2` 네트워크를 EIGRP에 포함하지 않는다.** (R1–R2 구간을 DMZ로 사용하기 위한 구성)

```cisco
     # R1

en
conf t
!
no ip domain-lookup
!
enable secret cisco
!
hostname R1
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.1.1 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.12 point-to-point
 ip add 13.13.12.1 255.255.255.0
 frame-relay interface-dlci 102
!

     # R2

en
conf t
!
no ip domain-lookup
!
enable secret cisco
!
hostname R2
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.2.2 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.12 point-to-point
 ip add 13.13.12.2 255.255.255.0
 frame-relay interface-dlci 201
!
interface serial 1/0.23 point-to-point
 ip add 13.13.23.2 255.255.255.0
 frame-relay interface-dlci 203
!
router eigrp 100
 no auto-summary
 network 13.13.2.2  0.0.0.0
 network 13.13.23.2  0.0.0.0
!
end
!

     # R3

en
conf t
!
no ip domain-lookup
!
enable secret cisco
!
hostname R3
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.3.3 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.23 point-to-point
 ip add 13.13.23.3 255.255.255.0
 frame-relay interface-dlci 302
!
interface serial 1/0.34 point-to-point
 ip add 13.13.34.3 255.255.255.0
 frame-relay interface-dlci 304
!
router eigrp 100
 no auto-summary
 network 13.13.3.3  0.0.0.0
 network 13.13.23.3  0.0.0.0
 network 13.13.34.3  0.0.0.0
end
!

     # R4

en
conf t
!
no ip domain-lookup
!
hostname R4
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.4.4 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.34 point-to-point
 ip add 13.13.34.4 255.255.255.0
 frame-relay interface-dlci 403
!
interface serial 1/0.45 point-to-point
 ip add 13.13.45.4 255.255.255.0
 frame-relay interface-dlci 405
!
router eigrp 100
 no auto-summary
 network 13.13.4.4  0.0.0.0
 network 13.13.34.4  0.0.0.0
 network 13.13.45.4  0.0.0.0
end
!

     # R5

en
conf t
!
no ip domain-lookup
!
hostname R5
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.5.5 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.45 point-to-point
 ip add 13.13.45.5 255.255.255.0
 frame-relay interface-dlci 504
!
interface serial 1/0.56 point-to-point
 ip add 13.13.56.5 255.255.255.0
 frame-relay interface-dlci 506
!
router eigrp 100
 no auto-summary
 network 13.13.5.5  0.0.0.0
 network 13.13.45.5  0.0.0.0
end
!

     # R6

en
conf t
!
no ip domain-lookup
!
hostname R6
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.6.6 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.56 point-to-point
 ip add 13.13.56.6 255.255.255.0
 frame-relay interface-dlci 605
!
interface serial 1/0.67 point-to-point
 ip add 13.13.67.6 255.255.255.0
 frame-relay interface-dlci 607
!
router eigrp 100
 no auto-summary
 network 13.13.6.6  0.0.0.0
 network 13.13.67.6  0.0.0.0
end
!

     # R7

en
conf t
!
no ip domain-lookup
!
hostname R7
!
line con 0
 logging syn
 exec-timeout 0 0
 password cisco
 login
!
line vty 0 4
 password cisco
 login
!
int lo 0
 ip add 13.13.7.7 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.67 point-to-point
 ip add 13.13.67.7 255.255.255.0
 frame-relay interface-dlci 706
!
router eigrp 100
 no auto-summary
 network 13.13.7.7  0.0.0.0
 network 13.13.67.7  0.0.0.0
end
!
```
