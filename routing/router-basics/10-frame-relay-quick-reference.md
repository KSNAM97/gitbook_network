# Frame-relay 명령어 퀵 레퍼런스

> **Tag:** #Network #QuickReference #FrameRelay #DLCI #LMI #WAN #CheatSheet
> **핵심 요약:** Frame-relay 설정·검증 명령어를 빠르게 조회하기 위한 암기 카드. DLCI는 로컬 식별자이며, Multipoint에서는 라우팅 프로토콜을 위한 `broadcast` 키워드가 필수다.

---

## 1. FR-Switch 설정

```cisco
! FR-Switch 역할 활성화
frame-relay switching
no ip routing                             ! 라우터를 FR-SW 전용으로 사용할 때

! DCE Interface 설정
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 frame-relay intf-type dce
 clock rate 1612800
 frame-relay route <INPUT_DLCI> interface <OUTPUT_IF> <OUTPUT_DLCI>
```

예시:

```cisco
frame-relay route 102 interface serial 1/1 201
frame-relay route 103 interface serial 1/2 301
```

---

## 2. 주 Interface (Multipoint — Static Map)

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
 ip address <IP> <MASK>
 frame-relay map ip <REMOTE_IP> <LOCAL_DLCI> [broadcast]
```

예시 (R1):

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
 ip address 192.168.10.1 255.255.255.0
 frame-relay map ip 192.168.10.2 102 broadcast
 frame-relay map ip 192.168.10.3 103 broadcast
```

---

## 3. Sub-interface Point-to-Point

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
!
interface serial 1/0.<NUMBER> point-to-point
 frame-relay interface-dlci <DLCI>
 ip address <IP> <MASK>
```

예시 (R1 — 두 개의 P2P Sub-if):

```cisco
interface serial 1/0.12 point-to-point
 frame-relay interface-dlci 102
 ip address 192.168.12.1 255.255.255.0
!
interface serial 1/0.13 point-to-point
 frame-relay interface-dlci 103
 ip address 192.168.13.1 255.255.255.0
```

> P2P는 `frame-relay map` 불필요. `frame-relay interface-dlci`로 1:1 바인딩.
> P2P Sub-if에서는 `broadcast`가 **자동 설정**되며, `show frame-relay map` 출력에 `point-to-point dlci ... broadcast`로 표시됨.

---

## 4. Sub-interface Multipoint (Hub-and-Spoke)

### Hub

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.123 multipoint
 ip address <HUB_IP> <MASK>
 frame-relay map ip <SPOKE1_IP> <DLCI_TO_SPOKE1> broadcast
 frame-relay map ip <SPOKE2_IP> <DLCI_TO_SPOKE2> broadcast
```

### Spoke

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.123 multipoint
 ip address <SPOKE_IP> <MASK>
 frame-relay map ip <HUB_IP>   <DLCI_TO_HUB> broadcast
 frame-relay map ip <SPOKE2_IP> <DLCI_TO_HUB> broadcast   ! 상대 Spoke도 Hub DLCI로 전달
```

---

## 5. LMI Type 수동 지정

```cisco
interface serial 1/0
 frame-relay lmi-type [cisco | ansi | q933a]
```

> Cisco IOS 기본값: AUTO 협상. 불일치 시 명시적으로 지정.

---

## 6. Split-horizon 비활성화 (Multipoint Hub)

```cisco
! EIGRP
interface serial 1/0.123
 no ip split-horizon eigrp <AS_NUMBER>

! RIP
interface serial 1/0.123
 no ip split-horizon
```

---

## 7. 검증 명령어

### 7-1. VC / DLCI 상태

```cisco
show frame-relay pvc                     ! 전체 PVC 상태 (Active/Inactive/Deleted)
show frame-relay pvc dlci 102            ! 특정 DLCI 상세
show frame-relay lmi                     ! LMI 통계 (Type, 카운터)
```

**PVC 상태 의미:**

| 상태 | 의미 |
|------|------|
| Active | VC 정상 |
| Inactive | 로컬~FR-SW OK, FR-SW~원격 불가 |
| Deleted | LMI 불일치 또는 encap 오류 |

### 7-2. Map 확인

```cisco
show frame-relay map
```

**출력 확인 포인트:**
- `static` (수동 map) vs `dynamic` (Inverse-ARP 학습)
- `active` / `inactive` / `deleted`
- `broadcast` 포함 여부

### 7-3. FR-Switch 스위칭 테이블

```cisco
show frame-relay route
```

### 7-4. Interface

```cisco
show interfaces serial 1/0               ! Encapsulation, LMI type, 에러 카운터
show ip interface brief                   ! up/down 상태 일람
show controllers serial 1/0              ! DCE/DTE 역할 확인
```

### 7-5. 연결성

```cisco
ping <REMOTE_IP>
ping <REMOTE_IP> source <LOCAL_IF>
traceroute <REMOTE_IP>
show ip route
```

---

## 8. 구성 방식 비교 요약

| 방식 | 명령 | map 필요 | broadcast 필요 | Split-horizon |
|------|------|----------|----------------|---------------|
| 주 Interface | `frame-relay map ip <IP> <DLCI>` | ✅ | 라우팅 프로토콜 시 ✅ | 주의 필요 |
| Sub-if Multipoint | `frame-relay map ip <IP> <DLCI>` | ✅ | 라우팅 프로토콜 시 ✅ | 주의 필요 |
| Sub-if P2P | `frame-relay interface-dlci <DLCI>` | ❌ | 불필요 | 문제 없음 |

---

## 9. 증상별 Quick Check

| 증상 | 먼저 확인 |
|------|-----------|
| Serial up/down | `show int s1/0` — encapsulation 확인 |
| VC Status: Deleted | `show frame-relay lmi` — LMI Type 불일치 |
| VC Status: Inactive | `show frame-relay route` (FR-SW) — 상대 라우터 링크 |
| DLCI Active + Ping 실패 | `show frame-relay map` — map 누락 또는 DLCI 오기입 |
| 라우팅 Neighbor 미수립 | `show frame-relay map` — broadcast 키워드 누락 |
| Spoke↔Spoke 통신 불가 | Spoke에서 상대 Spoke IP → Hub DLCI map 추가 |

---

## 10. 핵심 주의사항

```properties
DLCI = 로컬 식별자 (양단이 달라도 정상)
Multipoint = broadcast 키워드 필수 (라우팅 프로토콜)
Hub-and-Spoke = Spoke에서 상대 Spoke IP도 Hub DLCI로 map
Inverse-ARP = Static Map 사용 시 no frame-relay inverse-arp 권장
P2P Sub-if = frame-relay map 불필요, Split-horizon 문제 없음
```

---

> 📌 **핵심 요약**
> - FR-Switch: `frame-relay switching` + `frame-relay route <IN_DLCI> interface <IF> <OUT_DLCI>`
> - 주 If / Multipoint: `frame-relay map ip <IP> <DLCI> broadcast`
> - P2P Sub-if: `frame-relay interface-dlci <DLCI>`
> - 검증: `show frame-relay pvc` → `show frame-relay map` → `ping`
> - 관련: 7. 📡 Frame-relay (WAN L2 Protocol) · 8. 🧩 Frame-relay 통합 정리 · 9. 🚑 Frame-relay 트러블슈팅 치트시트

c, frame-relay sub-interface는, multi-access 구조, interface로 분할한

```text
R1# show ip route
C    192.168.12.0/24 is directly connected, Serial1/0.12
R1# ping 192.168.12.2

R2# show ip route
C    192.168.12.0/24 is directly connected, Serial1/0.12
C    192.168.12.0/24 is directly connected, Serial1/0.12
C    192.168.23.0/24 is directly connected, Serial1/0.23
R2# ping 192.168.12.1
R2# ping 192.168.23.3

R3# show ip route
R3# show ip route
C    192.168.23.0/24 is directly connected, Serial1/0.23
C    192.168.34.0/24 is directly connected, Serial1/0.34
R3# ping 192.168.23.2
R3# ping 192.168.34.4

R4# show ip route
C    192.168.34.0/24 is directly connected, Serial1/0.34
R4# ping 192.168.34.3

R1# show frame-relay map
-하나의 물리적 Interface를 여러개의 논리적인 Interface로 분할하여 사용하는 기능
  [Frame-relay Sub-interface는 V/C단위로 생성이 가능하다.]

-Frame-relay Sub-interface는 Point-to-point 방식과 Multipoint방식으로 구성이 가능하다.

-주 Interface		: Interface를 분할하지 않은 Interface	[1:N 연결 = Multi-Access 구조]
-주 Interface		: Interface를 분할하지 않은 Interface	[1:N 연결 = Multi-Access 구조]
-Sub-Interface Multipoint	: 논리적인 Interface로 분할한 Interface	[1:N 연결 = Multi-Access 구조]
-Sub-Interface Point-to-point	: 논리적인 Interface로 분할한 Interface  	[1:1 연결 = Point-to-Point  구조]
-주 Interface는 기본적으로 Sub-interface Multipoint와 설정 및 통신방식이 동일하다.

```

> `1) Frame-relay 기본 이론.txt`

```text

-Router와 Router간 Serial Interface를 사용하여 연결시 사용되는 Protocol

-종류	: HDLC , PPP , ATM , Frame-relay등...

-Router와  Router간 연결시 Connected통신시 사용되는 Protocol
-Router와 Router사이에서 Frame-relay Switch를 사용하여 빠른 Swtiching을 통해 데이터를 전송하는 Protocol

-Frame-relay는 DLCI(Data-Link Connection Identifier)라는 식별자 Code를 사용한다.

-Frame-relay LMI (Local Management Interface)
 .Router와 Frame-relay Switch사이에서 Virtual-Circuit의 상태 정보를 교환시 사용되는 Protocol
 .Router는 Frame-relay Switch의 Virtual-Circuit 상태를 확인할 수 없기때문에 직접 연결된 Frame-relay Switch와
  Virtual-Circuit 상태를 주기적으로 확인시 사용된다.
 .기본적인 프레임 릴레이 규격화된 종류로써 DLCI에 대한 상태를 관리하며 
   시그널 메세지를 사용하여 연결 유/무를 주기적으로 확인한다.
 .VC은 수동으로 연결를 실시한 PVC와 자동으로 통신을 실시하는 SVC로 구성이 가능하다.
 .PVC(Permanent VC) : 영구적 가상회선으로 통신이 빈번히 일어나는 구간에서 효율적인 통신
 .SVC(Switched VC)   : 교환식 가상회선으로 간혈적인 통신이 실시되는 구간에서 효율적인 통신

-Frame-relay LMI (Local Management Interface)
```

> `Basic Configuration.txt`

```text
R1# show ip route
     13.0.0.0/24 is subnetted, 2 subnets
C       13.13.1.0 is directly connected, Loopback0
C       13.13.11.0 is directly connected, FastEthernet0/1

R2# show ip route
     13.0.0.0/24 is subnetted, 2 subnets
C       13.13.2.0 is directly connected, Loopback0
C       13.13.12.0 is directly connected, FastEthernet0/1

R3# show ip route
R3# show ip route
     13.0.0.0/24 is subnetted, 2 subnets
C       13.13.3.0 is directly connected, Loopback0
C       13.13.13.0 is directly connected, FastEthernet0/1

R4# show ip route
R4# show ip route
     13.0.0.0/24 is subnetted, 2 subnets
C       13.13.4.0 is directly connected, Loopback0
C       13.13.14.0 is directly connected, FastEthernet0/1

R5# show ip route
R5# show ip route
     13.0.0.0/24 is subnetted, 2 subnets
C       13.13.5.0 is directly connected, Loopback0
C       13.13.15.0 is directly connected, FastEthernet0/1

R1# show ip route
     13.0.0.0/24 is subnetted, 3 subnets
C       13.13.1.0 is directly connected, Loopback0
C       13.13.9.0 is directly connected, Serial1/0.123
C       13.13.11.0 is directly connected, FastEthernet0/1
R1# ping 13.13.9.2
R2# show ip route
     13.0.0.0/24 is subnetted, 3 subnets
C       13.13.2.0 is directly connected, Loopback0
C       13.13.9.0 is directly connected, Serial1/0.123
```

> `Basic Configuration.txt`

```cisco

-R1 ~ R5에 Loopback 0을 생성한 후 13.13.X.X/24 IP 주소를 할당
-R1 ~ R5에 FastEthernet0/1 Interface에 13.13.1X.X/24 IP 주소를 할당

   # R1
interface loopback 0

                               ## R1 , R2 , R3 Frame-relay 연결구간 Connecte 설정

   # R2  [Hub]
interface serial 1/0
 no shutdown

                               ## R1 , R3 Frame-relay 연결구간 Connecte 설정

   # R1
interface serial 1/1
 no shutdown

                               ## R1 , R4 FastEthernet 연결구간 Connecte 설정

    # R1
interface fastethernet 0/0
 no shutdown

                               ## R3 , R5 FastEthernet 연결구간 Connecte 설정

    # R3
interface fastethernet 0/0
 no shutdown
```

---

> 관련: 1.  📖   라우팅(Routing) 기초 개념 · 2. 📍 Static Route (정적 라우팅) · 3. ⚡  라우터의 스위칭 방식 (CEF) · 4. 🧭 라우팅 기초 개념 통합 정리 · 5. 🚑 라우팅 기초 트러블슈팅 치트시트 · 6. ⚡ 라우팅 기초 명령어 퀵 레퍼런스 · 7. 📡 Frame-relay (WAN L2 Protocol) · 8. 🧩 Frame-relay 통합 정리 · 9. 🚑 Frame-relay 트러블슈팅 치트시트