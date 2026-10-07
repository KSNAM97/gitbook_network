# 라우팅 필터링 & 정책 제어 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #RouteFiltering #PrefixList #DistributeList #RouteMap #OffsetList
> **핵심 요약:** 라우팅 정책 장애는 **매칭 오류 → 암묵적 deny → 적용 방향 오류 → 프로토콜별 필터링 단계 오해 → 정책 재평가 누락** 순서로 확인한다. 먼저 원본 경로가 존재하는지 확인하고, Prefix-list/ACL 매칭, Route-map Sequence, Distribute-list 방향, RIB 설치 결과를 단계별로 비교한다.

---

## 1. 공통 진단 순서

```properties
[1] 원본 경로가 이웃 또는 Topology/Routing DB에 존재하는가?
[2] ACL/Prefix-list가 의도한 Prefix와 Length를 매칭하는가?
[3] Route-map의 Permit/Deny 및 Sequence가 올바른가?
[4] Distribute-list/Offset-list의 in/out 방향이 맞는가?
[5] 프로토콜이 정책을 어느 단계에서 적용하는가?
[6] 정책 변경 후 업데이트가 재평가됐는가?
[7] 최종 RIB와 실제 포워딩 경로가 일치하는가?
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 필터 적용 후 모든 경로 삭제 | 암묵적 deny | Prefix-list/Route-map | Catch-all permit 추가 |
| `/24`만 의도했는데 `/16~24` 매칭 | `le 24` 오해 | Prefix-list 상세 | `ge 24 le 24` 사용 |
| 필터가 전혀 동작하지 않음 | in/out 또는 인터페이스 오류 | `show ip protocols` | 적용 방향 수정 |
| OSPF LSDB에 경로가 남음 | 정상 동작 | LSDB/RIB 비교 | ABR/ASBR 필터 검토 |
| Route-map 적용 후 모두 차단 | 마지막 permit 누락 | `show route-map` | 빈 permit Sequence 추가 |
| Tag가 다음 프로토콜에서 사라짐 | 재분배 시 미설정 | RIB/Topology | `set tag` 명시 |
| Offset 후 경로가 그대로 | ACL 불일치/값 부족 | Topology Metric | Metric 차이보다 큰 값 설정 |
| RIP 경로가 사라짐 | Hop Count 16 이상 | RIP DB | Offset 감소 |
| BGP 정책 수정이 미반영 | Soft Refresh 미실행 | Neighbor Route | `soft in/out` 수행 |
| Bogon 필터 후 내부 CGN 장애 | 정책 범위 오류 | VRF/인터커넥트 | 내부 정책에서 예외 처리 |

---

## 3. 핵심 진단 명령어

### Prefix-list

```cisco
show ip prefix-list
show ip prefix-list <NAME>
show ip prefix-list detail <NAME>
show ip prefix-list <NAME> <PREFIX>/<LENGTH>
```

### ACL 및 Route-map

```cisco
show access-lists
show route-map
show route-map <NAME>
```

### Routing Protocol

```cisco
show ip protocols
show ip route
show ip route <PREFIX>
show ip eigrp topology all-links
show ip rip database
show ip ospf database
```

### BGP

```cisco
show ip bgp <PREFIX>
show ip bgp neighbors <PEER> routes
show ip bgp neighbors <PEER> advertised-routes
```

---

## 4. 대표 장애 시나리오

### 시나리오 1. Prefix-list 적용 후 모든 경로가 사라짐

#### 잘못된 정책

```cisco
ip prefix-list FILTER deny 10.0.0.0/8 le 32
```

#### 원인

Prefix-list 마지막에는 암묵적 deny가 존재합니다.

```properties
10.0.0.0/8 하위 = 명시적 deny
그 외 모든 Prefix = 암묵적 deny
```

#### 조치

```cisco
ip prefix-list FILTER seq 10 deny 10.0.0.0/8 le 32
ip prefix-list FILTER seq 100 permit 0.0.0.0/0 le 32
```

---

### 시나리오 2. 정확한 `/24`만 매칭되지 않음

#### 잘못된 설정

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 le 24
```

이 설정은 `/16~24`를 모두 매칭합니다.

#### 수정

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
```

---

### 시나리오 3. 연속된 8개 `/24`보다 더 많은 경로가 매칭됨

#### 기존 설정

```cisco
ip prefix-list NET172 permit 172.16.0.0/21 le 32
```

#### 원인

`/21 le 32`는 다음을 모두 허용합니다.

```properties
172.16.0.0/21
그 안의 /22, /23, /24 ... /32
```

정확히 8개의 `/24`만 허용하려면:

```cisco
ip prefix-list NET172 permit 172.16.0.0/21 ge 24 le 24
```

---

### 시나리오 4. Distribute-list가 동작하지 않음

#### 확인

```cisco
show ip protocols
show running-config | section router eigrp
show ip prefix-list FILTER
```

#### 원인 후보

- 잘못된 인터페이스 지정
- `in/out` 방향 반대
- 다른 Routing Process에 적용
- Prefix-list 이름 오타
- Prefix Length 불일치
- Connected/Static 등 해당 프로토콜 외의 경로 확인

#### 예시 수정

```cisco
router eigrp 100
 no distribute-list prefix FILTER out FastEthernet0/0
 distribute-list prefix FILTER in FastEthernet0/0
```

---

### 시나리오 5. OSPF `distribute-list in` 이후에도 LSA가 보임

#### 증상

```cisco
show ip route
```

에서는 경로가 사라졌지만:

```cisco
show ip ospf database
```

에는 LSA가 남아 있습니다.

#### 원인

정상 동작입니다. OSPF Inbound Distribute-list는 경로의 **로컬 RIB 설치를 억제**하지만 LSA Flooding과 LSDB 동기화는 차단하지 않습니다.

#### 조치 선택

- 로컬 라우터 RIB만 제외: `distribute-list in`
- Area 간 Summary 차단: ABR `area filter-list prefix`
- 외부 경로 유입 제어: ASBR 재분배 Route-map
- Stub/NSSA/Summarization 등 OSPF Area 설계 검토

```cisco
router ospf 1
 area 1 filter-list prefix AREA_FILTER in
```

> 방향은 ABR 기준으로 Area 1에 Summary가 들어가는 방향입니다.

---

### 시나리오 6. Route-map 적용 후 전체 경로가 차단됨

#### 잘못된 설정

```cisco
route-map FILTER deny 10
 match tag 100
```

#### 원인

Tag 100은 명시적으로 deny되고, 나머지는 Route-map의 암묵적 deny에 걸립니다.

#### 수정

```cisco
route-map FILTER deny 10
 match tag 100
!
route-map FILTER permit 20
```

---

### 시나리오 7. Route-map OR/AND 조건이 의도와 다름

#### 안전한 OR

```cisco
route-map FILTER deny 10
 match tag 100 200
```

또는 Sequence 분리:

```cisco
route-map FILTER deny 10
 match tag 100
!
route-map FILTER deny 20
 match tag 200
!
route-map FILTER permit 30
```

#### AND

```cisco
route-map FILTER deny 10
 match tag 100
 match ip address prefix-list PFX_LIST
```

```properties
Tag 100 AND PFX_LIST
```

---

### 시나리오 8. Tag를 설정했지만 다음 프로토콜에서 보이지 않음

#### 확인

```cisco
show ip route <PREFIX>
show ip eigrp topology <PREFIX>/<LENGTH>
show ip ospf database external <PREFIX>
show route-map
```

#### 원인 후보

- Route-map이 실제 정책에 연결되지 않음
- Prefix-list/ACL이 Prefix를 매칭하지 않음
- 재분배 Route-map에 `set tag` 누락
- 중간 프로토콜 또는 플랫폼에서 Tag 전달 방식 차이
- 더 앞선 Route-map Sequence에 매칭

#### 재분배 시 명시적 Tag 설정

```cisco
route-map EIGRP_TO_OSPF permit 10
 set tag 100
!
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP_TO_OSPF
```

---

### 시나리오 9. Offset-list 적용 후 경로가 바뀌지 않음

#### 확인

```cisco
show ip protocols
show access-lists 1
show ip eigrp topology all-links
show ip route <PREFIX>
```

#### 원인 후보

1. ACL이 목적지 Prefix를 매칭하지 않음
2. `in/out` 방향 오류
3. 잘못된 인터페이스 지정
4. Offset 값이 경로 간 Metric 차이보다 작음
5. 다른 경로가 Feasibility Condition을 만족하지 않음
6. 경로가 EIGRP가 아닌 다른 프로토콜에서 설치됨

#### 수정

```cisco
router eigrp 100
 no offset-list 1 in 100000 Serial1/1
 offset-list 1 in 700000 Serial1/1
```

> Offset 값은 임의의 암기값보다 실제 두 경로의 Metric 차이를 확인한 후 설정합니다.

---

### 시나리오 10. Offset-list 수정 중 EIGRP 설정 전체 삭제

#### 위험 명령

```cisco
no router eigrp 100
```

이 명령은 Offset 한 줄이 아니라 EIGRP Process 전체를 삭제합니다.

#### 올바른 수정

```cisco
router eigrp 100
 no offset-list 1 in 100000 Serial1/1
 offset-list 1 in 700000 Serial1/1
```

---

### 시나리오 11. RIP Offset 후 경로가 사라짐

#### 확인

```cisco
show ip rip database
show ip route rip
```

#### 원인

RIP은 Metric 16을 Unreachable로 처리합니다.

```properties
원본 Metric 14 + Offset 2 = 16 → Unreachable
```

#### 수정

```cisco
router rip
 no offset-list 20 in 2 Serial0/0
 offset-list 20 in 1 Serial0/0
```

---

### 시나리오 12. BGP Bogon 정책 수정 후 변화가 없음

#### 확인

```cisco
show ip prefix-list BOGON
show ip bgp neighbors <ISP-IP> routes
show ip bgp neighbors <ISP-IP> advertised-routes
```

#### Inbound 재평가

```cisco
clear ip bgp <ISP-IP> soft in
```

#### Outbound 재광고

```cisco
clear ip bgp <ISP-IP> soft out
```

> Hard Clear는 BGP 세션과 전체 경로를 제거하므로 장애 영향이 큽니다.

---

### 시나리오 13. Bogon Prefix-list를 적용했는데 Spoofing 패킷이 들어옴

#### 원인

BGP Prefix-list는 BGP 경로를 필터링할 뿐 데이터 패킷의 Source IP를 검사하지 않습니다.

#### 추가 방어

```cisco
interface GigabitEthernet0/0
 ip verify unicast source reachable-via rx
```

> Strict uRPF는 비대칭 라우팅 환경에서 정상 트래픽을 차단할 수 있습니다. 토폴로지에 따라 Loose Mode, ACL 또는 방화벽 정책을 선택합니다.

---

## 5. 안전한 정책 변경 절차

```properties
[1] 기존 Prefix-list/Route-map/Protocol 설정 백업
[2] 변경 전 원본 경로와 Metric 기록
[3] 신규 Match 객체를 먼저 생성
[4] show 명령으로 Prefix 매칭 검증
[5] Route-map Sequence 추가
[6] 정책에 연결
[7] Soft Refresh 또는 자연 업데이트 대기
[8] RIB와 실제 Traceroute 검증
[9] 이상 시 즉시 Rollback
```

```cisco
show running-config | section prefix-list
show running-config | section route-map
show running-config | section router
show ip route <PREFIX>
show ip eigrp topology all-links
```

---

> 📌 **핵심 요약**
> - 먼저 원본 경로 존재 여부 확인
> - 다음은 Prefix-list/ACL 매칭 확인
> - 전체 삭제는 대부분 암묵적 deny 또는 Route-map Catch-all Permit 누락
> - OSPF RIB 필터와 LSA 필터를 구분
> - Offset은 실제 Metric 차이를 확인한 후 적용
> - BGP 변경은 Hard Clear보다 Soft Refresh
> - Bogon Route Filter와 Source Spoofing 방어는 별도 계층
> - 관련: 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter · 3-2-2. ⛳  Distribute-list를 활용한 라우팅 필터링 · 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging · 3-2-4. ⚖️ Offset-list를 활용한 경로 조작 · 3-2-5. 🧩 라우팅 필터링 & 정책 제어 통합 정리
