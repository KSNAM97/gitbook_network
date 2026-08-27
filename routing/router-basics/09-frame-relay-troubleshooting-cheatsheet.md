# 🚑 Frame-relay 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #FrameRelay #DLCI #LMI #WAN #CheatSheet
> **핵심 요약:** Frame-relay 장애는 **Physical → Encapsulation → LMI 동기화 → DLCI Status → frame-relay map → 라우팅** 순서로 확인한다. DLCI Active인데 통신이 안 되면 map 누락이나 broadcast 미설정을 먼저 의심한다.

---

## 1. 🎯 핵심 진단 흐름

### Q1. 통신 불가 시 어떤 순서로 확인하는가?

👉 **핵심 답변:** 아래 순서를 고정해서 확인합니다.

```properties
[1] Serial Interface up/up
        ↓
[2] encapsulation frame-relay 확인
        ↓
[3] LMI 동기화 (show frame-relay lmi)
        ↓
[4] DLCI Status (Active/Inactive/Deleted)
        ↓
[5] frame-relay map 매핑 확인
        ↓
[6] broadcast 키워드 (라우팅 프로토콜)
        ↓
[7] Split-horizon (Multipoint Hub-and-Spoke)
        ↓
[8] 라우팅 테이블 및 Return Path
```

핵심 명령어:

```cisco
show ip interface brief
show interfaces serial 1/0
show frame-relay lmi
show frame-relay pvc
show frame-relay map
show frame-relay route
show ip route
```

---

### Q2. DLCI Active인데 Ping이 실패하는 이유는?

👉 **핵심 답변:** DLCI Active는 FR-Switch까지의 회선 상태이며, IP 통신을 보장하지 않습니다. 다음 조건을 모두 확인합니다.

```properties
DLCI Active
+ frame-relay map 정확성
+ broadcast 키워드 (라우팅 프로토콜)
+ Split-horizon 영향 없음
+ 라우팅 테이블 경로 존재
+ Return Path 확인
```

---

## 2. 🛠️ 증상별 즉시 대응표

### 2-1. 회선 및 DLCI 상태

| 증상 | 주요 원인 | 확인 명령 | 조치 |
|------|-----------|-----------|------|
| Serial `down/down` | 물리 케이블·DCE Clock 문제 | `show int s1/0`, `show controllers s1/0` | 케이블·Clock 점검 |
| Serial `up/down` | Encapsulation 불일치 또는 Keepalive | `show int s1/0` | `encapsulation frame-relay` 재설정 |
| VC Status: **Deleted** | LMI Type 불일치 또는 encapsulation 오류 | `show frame-relay lmi`, `show int s1/0` | LMI type 명시적 지정, encap 확인 |
| VC Status: **Inactive** | 상대 라우터가 FR-SW에 미연결 | `show frame-relay route` (FR-SW) | 상대 라우터 링크·설정 점검 |
| DLCI Active인데 Ping 실패 | frame-relay map 누락 또는 DLCI 오기입 | `show frame-relay map` | 정확한 DLCI로 map 재설정 |

---

### 2-2. 라우팅 프로토콜

| 증상 | 원인 | 확인 명령 | 조치 |
|------|------|-----------|------|
| EIGRP/OSPF Neighbor 미수립 | `broadcast` 키워드 누락 | `show frame-relay map` | `frame-relay map ip <IP> <DLCI> broadcast` 추가 |
| Spoke→Hub 경로만 있고 Hub→Spoke 없음 | Split-horizon (Multipoint) | `show ip route` | Hub에서 Split-horizon 비활성화 또는 P2P Sub-if 전환 |
| Hub-and-Spoke에서 Spoke↔Spoke 통신 불가 | Split-horizon 또는 map 누락 | `show frame-relay map`, `show ip route` | Spoke에서 Hub DLCI로 상대 Spoke IP map 추가 |

---

### 2-3. Inverse-ARP vs Static Map 충돌

| 증상 | 원인 | 조치 |
|------|------|------|
| `show frame-relay map`에서 static/dynamic 혼재 | Inverse-ARP 활성 + Static Map 동시 적용 | `no frame-relay inverse-arp` 후 Static Map만 사용 |
| dynamic map이 잘못된 IP를 학습 | Inverse-ARP 자동 학습 오류 | Inverse-ARP 비활성화 |

---

### 2-4. FR-Switch 설정 오류

| 증상 | 원인 | 확인 명령 | 조치 |
|------|------|-----------|------|
| 특정 VC만 Active, 나머지 Inactive | `frame-relay route` 매핑 누락 또는 DLCI 오기입 | `show frame-relay route` | 누락 route 추가, DLCI 값 재확인 |
| FR-SW 역할 라우터가 IP 포워딩 | `no ip routing` 미설정 | `show running-config` | `no ip routing` + `frame-relay switching` 확인 |

---

## 3. 🔍 필수 진단 명령어

### Physical / L2

```cisco
show ip interface brief
show interfaces serial 1/0
show controllers serial 1/0          ! DCE/DTE 역할 및 Clock 확인
```

### LMI

```cisco
show frame-relay lmi                 ! LMI Type, Status-enquiry 카운터
```

**LMI 정상 출력 특징:**
- `num status enq sent`와 `num status msgs rcvd`가 증가 중이어야 함
- 값이 멈춰 있으면 LMI 통신 단절

### PVC / DLCI

```cisco
show frame-relay pvc                 ! 모든 PVC 상태 일람
show frame-relay pvc dlci 102        ! 특정 DLCI 상세
```

### Map

```cisco
show frame-relay map                 ! IP ↔ DLCI 매핑 목록
```

**출력 확인 포인트:**
- `static` 또는 `dynamic` 구분
- `active` / `inactive` / `deleted` 상태
- `broadcast` 포함 여부

### FR-Switch

```cisco
show frame-relay route               ! FR-SW에서 DLCI 스위칭 테이블 확인
```

### 라우팅

```cisco
show ip route
show ip route <DESTINATION>
ping <REMOTE_IP>
traceroute <REMOTE_IP>
```

---

## 4. 🚨 트러블슈팅 시나리오

### 시나리오 1. VC Status: Deleted

#### 증상

```text
R1# show frame-relay pvc
DLCI = 102, DLCI USAGE = LOCAL, PVC STATUS = DELETED
```

#### 진단 순서

```cisco
show interfaces serial 1/0         ! encapsulation frame-relay 확인
show frame-relay lmi               ! LMI Type 및 통신 상태
```

#### 원인 및 조치

| 원인 | 조치 |
|------|------|
| encapsulation 미설정 | `encapsulation frame-relay` 적용 |
| LMI Type 불일치 | `frame-relay lmi-type [cisco\|ansi\|q933a]` 명시 |
| FR-Switch에 DLCI 미등록 | FR-SW에서 `frame-relay route` 확인 및 추가 |

---

### 시나리오 2. DLCI Active인데 Ping 실패

#### 증상

```text
R1# show frame-relay pvc
DLCI = 102, PVC STATUS = ACTIVE
R1# ping 192.168.10.2
.....
```

#### 진단

```cisco
show frame-relay map               ! R2 IP에 대한 map 존재 여부
show ip route 192.168.10.2         ! 라우팅 경로 확인
```

#### 원인 및 조치

| 원인 | 조치 |
|------|------|
| `frame-relay map` 누락 | `frame-relay map ip 192.168.10.2 102` 추가 |
| DLCI 값 오기입 | `show frame-relay route`(FR-SW)로 정확한 DLCI 확인 |
| Return Path 누락 | 상대 라우터에서 역방향 map 확인 |

---

### 시나리오 3. Spoke↔Spoke 통신 불가 (Hub-and-Spoke)

#### 증상

```text
R1(Spoke)# ping 192.168.100.3 (R3 Spoke)
.....
```

#### 진단

```cisco
! R1에서
show frame-relay map               ! R3 IP가 Hub DLCI로 map 되어 있는지 확인
show ip route 192.168.100.3

! Hub(R2)에서
show ip route 192.168.100.1
show ip route 192.168.100.3
```

#### 조치

```cisco
! Spoke R1에서 — R3 IP를 Hub DLCI(102)로 전달
interface serial 1/0.123 multipoint
 frame-relay map ip 192.168.100.3 102

! Hub에서 Split-horizon 비활성화 (EIGRP)
interface serial 1/0.123
 no ip split-horizon eigrp 1
```

---

### 시나리오 4. EIGRP Neighbor 미수립 (Multipoint)

#### 원인

```cisco
frame-relay map ip 192.168.100.2 102     ← broadcast 없음
```

#### 조치

```cisco
interface serial 1/0.123 multipoint
 no frame-relay map ip 192.168.100.2 102
 frame-relay map ip 192.168.100.2 102 broadcast
```

검증:

```cisco
show ip eigrp neighbors
show frame-relay map
```

---

## 5. 🔧 현장 Quick Check

```cisco
show ip interface brief
show interfaces serial 1/0
show frame-relay lmi
show frame-relay pvc
show frame-relay map
show frame-relay route
show ip route
ping <REMOTE_IP>
traceroute <REMOTE_IP>
```

---

> 📌 **핵심 요약**
> - 진단 순서: **Physical → Encap → LMI → DLCI Status → map → broadcast → Split-horizon → 라우팅**
> - Deleted = LMI 불일치 또는 encap 오류
> - Inactive = 상대 라우터 링크 문제
> - Active + Ping 실패 = map 누락 or broadcast 미설정
> - Hub-and-Spoke Spoke↔Spoke = Spoke에서 Hub DLCI로 상대 Spoke IP map 추가
> - 관련: 7. 📡 Frame-relay (WAN L2 Protocol) · 8. 🧩 Frame-relay 통합 정리 · 10. ⚡ Frame-relay 명령어 퀵 레퍼런스
