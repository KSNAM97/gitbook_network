# ⚡ OSPF 명령어 퀵 레퍼런스

> **Tag:** #Routing #OSPF #QuickReference #Area #LSA #CheatSheet  
> **핵심 요약:** OSPF 설정, Area, LSA, 인증, 요약 및 검증 명령어를 빠르게 조회하기 위한 명령어 카드.

---

## 1. 🛠️ 기본 설정

### 1-1. Router Process 방식

```cisco
router ospf 100
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Serial1/0
 network 13.13.12.1 0.0.0.0 area 0
 network 172.16.1.1 0.0.0.0 area 0
 auto-cost reference-bandwidth 10000
```

### 1-2. 인터페이스 방식

```cisco
interface Serial1/0
 ip ospf 100 area 0
```

### 1-3. Cost 지정

```cisco
interface Serial1/0
 ip ospf cost 10
```

### 1-4. Loopback를 원래 Prefix Length로 광고

```cisco
interface Loopback172
 ip address 172.16.1.1 255.255.255.0
 ip ospf 100 area 0
 ip ospf network point-to-point
```

---

## 2. 🔐 인증

### 2-1. Area 단위 MD5 인증

```cisco
router ospf 100
 area 0 authentication message-digest
!
interface Serial1/0
 ip ospf message-digest-key 100 md5 <KEY>
```

### 2-2. 인터페이스 단위 MD5 인증

```cisco
interface Serial1/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 100 md5 <KEY>
```

### 2-3. 인증 제거

```cisco
interface Serial1/0
 no ip ospf authentication
 no ip ospf message-digest-key 100 md5
```

> MD5는 레거시 호환 방식이다. 지원 플랫폼에서는 더 강한 인증 방식을 검토한다.

---

## 3. 🕸️ Network Type 및 DR

### 3-1. Network Type

```cisco
interface <interface>
 ip ospf network broadcast
 ip ospf network non-broadcast
 ip ospf network point-to-point
 ip ospf network point-to-multipoint
 ip ospf network point-to-multipoint non-broadcast
```

### 3-2. DR Priority

```cisco
interface <interface>
 ip ospf priority 255
```

```cisco
interface <interface>
 ip ospf priority 0
```

- 0: DR/BDR 후보 제외
- 1: 기본값
- 높은 Priority가 우선
- 동률이면 높은 Router-ID가 우선
- DR 선출은 비선점 방식

### 3-3. NBMA Neighbor

```cisco
router ospf 100
 neighbor 13.13.9.1
 neighbor 13.13.9.3
```

---

## 4. ⏱️ Timer 및 대역폭

```cisco
interface Serial1/0
 ip ospf hello-interval 10
 ip ospf dead-interval 40
```

```cisco
router ospf 100
 auto-cost reference-bandwidth 10000
```

| Network Type | 기본 Hello/Dead |
|---|---:|
| Broadcast | 10 / 40초 |
| Point-to-Point | 10 / 40초 |
| Point-to-Multipoint | 30 / 120초 |
| Non-Broadcast | 30 / 120초 |

> 플랫폼과 기능에 따라 기본값이 다를 수 있으므로 `show ip ospf interface`로 실제 값을 확인한다.

---

## 5. 🗂️ Area 설정

### 5-1. Stub

```cisco
router ospf 100
 area 14 stub
```

Area 내부 모든 라우터에 설정한다.

### 5-2. Totally Stub

```cisco
! ABR
router ospf 100
 area 14 stub no-summary
```

내부 라우터에는 일반 `area 14 stub`을 설정한다.

### 5-3. NSSA

```cisco
router ospf 100
 area 14 nssa
```

Area 내부 모든 라우터에 설정한다.

### 5-4. Totally NSSA

```cisco
! ABR
router ospf 100
 area 14 nssa no-summary
```

### 5-5. NSSA 기본 경로

```cisco
router ospf 100
 area 14 nssa default-information-originate
```

---

## 6. 📦 요약 및 재분배

### 6-1. ABR Inter-Area 요약

```cisco
router ospf 100
 area 13 range 100.100.0.0 255.255.248.0
```

### 6-2. ASBR External 요약

```cisco
router ospf 100
 summary-address 200.200.8.0 255.255.248.0
```

### 6-3. EIGRP 재분배

```cisco
router ospf 100
 redistribute eigrp 100 metric-type 1 subnets
```

### 6-4. Static 재분배

```cisco
router ospf 100
 redistribute static metric-type 1 subnets
```

### 6-5. 기본 경로 광고

```cisco
router ospf 100
 default-information originate
```

항상 기본 경로를 광고하려면:

```cisco
router ospf 100
 default-information originate always
```

---

## 7. 🌉 Virtual Link

```cisco
! R2
router ospf 100
 area 13 virtual-link 3.3.3.3
```

```cisco
! R3
router ospf 100
 area 13 virtual-link 2.2.2.2
```

주의 사항:

- 상대편 ABR의 Router-ID 사용
- Transit Area는 Stub 불가
- 임시 복구 목적으로 사용
- 장기적으로 Area 0 물리 연결 재설계

---

## 8. 🔢 핵심 파라미터

| 항목 | 값 |
|---|---|
| 알고리즘 | Dijkstra SPF |
| Administrative Distance | 110 |
| IP Protocol | 89 |
| AllSPFRouters | 224.0.0.5 |
| AllDRouters | 224.0.0.6 |
| 기본 Reference BW | 100Mbps |
| Router-ID 우선순위 | 수동 → 최고 Loopback IP → 최고 물리 IP |
| Backbone Area | Area 0 |

---

## 9. 📑 LSA 유형

| Type | 이름 | 생성 주체 |
|---:|---|---|
| 1 | Router LSA | 모든 라우터 |
| 2 | Network LSA | DR |
| 3 | Summary LSA | ABR |
| 4 | ASBR Summary LSA | ABR |
| 5 | AS External LSA | ASBR |
| 7 | NSSA External LSA | NSSA ASBR |

---

## 10. 🗺️ Area 유형

| 유형 | 주요 동작 | 핵심 명령 |
|---|---|---|
| Normal | 일반 Area | 별도 명령 없음 |
| Stub | Type 5 차단 | `area X stub` |
| Totally Stub | 대부분의 Type 3 및 Type 4·5 차단 | `area X stub no-summary` |
| NSSA | Type 5 차단, Type 7 허용 | `area X nssa` |
| Totally NSSA | 대부분의 Type 3 및 Type 4·5 차단, Type 7 허용 | `area X nssa no-summary` |

---

## 11. 🔄 Neighbor State

```text
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

| 정체 상태 | 우선 확인 |
|---|---|
| Down | Interface, Passive, ACL, OSPF 적용 |
| Init | 단방향 통신, Multicast/NBMA |
| 2-Way | DROTHER 관계 및 DR/BDR |
| ExStart | MTU, 중복 Router-ID |
| Exchange | MTU, DBD Sequence |
| Loading | LSR/LSU 손실 |
| Full | 인접성 정상, LSDB/RIB 확인 |

---

## 12. 🔍 검증 명령어

```cisco
show ip ospf
show ip ospf neighbor
show ip ospf neighbor detail
show ip ospf interface brief
show ip ospf interface <interface>
show ip ospf database
show ip ospf database router
show ip ospf database network
show ip ospf database summary
show ip ospf database asbr-summary
show ip ospf database external
show ip ospf database nssa-external
show ip ospf border-routers
show ip route ospf
show ip protocols
```

---

## 13. 🧪 Debug 및 초기화

```cisco
debug ip ospf adj
debug ip ospf hello
debug ip ospf packet
debug ip ospf events
undebug all
```

Router-ID 변경 반영:

```cisco
clear ip ospf process
```

> [!WARNING]
> `clear ip ospf process`는 Neighbor와 경로를 초기화하므로 유지보수 시간에만 사용한다.

---

## 14. 🚑 빠른 장애 대응

```text
Neighbor 없음
 ├─ Interface Up/Up 확인
 ├─ OSPF 적용 및 Area 확인
 ├─ Passive Interface 확인
 ├─ Hello/Dead 확인
 ├─ 인증 확인
 └─ ACL/NBMA Mapping 확인

2-Way
 ├─ DROTHER끼리면 정상
 ├─ DR/BDR 존재 확인
 └─ Priority 0 여부 확인

ExStart/Exchange
 ├─ MTU 확인
 ├─ Router-ID 중복 확인
 └─ DBD 손실 확인

Full인데 경로 없음
 ├─ LSDB 확인
 ├─ Stub/NSSA 확인
 ├─ Summary/Filtering 확인
 ├─ 더 좋은 AD 경로 확인
 └─ CEF 확인
```

---

## 15. 📌 암기 요약

- `network` = 인터페이스 선택 + Area 할당
- DROTHER끼리 2-Way = 정상
- ExStart/Exchange = MTU와 중복 RID
- Area 불일치 = Neighbor 형성 실패
- ABR 요약 = `area range`
- ASBR 요약 = `summary-address`
- Stub 내부 재분배 필요 = NSSA 검토
- Reference Bandwidth = 도메인 전체 동일
- `mtu-ignore` = 임시 우회일 뿐 근본 해결 아님

> - 관련: 3-1. 🌲 OSPF (Open Shortest Path First) · 3-2. 🌲 OSPF 통합 정리 · 3-3. 🚑 OSPF 트러블슈팅 치트시트
