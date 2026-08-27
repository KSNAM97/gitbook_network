# ⚡ 라우팅 재분배 퀵 레퍼런스

> **Tags:** #Routing #Redistribution #QuickReference #RouteMap #PrefixList #Tag #CheatSheet
> **핵심 요약:** 재분배 설정·Route-map·Prefix-list·Tag 루프 방지·검증 명령어를 빠르게 조회하기 위한 암기 카드. 핵심 패턴은 `Prefix-list → Route-map → redistribute` 조합이며, 양방향 재분배에서는 반드시 Tag 기반 루프 차단이 필요하다.

---

## 1. 🛠️ 재분배 기본 문법

### OSPF로 재분배

```cisco
router ospf 1
 redistribute eigrp 100 metric-type 1 subnets route-map EIGRP->OSPF
 redistribute rip subnets route-map RIP->OSPF
 redistribute connected subnets route-map CONN->OSPF
 redistribute static subnets
```

> `subnets` 키워드 누락 시 Classful 네트워크만 재분배됨 — OSPF 재분배의 대표 함정.

### EIGRP로 재분배

```cisco
router eigrp 100
 redistribute ospf 1 metric 100000 10 255 1 1500 route-map OSPF->EIGRP
 redistribute rip metric 100000 10 255 1 1500
```

> Seed Metric 5개 (BW / Delay / Reliability / Load / MTU) 필수. 미설정 시 재분배 안 됨.

```cisco
router eigrp 100
 default-metric 100000 10 255 1 1500
 redistribute ospf 1 route-map OSPF->EIGRP
```

### RIP으로 재분배

```cisco
router rip
 version 2
 no auto-summary
 redistribute ospf 1 metric 4
 redistribute eigrp 100 metric 4
```

> Metric 1~15 필수. 16 이상이면 Unreachable 처리.

---

## 2. 📋 Prefix-list

```cisco
ip prefix-list ALLOW permit 13.13.0.0/16 le 24
ip prefix-list PRIVATE permit 10.0.0.0/8 le 32
ip prefix-list PRIVATE permit 172.16.0.0/12 le 32
ip prefix-list PRIVATE permit 192.168.0.0/16 le 32
```

| 옵션 | 동작 |
|---|---|
| `/24` (단독) | 정확히 /24만 매치 |
| `/16 le 24` | /16 이상 /24 이하 |
| `/8 le 32` | /8 이상 /32 이하 (전체 서브넷) |

---

## 3. 🗺️ Route-map 기본 패턴

```cisco
route-map POLICY deny 10
 match ip address prefix-list PRIVATE        ! 사설망 최우선 차단

route-map POLICY permit 20
 match ip address prefix-list METRIC_E1
 set metric-type type-1
 set metric 100

route-map POLICY permit 30
 match ip address prefix-list TAG_SET
 set tag 199

route-map POLICY permit 40                   ! Catch-all: 나머지 전부 허용
```

> Route-map 마지막에 Catch-all permit이 없으면 매칭 안 된 경로는 모두 차단 (Implicit deny).

---

## 4. 🔄 Tag 기반 루프 방지 (양방향 재분배 필수)

```cisco
! A → B 방향: Tag 부착
route-map A->B permit 10
 set tag 100

! B → A 방향: Tag 100 차단 후 나머지 허용
route-map B->A deny 5
 match tag 100
route-map B->A permit 10
```

> 모든 재분배 경계 라우터에 양방향 Tag 정책을 적용해야 루프가 완전히 차단됨.

---

## 5. ⚖️ E1 vs E2

| 구분 | Metric 계산 | 사용 시점 |
|---|---|---|
| **E2** (기본) | 외부 Metric 고정 | Single ASBR, Stub 환경 |
| **E1** | 외부 Metric + 내부 OSPF Cost 누적 | Dual ASBR, 경로 선택 세밀화 |

```cisco
set metric-type type-1      ! E1
set metric-type type-2      ! E2 (기본값)
```

---

## 6. 📌 Offset-list (RIP Metric 보정)

```cisco
access-list 1 permit 13.13.1.0 0.0.0.255
!
router rip
 offset-list 1 out 3 FastEthernet0/0    ! 매칭 경로에 +3 Metric 추가
```

---

## 7. 🔢 핵심 파라미터

| 항목 | 내용 |
|---|---|
| OSPF `subnets` | OSPF 재분배 필수. 누락 시 Classful만 광고 |
| EIGRP Seed Metric | BW·Delay·Rel·Load·MTU 5개 필수 |
| RIP Seed Metric | 1~15 (16 = Unreachable) |
| E2 기본 Metric | 20 |
| Tag 범위 | 1 ~ 4294967295 |
| Route-map Implicit deny | 마지막 permit 이후 모든 경로 차단 |

---

## 8. 🔍 검증 명령어

```cisco
show ip route
show ip route ospf
show ip route eigrp
show ip route rip
show ip protocols
show route-map
show route-map <NAME>
show ip prefix-list
show ip prefix-list <NAME>
show access-lists
```

```cisco
show ip ospf database external
show ip ospf database external <prefix>
show ip eigrp topology
show ip eigrp topology <prefix>/<length>
```

---

## 9. 🚨 증상별 Quick Check

| 증상 | 먼저 확인 |
|---|---|
| OSPF 재분배 후 특정 Subnet 누락 | `subnets` 키워드 |
| Route-map 적용했는데 전 경로 누락 | Catch-all permit 누락 (Implicit deny) |
| Prefix-list hit count 0 | `ge`/`le` 범위 오류 |
| EIGRP 재분배 경로 없음 | Seed Metric 5개 항목 |
| RIP 경로 16 hop | Seed/Offset Metric 과다 |
| 양방향 재분배 후 경로 재유입 | Tag 정책 방향 누락 |
| E2 경로 선택이 이상함 | Forwarding Address 도달 여부, ASBR 경로 |
| OSPF DB에는 있는데 RIB 없음 | Connected/Static AD 경쟁, Forwarding Address |

---

## 10. ⚠️ 핵심 주의사항

```properties
OSPF + subnets
= 필수. 없으면 /24 이하 서브넷 누락
```

```properties
Route-map permit N (마지막 Catch-all)
= 없으면 Implicit deny로 전 경로 차단
```

```properties
Tag = 루프 방지의 핵심
= 양쪽 재분배 경계 라우터 모두 적용
```

```properties
AD 조정은 최후 수단
= 먼저 Prefix-list > Route-map > Tag > Metric 순으로 해결
```

---

> 📌 **핵심 요약**
> - OSPF: `subnets` 필수, `metric-type 1/2`, `set tag` 루프 차단
> - EIGRP: Seed Metric 5개 (BW Delay Rel Load MTU) 필수
> - RIP: Seed Metric 1~15, 16 = Unreachable
> - Route-map 마지막 Catch-all permit 누락 → Implicit deny
> - 양방향 재분배 = Tag 부착(out) + Tag 차단(in) 반드시 쌍으로
> - 관련: 3-1. 🔄 Route Redistribution (재분배) with Route-Map Filtering · 3-2. 🧩 라우팅 재분배 통합 정리 · 3-3. 🚑 라우팅 재분배 트러블슈팅 치트시트
