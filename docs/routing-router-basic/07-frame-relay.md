# Frame-relay (WAN Layer 2 Protocol)

> **Tag:** #Network #Cisco #WAN #FrameRelay #DLCI #LMI
> 핵심 요약: Frame-relay는 Router–Router 간 Serial Interface를 공유 WAN 인프라(FR-Switch)를 통해 연결하는 Layer 2 프로토콜이다. DLCI로 VC(Virtual Circuit)을 식별하고, LMI로 VC 상태를 유지한다. 주 Interface(Multipoint)와 Sub-interface(P2P·Multipoint) 구성 방식을 구분해야 한다.

---

## Q1. Frame-relay가 뭔가요?

- **핵심 답변:** 하나의 물리 Serial Interface를 공유하여 여러 라우터가 FR-Switch를 통해 논리적 가상회선(VC)으로 연결되는 **WAN Layer 2 프로토콜**입니다.
- WAN Layer 2 Protocol 종류: `HDLC`, `PPP`, `ATM`, `Frame-relay`
- **왜 필요한가:**
  - 전용선(P2P 직결)은 라우터 수만큼 물리 인터페이스가 필요하지만, Frame-relay는 **하나의 Serial Interface로 N개의 원격 사이트를 연결**합니다.
  - 통신사 FR-Switch를 경유하므로 **Layer 2 서비스를 임차**하여 비용을 절감합니다.
- **단점:** 다수 사용자가 대역폭을 공유하므로 트래픽 집중 시 병목이 발생할 수 있습니다.

---

## 동작 원리

### DLCI (Data-Link Connection Identifier)
- DLCI는 **로컬 라우터와 FR-Switch 사이**에서만 의미 있는 로컬 식별자입니다.
- 같은 VC라도 라우터 쪽 DLCI와 상대 라우터 쪽 DLCI가 **다를 수 있습니다.**
- FR-Switch는 `입력 Interface + 입력 DLCI → 출력 Interface + 출력 DLCI` 매핑으로 스위칭합니다.

```text
  R1 ─────[DLCI 102]───── FR-SW ─────[DLCI 201]───── R2
          로컬 DLCI                       로컬 DLCI
```

### VC (Virtual Circuit)
| 종류 | 설명 |
|------|------|
| **PVC** (Permanent VC) | 고정 통신사 설정. 장애가 없는 한 항상 유지. 일반적으로 사용 |
| **SVC** (Switched VC) | 필요 시 동적으로 개설·해제. 현재는 거의 미사용 |

### LMI (Local Management Interface)
- **라우터 ↔ FR-Switch 사이**에서 VC 상태를 주기적으로 확인하는 Keepalive 프로토콜
- 기본값: Cisco IOS는 **AUTO**로 협상 (수동 지정 가능)
- LMI 타입 3종: `cisco` / `ansi` / `q933a`

```cisco
interface serial 1/0
 frame-relay lmi-type [cisco | ansi | q933a]
```

### FR VC 상태 (show 명령에서 확인)
| 상태 | 의미 |
|------|------|
| **Active** | VC 정상 동작 |
| **Inactive** | 로컬 ↔ FR-SW 연결은 되었으나, FR-SW ↔ 원격 라우터 연결 불가 |
| **Deleted** | FR 프레임으로 인식 불가 (LMI Type 불일치 또는 L2 Protocol 불일치) |

---

## 표준 설정

### 구성 방식 비교

| 방식 | 구조 | 특징 |
|------|------|------|
| 주 Interface | 분할하지 않은 원본 인터페이스 | Multipoint와 동일 동작 |
| Sub-interface **Multipoint** | 하나의 서브인터페이스에 다수 VC | 동일 네트워크(NBMA) |
| Sub-interface **Point-to-Point** | 서브인터페이스 1개 = VC 1개 | 서브넷이 각각 독립 |

> 주 Interface = Sub-interface Multipoint (동일 동작)
> Sub-interface P2P ↔ Sub-interface Multipoint 혼용 불가

---

### 1) FR-Switch 설정 (공통)

```cisco
! FR-Switch 역할을 할 라우터에서 설정
no ip routing               ! FR-SW 전용 장비로 사용할 때 IP 포워딩 비활성화
frame-relay switching
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 frame-relay intf-type dce          ! DCE쪽(FR-SW)에서만 지정
 clock rate 1612800
 frame-relay route 102 interface serial 1/1 201   ! DLCI 매핑: 입력 102 → S1/1의 201로 스위칭
 frame-relay route 103 interface serial 1/2 301
!
interface serial 1/1
 no shutdown
 encapsulation frame-relay
 frame-relay intf-type dce
 clock rate 1612800
 frame-relay route 201 interface serial 1/0 102
 frame-relay route 203 interface serial 1/2 302
```

---

### 2) 주 Interface 구성 (Multipoint — Static Map)

> 하나의 인터페이스로 여러 원격지와 같은 서브넷 공유

```cisco
! R1 (주 Interface 방식)
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp          ! Inverse-ARP 비활성화 (Static Map 사용 시 권장)
 ip address 192.168.10.1 255.255.255.0
 frame-relay map ip 192.168.10.2 102 ! 상대 IP ↔ 로컬 DLCI 매핑
 frame-relay map ip 192.168.10.3 103
 frame-relay map ip 192.168.10.4 104
!
```

```cisco
! R2
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
 ip address 192.168.10.2 255.255.255.0
 frame-relay map ip 192.168.10.1 201
 frame-relay map ip 192.168.10.3 203
 frame-relay map ip 192.168.10.4 204
```

---

### 3) Sub-interface Point-to-Point 구성

> 각 VC를 독립 서브넷으로 분리. **Inverse-ARP 자동 불필요 (DLCI 지정 = 직접 매핑)**

```text
R1 ──[S1/0.12 / DLCI 102]── FR-SW ──[DLCI 201]── R2 ──[S1/0.23 / DLCI 203]── FR-SW ──[DLCI 302]── R3
     192.168.12.0/24                                   192.168.23.0/24
```

```cisco
! R1
interface serial 1/0
 no shutdown
 encapsulation frame-relay
!
interface serial 1/0.12 point-to-point        ! 서브인터페이스 + P2P 지정
 frame-relay interface-dlci 102               ! 해당 서브인터페이스에 DLCI 바인딩
 ip address 192.168.12.1 255.255.255.0
!

! R2
interface serial 1/0
 no shutdown
 encapsulation frame-relay
!
interface serial 1/0.12 point-to-point
 frame-relay interface-dlci 201
 ip address 192.168.12.2 255.255.255.0
!
interface serial 1/0.23 point-to-point
 frame-relay interface-dlci 203
 ip address 192.168.23.2 255.255.255.0
!

! R3
interface serial 1/0
 no shutdown
 encapsulation frame-relay
!
interface serial 1/0.23 point-to-point
 frame-relay interface-dlci 302
 ip address 192.168.23.3 255.255.255.0
```

---

### 4) Sub-interface Multipoint 구성 (Hub-and-Spoke)

> Hub(R2)는 Spoke들의 트래픽을 중계. Spoke끼리 직접 VC 없이 Hub 경유

```text
        R1 (Spoke)
       /  DLCI 102
     R2 (Hub) ─── FR-SW     192.168.100.0/24
       \  DLCI 203
        R3 (Spoke)
```

```cisco
! R2 (Hub)
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.123 multipoint
 frame-relay map ip 192.168.100.1 201
 frame-relay map ip 192.168.100.3 203
 ip address 192.168.100.2 255.255.255.0
!

! R1 (Spoke) — Hub 경유로 R3에 접근
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.123 multipoint
 frame-relay map ip 192.168.100.2 102   ! Hub IP → Hub DLCI
 frame-relay map ip 192.168.100.3 102   ! Spoke R3 IP도 Hub DLCI로 전달 (Hub 경유)
 ip address 192.168.100.1 255.255.255.0
!

! R3 (Spoke)
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.123 multipoint
 frame-relay map ip 192.168.100.1 302
 frame-relay map ip 192.168.100.2 302
 ip address 192.168.100.3 255.255.255.0
```

> ⚠️ Hub-and-Spoke에서 Spoke ↔ Spoke 직접 통신은 Hub를 경유해야 합니다. EIGRP/OSPF의 Split-horizon 문제에 주의하십시오.

---

## 검증 명령어 & 출력 판독

```cisco
show frame-relay map           ! IP ↔ DLCI 매핑 및 상태 확인
show frame-relay pvc           ! PVC 상태 (Active/Inactive/Deleted), 패킷 카운터
show frame-relay lmi           ! LMI 통계 (Status-enquiry/update 횟수)
show frame-relay route         ! FR-Switch의 DLCI 스위칭 테이블 (FR-SW에서 실행)
show interfaces serial 1/0     ! Encapsulation, LMI type, 에러 카운터
```

**출력 예시 — 주 Interface (Static Map) `show frame-relay map`**
```text
Serial1/0 (up): ip 192.168.10.2 dlci 102(0x66,0x1860), static,
              CISCO, status defined, active
Serial1/0 (up): ip 192.168.10.3 dlci 103(0x67,0x1870), static,
              CISCO, status defined, active
```
- `static` = `frame-relay map` 명령으로 수동 매핑
- `dynamic` = Inverse-ARP로 자동 학습
- `status: active` = VC 정상 / `inactive` = 원격 불가 / `deleted` = LMI 불일치

**출력 예시 — Sub-interface P2P `show frame-relay map`**
```text
Serial1/0.12 (up): point-to-point dlci, dlci 102(0x66,0x1860), broadcast
                   status defined, active
```
- P2P Sub-interface는 `frame-relay map` 명령 없이도 `broadcast`가 **자동 설정**됨
- `point-to-point dlci`로 표시되며 IP 주소 항목 없음 (1:1 직결이므로 불필요)

**출력 예시 — FR-Switch의 `show frame-relay route`**
```text
Input Intf   Input Dlci   Output Intf   Output Dlci   Status
Serial1/0    102          Serial1/1     201           active
Serial1/1    201          Serial1/0     102           active
```

---

## 대표 장애 & 트러블슈팅

| 증상 | 원인 | 확인 명령 | 조치 |
|------|------|-----------|------|
| VC Status: Deleted | LMI Type 불일치 또는 encapsulation 불일치 | `show frame-relay lmi`, `show int s1/0` | LMI type 맞춤 / encapsulation frame-relay 재설정 |
| VC Status: Inactive | 상대 라우터가 FR-SW에 연결 안됨 | `show frame-relay route` (FR-SW) | 상대 라우터 링크/설정 점검 |
| Ping 불가 (DLCI Active) | `frame-relay map` 누락 또는 DLCI 오기입 | `show frame-relay map` | 정확한 DLCI로 map 재설정 |
| Spoke↔Spoke 통신 불가 | Split-horizon 문제 (EIGRP/RIP) | `show ip route` | Hub에서 `no ip split-horizon eigrp <AS>` 또는 P2P Sub-if 사용 |
| 라우팅 프로토콜 neighbor 미수립 | Multipoint NBMA에서 broadcast 미설정 | `show frame-relay map` | `frame-relay map ip <IP> <DLCI> broadcast` 추가 |

> **진단 순서:** Physical(Serial up?) → Encapsulation → LMI 동기화 → DLCI Status → frame-relay map → 라우팅

---

## 플랫폼별 주의사항

- **Packet Tracer:** FR-Switch를 별도 장비로 구성하거나 라우터에 Frame-relay 서비스 설정이 필요합니다. `frame-relay switching` 지원 여부는 버전마다 다를 수 있습니다.
- **GNS3 / Dynamips:** 실장비에 가까운 동작. `clock rate`는 DCE 쪽(FR-SW)에서만 설정합니다.
- **Inverse-ARP:** 기본 활성화. 자동 학습된 dynamic 매핑과 static map이 **충돌하지 않도록** 일반적으로 `no frame-relay inverse-arp`를 함께 설정합니다.
- Frame-relay는 **레거시 WAN 기술**로, 현대 네트워크에서는 MPLS·SD-WAN으로 대체되었습니다. CCNA/CCNP 시험 범위 확인 후 학습 깊이를 조절하십시오.

---

## 운영망 변경 위험 & Rollback

> ⚠️ Frame-relay Serial Interface 변경(encapsulation, DLCI, LMI) 시 해당 WAN 구간 통신이 즉시 차단됩니다. Console 또는 OOB 접근을 준비하십시오.

**변경 순서:**
1. `show frame-relay map` / `show frame-relay pvc` 현황 저장
2. `copy running-config startup-config` 백업
3. 변경 적용
4. `show frame-relay map` + `ping` 검증
5. 문제 없으면 저장 / 문제 발생 시 저장된 설정으로 복구

---

## 시험 & 면접 포인트

- **DLCI는 로컬 의미**. 양단 DLCI가 달라도 정상입니다.
- `frame-relay map ip <IP> <DLCI> broadcast`: DLCI로 Broadcast 전달 허용 → 라우팅 프로토콜 hello/update 전달 필수
- **Inverse-ARP vs Static Map**: Inverse-ARP는 상대 IP를 자동 학습하지만 제어가 불명확. Static map이 권장됩니다.
- Sub-interface **P2P**는 DLCI 하나당 서브넷 하나 → Split-horizon 문제 없음
- Sub-interface **Multipoint**는 Hub-and-Spoke에서 Spoke끼리 통신 시 Split-horizon 주의
- LMI Type **auto 협상**은 Cisco IOS 기본값이지만, FR-SW 장비에 따라 명시적 설정이 필요할 수 있습니다.

> 관련: 라우팅 기초 개념 · EIGRP · OSPF · GRE Static
> 통합 정리: 8. 🧩 Frame-relay 통합 정리 · 9. 🚑 Frame-relay 트러블슈팅 치트시트 · 10. ⚡ Frame-relay 명령어 퀵 레퍼런스
