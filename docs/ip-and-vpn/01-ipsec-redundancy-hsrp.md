# IPsec 이중화 (HSRP 연동)

> **Tag:** #network #vpn #ipsec #hsrp #redundancy #high-availability #dpd #cisco
> **핵심 요약:** 본사 VPN 라우터를 이중화하고 HSRP Virtual IP를 VPN Peer로 삼아 무중단 VPN을 구현

> 🔗 선행: 5-3-2. 🔒 IPsec VPN · 7. ⚖️ Gateway 이중화 (FHRP)

---

## 1. 핵심 기술 개념 (Concept)

### 본사 VPN 라우터 2대를 이중화할 때 HSRP Virtual IP를 VPN Peer로 쓰는 이유

원격지가 본사 라우터의 물리 IP를 Peer로 쓰면 Active가 바뀔 때마다 원격지 설정을 변경해야 한다. 두 본사 라우터가 공유하는 HSRP Virtual IP를 Peer로 지정하면, Active가 HQ-1에서 HQ-2로 넘어가도 원격지의 Peer 주소(예: 121.160.10.4)가 그대로라 무중단으로 VPN이 승계된다.

단순히 Virtual IP를 쓰는 것만으로는 안 되고, HSRP 그룹에 이름을 부여하고 `crypto map ... redundancy [이름]`으로 연동해야 Active가 IPsec SA를 대표 처리한다. Standby에는 SA가 안 잡히고 Active에만 `show crypto isakmp peer`가 확인되는 것이 정상이다.

### FHRP만으로 VPN 이중화가 안 되는 이유와 Track이 필수인 이유

HSRP는 게이트웨이 IP만 넘기고 IPsec SA는 자동 이전하지 않으며, 게다가 자기 인터페이스 장애만 감지한다. 그래서 crypto map redundancy로 SA를 연동하고, Track으로 반대편(WAN/LAN) 장애까지 교차 감지해야 진정한 이중화가 된다.

내부 HSRP 그룹은 외부 IF를(`track fa0/1`), 외부 그룹은 내부 IF를(`track fa0/0`) 추적해 어느 쪽이 끊겨도 절체가 일어난다. `crypto isakmp keepalive`(DPD)로 죽은 Peer를 빨리 감지해 재협상을 앞당길 수 있다.

### 공중망(ISP) 구성 시 HQ 이중화에서 OSPF DR을 고정하는 이유

HQ-1, HQ-2, ISP-1이 같은 브로드캐스트 세그먼트에 있으면 DR/BDR 선출이 불안정할 수 있어, ISP-1을 DR로 고정(priority 255)하고 HQ들은 DR 후보에서 제외(priority 0)해 안정적인 인접을 확보한다.

---

## 2. 표준 설정 템플릿 (Configuration)

### 1단계: 공중망 OSPF + DR 고정 (HQ 이중화 구간)

```cisco
! === ISP-1 : DR로 고정 ===
interface fastethernet 0/1
 ip ospf priority 255                       ! 최고 우선순위 → DR
!
! === HQ-1 / HQ-2 : DR 후보 제외 ===
interface fastethernet 0/1
 ip ospf priority 0                         ! DR/BDR 선출 배제
```
### 2단계: 내부/외부 HSRP 이중화 + Track 교차 감지

```cisco
! === HQ-1 (Active) ===
interface fastethernet 0/0                  ! 내부망
 standby 1 ip 192.168.10.254
 standby 1 priority 120
 standby 1 timer 1 3
 standby 1 preempt delay minimum 60
 standby 1 track fastethernet0/1 100         ! 외부(WAN) 장애도 감지
!
interface fastethernet 0/1                  ! 외부망 (IPsec 연동용)
 standby 2 ip 121.160.10.4
 standby 2 priority 120
 standby 2 timer 1 3
 standby 2 preempt delay minimum 60
 standby 2 track fastethernet0/0 100         ! 내부(LAN) 장애도 감지
 standby 2 name IPSEC_R                       ! IPsec 연동용 HSRP 이름 (필수)

! === HQ-2 (Standby) : priority 80, preempt만 ===
interface fastethernet 0/0
 standby 1 ip 192.168.10.254
 standby 1 priority 80
 standby 1 timer 1 3
 standby 1 preempt
!
interface fastethernet 0/1
 standby 2 ip 121.160.10.4
 standby 2 priority 80
 standby 2 timer 1 3
 standby 2 preempt
 standby 2 name IPSEC_R
```

### 3단계: crypto map을 HSRP와 연동 (핵심, 2가지 방식)

```cisco
! === HQ-1, HQ-2 공통 ===
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash md5
 group 2
 lifetime 1800
!
crypto isakmp key 0 cisco address 121.160.10.38
crypto isakmp keepalive 10                   ! DPD: 죽은 Peer 10초 주기 감지
!
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_REDUNDANCY 10 ipsec-isakmp
 set peer 121.160.10.38
 set transform-set IPSEC
 match address 101
!
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
! [방식1] 처음부터 redundancy로 적용
interface fastethernet 0/1
 crypto map IPSEC_REDUNDANCY redundancy IPSEC_R

! [방식2] 이미 적용돼 있으면 제거 후 재적용
interface fastethernet 0/1
 no crypto map IPSEC_REDUNDANCY
 crypto map IPSEC_REDUNDANCY redundancy IPSEC_R
```

### 4단계: 다중 지사 이중화 (지사별 다른 Phase1 정책)

```cisco
! HQ에서 지사(GIT-A/GIT-B)마다 다른 정책·transform-set을 crypto map sequence로 분리
crypto isakmp policy 10                      ! GIT-A용 (3des/sha/DH2)
 encryption 3des
 hash sha
 group 2
crypto isakmp policy 20                      ! GIT-B용 (aes/md5/DH5)
 encryption aes
 hash md5
 group 5
!
crypto isakmp key 0 GIT-HQ-GIT-A address 121.160.10.38
crypto isakmp key 0 GIT-HQ-GIT-B address 121.160.10.42
!
crypto ipsec transform-set GIT-A esp-aes esp-md5-hmac
crypto ipsec transform-set GIT-B esp-aes esp-sha-hmac
!
crypto map IPSEC_HSRP 10 ipsec-isakmp        ! 지사 A
 set peer 121.160.10.38
 set transform-set GIT-A
 match address 101
crypto map IPSEC_HSRP 20 ipsec-isakmp        ! 지사 B
 set peer 121.160.10.42
 set transform-set GIT-B
 match address 102
!
interface fastethernet 0/1
 crypto map IPSEC_HSRP redundancy IPSEC_R
```

### 5단계: 원격 지사(GIT-A)는 Virtual IP를 Peer로 + NAT 예외

```cisco
! === GIT-A ===
crypto isakmp key 0 cisco address 121.160.10.4   ! ← HSRP Virtual IP
crypto isakmp keepalive 10
!
crypto map IPSEC_INT 10 ipsec-isakmp
 set peer 121.160.10.4                            ! ← 물리IP 아닌 Virtual IP
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.20
 crypto map IPSEC_INT
!
! VPN 트래픽 NAT 예외 (deny 먼저)
access-list 110 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.20.0 0.0.0.255 any
ip nat inside source list 110 pool GIT_NAT overload
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Router# show standby brief                   ! Active/Standby + Virtual IP
Router# show standby all                     ! Track 상태 + 실시간 우선순위
Router# show crypto isakmp peer              ! Active에만 Peer 표시
Router# show crypto engine connections active   ! 암/복호화 카운터
Router# show ip ospf neighbor                 ! DR/BDR 확인
```

### 정상 동작 판독

- `show standby brief`: HQ-1이 내부(Grp1)·외부(Grp2) 모두 **Active**, Virtual IP 소유.
- `show crypto isakmp peer`: **Active(HQ-1)에만** Peer 표시, Standby(HQ-2)엔 없음.
- 절체 후: HQ-2가 Active로, 같은 Virtual IP로 Peer가 나타나며 암/복호화 카운터 증가.
### Case 1: Active 절체는 됐는데 VPN이 안 넘어감

- **증상:** HSRP는 HQ-2로 넘어갔으나 원격지와 VPN 불통. 로그에 `%CRYPTO-6-IKMP_MODE_FAILURE`.
- **원인 파악:** crypto map을 **HSRP와 연동(`redundancy [이름]`)하지 않음** 또는 HSRP `name` 미지정. IPsec이 물리 IP에 묶여 절체 안 됨.
- **해결:** `standby [grp] name IPSEC_R` + `crypto map ... redundancy IPSEC_R`를 양쪽 HQ에 적용. `crypto isakmp keepalive`로 죽은 SA 조기 정리.

### Case 2: WAN(외부) 회선만 끊겼는데 절체가 안 됨

- **원인 파악:** HSRP는 자기 인터페이스 장애만 감지 → 반대편 장애를 모름. **Track 미설정.**
- **해결:** 내부 그룹이 외부 IF(`track fa0/1`), 외부 그룹이 내부 IF(`track fa0/0`)를 **교차 추적**하도록 설정. `show standby all`에서 Track state와 차감된 우선순위 확인.

### Case 3: 절체 후 원격지가 Peer를 못 찾음

- **원인 파악:** 원격지가 HQ의 **물리 IP를 Peer로 지정**해 둠. Active가 바뀌면 그 물리 IP는 응답 안 함.
- **해결:** 원격지 `set peer`와 `crypto isakmp key address`를 모두 **HSRP Virtual IP(121.160.10.4)** 로 통일.

### Case 4: OSPF 인접이 불안정하거나 BDR이 자꾸 바뀜

- **원인 파악:** HQ 이중화 구간에서 DR/BDR 선출이 유동적.
- **해결:** ISP를 `ip ospf priority 255`로 DR 고정, HQ들을 `ip ospf priority 0`으로 후보 제외.

```cisco
Interface Grp  Pri P State   	Active	          	Standby              Virtual IP
Fa0/0     1     120 P Active  	local	           	192.168.10.252     192.168.10.254	<--- Virtua-IP에 대해서 Active Router

GIT-HQ-2# show standby brief 
                        P indicates configured to preempt.
Interface Grp  Pri P State    	Active          	Standby	           Virtual IP
Fa0/0     1    80   P Standby	192.168.10.251	local	           192.168.10.254	<--- Virtua-IP에 대해서 Standby Router

GIT-HQ-1# show standby all
FastEthernet0/0 - Group 1			<--- FastEthernet0/0에 설정된 HSRP Group 1의 상세 정보 
  Active virtual MAC address is 0000.0c07.ac01	<--- 현재 Active Router가 사용하는 HSRP Virtual MAC 주소
    Local virtual MAC address is 0000.0c07.ac01 (v1 default)
  Hello time 1 sec, hold time 3 sec		<--- HSRP Hello를 1초마다 전송, 3초 동안 Hello를 받지 못하면 상대 Router 장애로 판단
    Next hello sent in 0.300 secs
  Preemption enabled, delay min 60 secs		<--- Preempt 기능이 활성화 , 우선순위가 높은 Router가 복구되어도 60초 후 Active 역할을 회수함
  Active router is local
  Active virtual MAC address is 0000.0c07.ac01
    Local virtual MAC address is 0000.0c07.ac01 (v1 default)
  Hello time 1 sec, hold time 3 sec
    Next hello sent in 0.160 secs
  Preemption enabled
  Active router is 192.168.10.251, priority 120 (expires in 1.396 sec)
  Active router is 192.168.10.251, priority 120 (expires in 1.396 sec)
  Standby router is local
  Priority 80 (configured 80)
  IP redundancy name is "hsrp-Fa0/0-1" (default)

Interface Grp  Pri P State   Active          Standby         Virtual IP
Fa0/0     1     80  P Active  local           unknown         192.168.10.254

Interface Grp  Pri P State   	Active	       Standby	  Virtual IP
Fa0/0     1     120 P Active  	local	       192.168.10.252	  192.168.10.254	<---- Active Router (HQ-1 내부 네트워크)
Fa0/1     2     120 P Active  	local	       121.160.10.2	  121.160.10.4	<---- Active Router (HQ-1 외부 네트워크)

```

> `2) IPsec 이중화 실습 - 답.txt`

```text
     123.0.0.0/24 is subnetted, 2 subnets
C       123.123.11.0 is directly connected, Loopback0
O       123.123.22.0 [110/65] via 121.160.10.34, 00:00:08, Serial1/0.10
     121.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C       121.160.10.0/29 is directly connected, FastEthernet0/1
C       121.160.10.32/30 is directly connected, Serial1/0.10
O       123.123.11.0 [110/65] via 121.160.10.33, 00:00:02, Serial1/0.10
C       123.123.22.0 is directly connected, Loopback0
     121.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
O       121.160.10.0/29 [110/74] via 121.160.10.33, 00:00:02, Serial1/0.10
C       121.160.10.32/30 is directly connected, Serial1/0.10
C       121.160.10.36/30 is directly connected, Serial1/0.20
C       121.160.10.36/30 is directly connected, Serial1/0.20
C       121.160.10.40/30 is directly connected, Serial1/0.30

GIT-HQ-1# show ip route
C    192.168.10.0/24 is directly connected, FastEthernet0/0
     123.0.0.0/24 is subnetted, 1 subnets
C       123.123.1.0 is directly connected, Loopback0
     121.0.0.0/29 is subnetted, 1 subnets
C       121.160.10.0 is directly connected, FastEthernet0/1
GIT-HQ-2# show ip route
C    192.168.10.0/24 is directly connected, FastEthernet0/0
     123.0.0.0/24 is subnetted, 1 subnets
C       123.123.2.0 is directly connected, Loopback0
     121.0.0.0/29 is subnetted, 1 subnets
C       121.160.10.0 is directly connected, FastEthernet0/1
GIT-A# show ip route
C    192.168.20.0/24 is directly connected, FastEthernet0/0
      123.0.0.0/24 is subnetted, 1 subnets
C        123.123.20.0 is directly connected, Loopback0
      121.0.0.0/30 is subnetted, 1 subnets
C        121.160.10.36 is directly connected, Serial1/0.20
GIT-B# show ip route
C    192.168.30.0/24 is directly connected, FastEthernet0/0
      123.0.0.0/24 is subnetted, 1 subnets
C       123.123.30.0 is directly connected, Loopback0
```

> `2) IPsec 이중화 실습.txt`

```text
     123.0.0.0/24 is subnetted, 2 subnets
C       123.123.11.0 is directly connected, Loopback0
O       123.123.22.0 [110/65] via 121.160.10.34, 00:00:08, Serial1/0.10
     121.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C       121.160.10.0/29 is directly connected, FastEthernet0/1
C       121.160.10.32/30 is directly connected, Serial1/0.10
O       123.123.11.0 [110/65] via 121.160.10.33, 00:00:02, Serial1/0.10
C       123.123.22.0 is directly connected, Loopback0
     121.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
O       121.160.10.0/29 [110/74] via 121.160.10.33, 00:00:02, Serial1/0.10
C       121.160.10.32/30 is directly connected, Serial1/0.10
C       121.160.10.36/30 is directly connected, Serial1/0.20
C       121.160.10.36/30 is directly connected, Serial1/0.20
C       121.160.10.40/30 is directly connected, Serial1/0.30

GIT-HQ-1# show ip route
C    192.168.10.0/24 is directly connected, FastEthernet0/0
     123.0.0.0/24 is subnetted, 1 subnets
C       123.123.1.0 is directly connected, Loopback0
     121.0.0.0/29 is subnetted, 1 subnets
C       121.160.10.0 is directly connected, FastEthernet0/1
GIT-HQ-2# show ip route
C    192.168.10.0/24 is directly connected, FastEthernet0/0
     123.0.0.0/24 is subnetted, 1 subnets
C       123.123.2.0 is directly connected, Loopback0
     121.0.0.0/29 is subnetted, 1 subnets
C       121.160.10.0 is directly connected, FastEthernet0/1
GIT-A# show ip route
C    192.168.20.0/24 is directly connected, FastEthernet0/0
      123.0.0.0/24 is subnetted, 1 subnets
C        123.123.20.0 is directly connected, Loopback0
      121.0.0.0/30 is subnetted, 1 subnets
C        121.160.10.36 is directly connected, Serial1/0.20
GIT-B# show ip route
C    192.168.30.0/24 is directly connected, FastEthernet0/0
      123.0.0.0/24 is subnetted, 1 subnets
C       123.123.30.0 is directly connected, Loopback0
```

> `2) IPsec 이중화 실습 - 답.txt`

```text

EX3) 아래의 조건에 맞게 Default-route를 구성하시오
 .GIT-HQ-1은 외부 네트워크로 통신하기 위해서 ISP-1으로 Default-route를 구성해야한다.
 .GIT-HQ-2는 외부 네트워크로 통신하기 위해서 ISP-1으로 Default-route를 구성해야한다.
 .GIT-A는 외부 네트워크로 통신하기 위해서 ISP-2로 Default-route를 구성해야한다.
 .GIT-B는 외부 네트워크로 통신하기 위해서 ISP-2로 Default-route를 구성해야한다.

EX4-1) 아래의 조건에 맞게 GIT-HQ-1에서 NAT를 구성하시오
 .GIT-HQ-1의 내부 네트워크인 192.168.10.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-HQ-1의 FastEthernet 0/1 Interface의 IP 주소로 변환되어 통신해야한다.

   # GIT-HQ-1  (R1)

EX4-2) 아래의 조건에 맞게 GIT-HQ-2에서 NAT를 구성하시오
 .GIT-HQ-2의 내부 네트워크인 192.168.10.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-HQ-2의 FastEthernet 0/1 Interface의 IP 주소로 변환되어 통신해야한다.

   # GIT-HQ-2  (R2)

EX4-3) 아래의 조건에 맞게 GIT-A에서 NAT를 구성하시오
 .GIT-A의 내부 네트워크인 192.168.20.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-A의 Serial 1/0.20 Interface의 IP 주소로 변환되어 통신해야한다.

   # GIT-A  (R5)

EX4-4) 아래의 조건에 맞게 GIT-B에서 NAT를 구성하시오
 .GIT-B의 내부 네트워크인 192.168.30.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-B의 Serial 1/0.30 Interface의 IP 주소로 변환되어 통신해야한다.

   # GIT-B  (R6)

EX5-1) 아래의 조건에 맞게 IPsec 이중화를 구성하시오
 .GIT-HQ-1 , GIT-HQ-2의 192.168.10.0/24와  GIT-A의 192.168.20.0/24 네트워크간 통신시 정보 보호되어야 한다.

     [Phase 1]
-인증방식 	: 사전 인증방식

EX5-2) 아래의 조건에 맞게 IPsec 이중화를 구성하시오
 .GIT-HQ-1 , GIT-HQ-2의 192.168.10.0/24와  GIT-B의 192.168.30.0/24 네트워크간 통신시 정보 보호되어야 한다.

```

> `2) IPsec 이중화 실습.txt`

```text

EX3) 아래의 조건에 맞게 Default-route를 구성하시오
 .GIT-HQ-1은 외부 네트워크로 통신하기 위해서 ISP-1으로 Default-route를 구성해야한다.
 .GIT-HQ-2는 외부 네트워크로 통신하기 위해서 ISP-1으로 Default-route를 구성해야한다.
 .GIT-A는 외부 네트워크로 통신하기 위해서 ISP-2로 Default-route를 구성해야한다.
 .GIT-B는 외부 네트워크로 통신하기 위해서 ISP-2로 Default-route를 구성해야한다.

EX4-1) 아래의 조건에 맞게 GIT-HQ-1에서 NAT를 구성하시오
 .GIT-HQ-1의 내부 네트워크인 192.168.10.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-HQ-1의 FastEthernet 0/1 Interface의 IP 주소로 변환되어 통신해야한다.

EX4-2) 아래의 조건에 맞게 GIT-HQ-2에서 NAT를 구성하시오
 .GIT-HQ-2의 내부 네트워크인 192.168.10.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-HQ-2의 FastEthernet 0/1 Interface의 IP 주소로 변환되어 통신해야한다.

EX4-3) 아래의 조건에 맞게 GIT-A에서 NAT를 구성하시오
 .GIT-A의 내부 네트워크인 192.168.20.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-A의 Serial 1/0.20 Interface의 IP 주소로 변환되어 통신해야한다.

EX4-4) 아래의 조건에 맞게 GIT-B에서 NAT를 구성하시오
 .GIT-B의 내부 네트워크인 192.168.30.0/24 네트워크가 외부 네트워크로 통신시 
  GIT-B의 Serial 1/0.30 Interface의 IP 주소로 변환되어 통신해야한다.

EX5-1) 아래의 조건에 맞게 IPsec 이중화를 구성하시오
 .GIT-HQ-1 , GIT-HQ-2의 192.168.10.0/24와  GIT-A의 192.168.20.0/24 네트워크간 통신시 정보 보호되어야 한다.

     [Phase 1]
-인증방식 	: 사전 인증방식

EX5-2) 아래의 조건에 맞게 IPsec 이중화를 구성하시오
 .GIT-HQ-1 , GIT-HQ-2의 192.168.10.0/24와  GIT-B의 192.168.30.0/24 네트워크간 통신시 정보 보호되어야 한다.

```

---
