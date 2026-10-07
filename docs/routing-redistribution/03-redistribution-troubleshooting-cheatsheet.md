# 라우팅 재분배 트러블슈팅 치트시트

> **Tags:** #Routing #Redistribution #Troubleshooting #Loop #RouteMap #Metric #CheatSheet  
> **핵심 요약:** 재분배 장애는 **원본 RIB → Prefix-list → Route-map Sequence → Redistribute 명령 → 대상 Protocol DB → 대상 RIB** 순서로 확인한다. 주요 원인은 원본 경로 부재, Route-map 미매칭, Seed Metric 오류, Tag 차단 오류, 잘못된 방향 및 경로 재유입이다.

---

## 1. 빠른 진단 순서

```text
① 원본 RIB에 경로가 있는가?
        ↓
② Prefix-list가 Prefix/Length를 정확히 선택하는가?
        ↓
③ Route-map Sequence에 매치되는가?
        ↓
④ Redistribute 명령과 Seed Metric이 올바른가?
        ↓
⑤ 대상 프로토콜 DB에 경로가 생성됐는가?
        ↓
⑥ 대상 RIB에 경로가 설치됐는가?
        ↓
⑦ 더 좋은 AD 또는 동일 Prefix의 경로가 있는가?
```

```cisco
show ip route <prefix>
show ip prefix-list
show route-map
show ip protocols
show ip ospf database external
show ip eigrp topology
show ip route <prefix>
```

---

## 2. 증상별 즉시 대응표

### 2-1. 광고 누락

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 원본부터 경로가 없음 | Source RIB 부재 | `show ip route <prefix>` |
| 서브넷이 누락됨 | OSPF `subnets` 누락 | `redistribute ... subnets` |
| 특정 Prefix만 누락 | Prefix/Length 불일치 | `show ip prefix-list detail` |
| 전 경로가 누락됨 | Route-map 미매칭 | Sequence와 Match Count 확인 |
| 일부 경로만 통과 | Catch-all 미설정 | 의도한 정책인지 확인 |
| RIP 재분배 실패 | Seed Metric 오류 | Metric 1~15 설정 |
| EIGRP 재분배 실패 | Seed Metric 미설정 | 5개 Metric 또는 `default-metric` |
| 경로가 DB에만 있음 | 더 좋은 RIB 경로 존재 | AD 및 Prefix Length 확인 |

> [!NOTE]
> Catch-all Permit은 항상 필수가 아니다. 정책에 매치되지 않은 경로까지 허용하려는 경우에만 추가한다.

---

### 2-2. 루프 및 차선 경로

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 경로가 원래 프로토콜로 재유입 | Tag 차단 없음 | `set tag` + `match tag deny` |
| Tag는 있지만 루프 지속 | 반대 방향 차단 누락 | 재분배 방향별 Route-map 확인 |
| 예상과 다른 출구 선택 | E1/E2 또는 Metric 설계 | 내부 Cost 반영 여부 검토 |
| RIB 경로가 계속 변경됨 | 다중 재분배 지점 | 모든 경계 지점 정책 비교 |
| 동일 Prefix가 여러 Source로 보임 | AD 경쟁 | 먼저 Tag/Prefix 정책 수정 |
| 장애 시 Blackhole 발생 | Summary 또는 Null0 | Summary와 상세 경로 확인 |

---

### 2-3. 오광고 및 Metric 문제

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| RFC1918이 외부로 광고됨 | Export 정책 누락 | PRIVATE Prefix를 Route-map에서 Deny |
| EIGRP External 경로 없음 | Metric 5개 누락 | 명시적 Metric 또는 `default-metric` |
| RIP 경로가 도달 불가 | Metric이 16 이상 | Seed/Offset Metric 감소 |
| RIP Metric이 예상보다 큼 | Offset-list 중복 적용 | 방향·Interface·ACL 확인 |
| OSPF E2 경로가 기대와 다름 | 외부 Metric 동일 | 내부 Tie-break Cost와 Forwarding Address 확인 |
| Tag가 보이지 않음 | Set Sequence 미매칭 | Route-map Match Count 확인 |

---

## 3. 대표 장애 시나리오

### 시나리오 1. 특정 경로가 재분배되지 않음

```cisco
! 1. 원본 RIB 확인
show ip route <prefix>
show ip route rip
show ip route ospf
show ip route eigrp

! 2. Prefix-list 확인
show ip prefix-list
show ip prefix-list <NAME>

! 3. Route-map Match Count 확인
show route-map <NAME>

! 4. 재분배 설정 확인
show running-config | section router ospf
show running-config | section router eigrp
show running-config | section router rip
```

확인 사항:

```text
□ 원본 경로의 Route Source가 예상과 같은가?
□ Prefix-list의 Base Prefix가 올바른가?
□ ge/le 범위가 실제 Prefix Length와 일치하는가?
□ Route-map의 이전 Deny Sequence에 먼저 매치되지 않는가?
□ 미매칭 경로를 허용해야 하는데 Catch-all Permit이 빠지지 않았는가?
□ OSPF 방향에 subnets가 있는가?
□ 대상 프로토콜의 Seed Metric이 유효한가?
```

---

### 시나리오 2. 양방향 재분배 후 경로 재유입

```cisco
! A → B에서 Tag 부착
route-map A->B permit 10
 set tag 100
!
! B → A에서 Tag 100 차단
route-map B->A deny 5
 match tag 100
!
route-map B->A permit 10
```

검증:

```cisco
show route-map A->B
show route-map B->A
show ip ospf database external <prefix>
show ip route <prefix>
```

> [!IMPORTANT]
> 하나의 재분배 지점에만 Tag 정책을 적용하면 다른 재분배 지점으로 경로가 돌아올 수 있다. 모든 경계 라우터에서 방향별 정책을 확인한다.

---

### 시나리오 3. RFC1918 경로가 외부로 광고됨

```cisco
ip prefix-list PRIVATE permit 10.0.0.0/8 le 32
ip prefix-list PRIVATE permit 172.16.0.0/12 le 32
ip prefix-list PRIVATE permit 192.168.0.0/16 le 32
!
route-map X->Y deny 10
 match ip address prefix-list PRIVATE
!
route-map X->Y permit 20
```

> Prefix-list의 `permit`은 PRIVATE Prefix를 선택하고, 실제 차단은 Route-map의 `deny`가 수행한다.

---

### 시나리오 4. OSPF EIGRP 재분배가 안 됨

```cisco
router eigrp 100
 redistribute ospf 1 metric 100000 10 255 1 1500 route-map OSPF->EIGRP
```

또는:

```cisco
router eigrp 100
 default-metric 100000 10 255 1 1500
 redistribute ospf 1 route-map OSPF->EIGRP
```

검증:

```cisco
show ip protocols
show route-map OSPF->EIGRP
show ip eigrp topology
show ip route eigrp
```

---

### 시나리오 5. RIP 경로가 광고되지 않음

```cisco
router rip
 version 2
 no auto-summary
 redistribute ospf 1 metric 4
```

검증:

```cisco
show ip protocols
show ip route ospf
show ip route rip
debug ip rip
```

RIP Metric이 16 이상이면 도달 불가로 처리된다.

운영 장비에서 Debug 사용 후 반드시 종료한다.

```cisco
undebug all
```

---

### 시나리오 6. OSPF External LSA는 있지만 RIB에 없음

```cisco
show ip ospf database external <prefix>
show ip route <prefix>
show ip route ospf
```

확인 사항:

- Connected 또는 Static 경로가 있는가?
- 더 낮은 AD의 EIGRP Internal 경로가 있는가?
- 동일 Prefix Length인가?
- Forwarding Address에 도달 가능한가?
- ASBR 또는 Forwarding Address까지의 내부 경로가 있는가?
- Route Filtering 또는 Area Type 문제가 있는가?

---

### 시나리오 7. E1/E2 경로 선택이 예상과 다름

```cisco
show ip route <prefix>
show ip ospf database external <prefix>
show ip ospf border-routers
show ip ospf interface brief
```

판단 기준:

```text
E1 = 외부 Metric + 내부 OSPF Cost
E2 = 외부 Metric 우선
     외부 Metric이 같으면 내부 Cost를 Tie-break로 사용
```

E2가 Router-ID 또는 최초 학습 순서만으로 선택된다고 단정하면 안 된다.

---

### 시나리오 8. Offset-list 적용 후 RIP 경로가 사라짐

```cisco
show access-lists 1
show ip protocols
show ip route rip
show running-config | section router rip
```

원인:

```text
기존 Metric + Offset ≥ 16
```

조치:

```cisco
router rip
 no offset-list 1 out 3 FastEthernet0/0
```

또는 Offset 값을 낮춘다.

---

## 4. AD 조정 시 주의사항

AD 조정은 루프 방지의 기본 수단이 아니다.

우선순위:

```text
1. Prefix-list 정책 수정
2. Route-map Sequence 수정
3. Tag 기반 재유입 차단
4. Metric 및 E1/E2 조정
5. 마지막 수단으로 AD 조정
```

예시:

```cisco
router eigrp 100
 distance 171 0.0.0.0 255.255.255.255 <ACL>
```

- Source Address/Wildcard는 Route를 보낸 라우터를 선택
- 마지막 ACL은 대상 Route Prefix를 선택
- 프로토콜별 `distance` 문법과 적용 범위가 다름
- 변경 전후 RIB 변화를 반드시 검증

---

## 5. 핵심 검증 명령어

```cisco
show ip route
show ip route <prefix>
show ip route ospf
show ip route eigrp
show ip route rip
show ip protocols
show route-map
show ip prefix-list
show access-lists
```

```cisco
show ip ospf database external
show ip ospf database external <prefix>
show ip ospf database nssa-external
show ip eigrp topology
show ip eigrp topology <prefix>/<length>
```

---

## 6. 강제 초기화 주의

다음 명령은 일반적인 첫 조치로 사용하지 않는다.

```cisco
clear ip route *
clear ip ospf process
```

영향:

- 다수 경로 제거 및 재학습
- OSPF Neighbor 재형성
- 일시적인 트래픽 손실
- CPU 사용량 및 Update 증가

먼저 정책 Match Count와 Protocol DB를 확인한 뒤 유지보수 시간에 제한적으로 사용한다.

---

## 7. 최종 체크리스트

```text
□ 원본 RIB에 경로가 존재하는가?
□ 실제 Route Source가 예상과 같은가?
□ Prefix-list가 정확한 Prefix Length를 선택하는가?
□ Route-map의 이전 Sequence에 먼저 매치되지 않는가?
□ Route-map Match Count가 증가하는가?
□ 미매칭 경로를 허용할지 차단할지 결정했는가?
□ OSPF 재분배에 subnets가 있는가?
□ RIP Metric이 1~15인가?
□ EIGRP Seed Metric이 설정됐는가?
□ Tag가 부착되고 반대 방향에서 차단되는가?
□ 대상 Protocol DB에 경로가 생성됐는가?
□ RIB에 더 선호되는 경로가 존재하지 않는가?
```

> - 관련: 3-1. 🔄 Route Redistribution (재분배) with Route-Map Filtering · 3-2. 🧩 라우팅 재분배 통합 정리 · 3-4. ⚡ 라우팅 재분배 퀵 레퍼런스