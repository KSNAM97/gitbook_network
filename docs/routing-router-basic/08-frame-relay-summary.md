# Frame-relay 통합 정리 — WAN L2 이론·구성 한눈에

> **Tag:** #Network #Cisco #WAN #FrameRelay #DLCI #LMI #Summary
> **핵심 요약:** Frame-relay는 하나의 Serial Interface로 FR-Switch를 통해 N개의 원격지를 연결하는 WAN Layer 2 프로토콜이다. DLCI는 로컬 식별자이며, LMI로 VC 상태를 감시한다. 구성 방식은 주 Interface(Multipoint), Sub-interface P2P, Sub-interface Multipoint 3가지이며 Hub-and-Spoke에서는 Split-horizon 문제에 주의해야 한다.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. Frame-relay가 필요한 이유는?

👉 **핵심 답변:** 라우터 N대를 전용선으로 직접 연결하면 라우터당 N-1개의 물리 Interface가 필요하지만, Frame-relay는 **하나의 Serial Interface로 통신사 FR-Switch를 통해 N개의 원격지를 연결**합니다. Layer 2 서비스를 임차하므로 회선 비용을 절감할 수 있습니다.

```properties
전용선 (Mesh)   : N*(N-1)/2 개의 물리 링크 필요
Frame-relay     : 라우터당 Serial 1개 + FR-Switch 경유
```

---

### Q2. DLCI·LMI·VC의 관계는?

👉 **핵심 답변:** DLCI는 라우터와 FR-Switch 사이 로컬 회선 식별자, LMI는 그 상태를 주기적으로 확인하는 Keepalive 프로토콜, VC는 두 라우터 사이에 논리적으로 형성된 가상 회선입니다.

```text
R1 ─────[DLCI 102]───── FR-SW ─────[DLCI 201]───── R2
         로컬 DLCI                      로컬 DLCI
                ← LMI Keepalive →
```

| 개념 | 범위 | 역할 |
|------|------|------|
| **DLCI** | 로컬 (라우터 ↔ FR-SW) | Virtual Circuit 식별 번호 |
| **LMI** | 라우터 ↔ FR-SW 구간 | VC 상태(Active/Inactive/Deleted) 확인 |
| **PVC** | End-to-End | 고정 가상 회선. 통신사 설정 |
| **SVC** | End-to-End | 동적 개설/해제. 현재 거의 미사용 |

---

### Q3. 구성 방식 3가지 차이점은?

👉 **핵심 답변:** 주 Interface와 Sub-interface Multipoint는 동일하게 동작하며, Sub-interface P2P는 VC 1개당 독립 서브넷을 갖습니다.

| 방식 | 구조 | 서브넷 | Split-horizon |
|------|------|--------|---------------|
| 주 Interface | 분할 없음, NBMA | 전체 공유 | 주의 필요 |
| Sub-if Multipoint | 논리 분할, NBMA | 그룹 공유 | 주의 필요 |
| Sub-if P2P | 논리 분할, P2P | VC당 독립 | 문제 없음 |

> 주 Interface = Sub-interface Multipoint (동일 동작)

---

### Q4. Hub-and-Spoke에서 Split-horizon 문제란?

👉 **핵심 답변:** RIP·EIGRP는 수신한 인터페이스로는 업데이트를 재전송하지 않는 Split-horizon 규칙이 있습니다. Multipoint 구성의 Hub는 Spoke-A에서 받은 경로를 같은 인터페이스를 통해 Spoke-B에 광고하지 않으므로 Spoke↔Spoke 간 라우팅이 안 됩니다.

```text
Spoke-A → Hub (수신)
Hub → Spoke-B ✗ (Split-horizon으로 차단)
```

**해결책:**
1. Sub-interface P2P 사용 (인터페이스가 논리 분리 → Split-horizon 대상 아님)
2. Hub에서 `no ip split-horizon eigrp <AS>` 또는 `no ip split-horizon` 설정

---

## 2. 표준 설정 (Configuration)

### 2-1. FR-Switch 설정 (공통 전제)

```cisco
no ip routing               ! FR-SW 전용 장비일 때 IP 포워딩 비활성화
frame-relay switching
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 frame-relay intf-type dce
 clock rate 1612800
 frame-relay route 102 interface serial 1/1 201
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

### 2-2. 주 Interface — Static Map

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
 ip address 192.168.10.1 255.255.255.0
 frame-relay map ip 192.168.10.2 102
 frame-relay map ip 192.168.10.3 103
```

---

### 2-3. Sub-interface Point-to-Point

```cisco
interface serial 1/0
 no shutdown
 encapsulation frame-relay
!
interface serial 1/0.12 point-to-point
 frame-relay interface-dlci 102
 ip address 192.168.12.1 255.255.255.0
!
interface serial 1/0.13 point-to-point
 frame-relay interface-dlci 103
 ip address 192.168.13.1 255.255.255.0
```

> P2P에서는 `frame-relay map` 불필요. `frame-relay interface-dlci`로 1:1 바인딩.

---

### 2-4. Sub-interface Multipoint — Hub-and-Spoke

```cisco
! Hub (R2)
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

! Spoke (R1) — R3 IP도 Hub DLCI로 전달
interface serial 1/0.123 multipoint
 frame-relay map ip 192.168.100.2 102
 frame-relay map ip 192.168.100.3 102
 ip address 192.168.100.1 255.255.255.0
```

---

### 2-5. 라우팅 프로토콜과 broadcast 키워드

```cisco
frame-relay map ip 192.168.10.2 102 broadcast
```

> `broadcast`를 추가해야 EIGRP Hello, OSPF Hello, RIP Update 등 브로드캐스트/멀티캐스트가 전달됩니다.

---

## 3. 검증 명령어

```cisco
show frame-relay map              ! IP ↔ DLCI 매핑, static/dynamic, 상태
show frame-relay pvc              ! PVC 상태 (Active/Inactive/Deleted), 패킷 카운터
show frame-relay lmi              ! LMI 통계 (Status-enquiry/update 횟수)
show frame-relay route            ! FR-Switch DLCI 스위칭 테이블
show interfaces serial 1/0        ! Encapsulation, LMI type, 에러
```

**VC 상태 요약:**

| 상태 | 의미 |
|------|------|
| **Active** | VC 정상 |
| **Inactive** | 로컬~FR-SW는 OK, FR-SW~원격 불가 |
| **Deleted** | LMI 불일치 또는 L2 설정 오류 |

---

## 4. 대표 함정

| 함정 | 결과 | 조치 |
|------|------|------|
| LMI Type 불일치 | VC Status Deleted | `show frame-relay lmi`로 타입 확인, 명시적 지정 |
| `frame-relay map` 누락 | DLCI Active인데 Ping 실패 | 누락 map 추가 |
| `broadcast` 미설정 | 라우팅 프로토콜 Neighbor 미수립 | map 명령에 `broadcast` 추가 |
| Inverse-ARP + Static Map 혼용 | dynamic/static 충돌 가능 | `no frame-relay inverse-arp` 적용 |
| Multipoint에서 Split-horizon | Spoke↔Spoke 통신 불가 | P2P Sub-if 또는 Split-horizon 비활성화 |
| FR-Switch에서 `ip routing` 미설정 여부 | 라우터로 착각 | `no ip routing` + `frame-relay switching` 확인 |

---

## 5. 핵심 요약

> - DLCI는 **로컬 의미**. 양단 DLCI가 달라도 정상
> - VC 상태 순서: **Active → Inactive → Deleted** (LMI 불일치 시 Deleted)
> - 주 Interface = Sub-if Multipoint (동일 동작)
> - P2P Sub-if = Split-horizon 문제 없음, map 명령 불필요
> - Multipoint = `frame-relay map ip <IP> <DLCI> broadcast` 필수 (라우팅 프로토콜)
> - 진단 순서: **Physical → Encapsulation → LMI → DLCI Status → frame-relay map → 라우팅**
> - 관련: 7. 📡 Frame-relay (WAN L2 Protocol) · 9. 🚑 Frame-relay 트러블슈팅 치트시트 · 10. ⚡ Frame-relay 명령어 퀵 레퍼런스 · EIGRP · OSPF
