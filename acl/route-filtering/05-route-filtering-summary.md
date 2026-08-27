# 🧩 라우팅 필터링 & 정책 제어 통합 정리

> **Tag:** #Network #Routing #RouteFiltering #PrefixList #DistributeList #RouteMap #OffsetList #Bogon
> **핵심 요약:** 라우팅 정책은 **무엇을 고를 것인가(Prefix-list/ACL) → 허용·차단할 것인가(Distribute-list) → 속성을 변경할 것인가(Route-map) → 기존 Metric에 가중치를 더할 것인가(Offset-list)**로 나눠 이해한다. Prefix-list는 Prefix Length까지 매칭하고, Distribute-list는 경로의 송수신을 필터링하며, Route-map은 Match+Set 정책을 수행한다. Offset-list는 RIP/EIGRP Metric을 증가시켜 경로 우선순위를 낮춘다.

---

## 1. 🎯 핵심 기술 개념

### Q1. 라우팅 정책 도구를 관통하는 하나의 구조는?

👉 **핵심 답변:** 라우팅 정책은 일반적으로 **매칭 도구 → 정책 엔진 → 적용 지점**의 3단계로 구성됩니다.

```properties
[매칭 도구]
ACL / Prefix-list / Tag / Metric / Route Type

        ↓

[정책 엔진]
Distribute-list / Route-map / Offset-list

        ↓

[적용 지점]
Routing Protocol in/out / Redistribution / BGP Neighbor
```

| 도구 | 핵심 역할 |
|---|---|
| ACL | 주소 범위 단순 매칭 |
| Prefix-list | Prefix와 Prefix Length 매칭 |
| Distribute-list | 경로 허용·차단 |
| Route-map | Match + Set 기반 복합 정책 |
| Offset-list | 기존 Metric에 값 추가 |
| Route Tag | 경로 출처 표시 및 재분배 루프 방지 |

---

### Q2. ACL과 Prefix-list는 무엇이 다른가?

👉 **핵심 답변:** ACL은 주소 범위를 매칭하지만, Prefix-list는 **Network Prefix와 Prefix Length를 동시에 검사**할 수 있습니다.

```cisco
! 172.16.0.0/16 범위 안의 정확한 /24만 매칭
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
```

| 항목 | ACL | Prefix-list |
|---|---|---|
| 주소 범위 매칭 | 가능 | 가능 |
| Prefix Length 조건 | 불가 | 가능 |
| `ge`, `le` | 불가 | 가능 |
| 트래픽 필터링 | 가능 | 불가 |
| 라우팅 정책 매칭 | 가능 | 권장 |
| 라인 편집 | 형식/IOS에 따라 차이 | Sequence 기본 지원 |

> Prefix-list 자체는 방화벽이 아닙니다. BGP, EIGRP, RIP, OSPF 및 Route-map 등의 라우팅 정책에서 경로를 매칭하는 도구입니다.

---

### Q3. Prefix-list의 `ge`와 `le`는 어떻게 해석하는가?

```cisco
ip prefix-list NET172 permit 172.16.0.0/16
```

정확히 `172.16.0.0/16`만 매칭합니다.

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 le 32
```

`172.16.0.0/16`에 포함되면서 Prefix Length가 `/16~32`인 모든 경로를 매칭합니다.

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 27
```

다음 두 조건을 모두 만족해야 합니다.

```properties
1. 경로가 172.16.0.0/16 범위 안에 존재
2. Prefix Length가 /24 이상 /27 이하
```

정확히 `/24`만 매칭:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
```

---

### Q4. Distribute-list의 `in/out`은 어떻게 판단하는가?

👉 **핵심 답변:** 항상 **현재 라우터 기준**으로 판단합니다.

```properties
in  = 이웃으로부터 받은 경로를 로컬에서 수용하기 전 필터링
out = 이웃에게 경로를 광고하기 전 필터링
```

- 정책 소유자가 광고자라면 `out`
- 정책 소유자가 수신자라면 `in`
- `out` 필터는 하위 라우터까지 경로가 전달되지 않는 파급 효과가 있음
- 프로토콜마다 필터가 작동하는 단계가 다름

---

### Q5. Route-map은 왜 라우팅 세계의 if-then-else인가?

👉 **핵심 답변:** Route-map은 `match`로 조건을 검사하고 `set`으로 속성을 변경하는 정책 엔진입니다.

```cisco
route-map POLICY permit 10
 match ip address prefix-list PFX_NET
 set tag 100
```

```properties
IF Prefix-list PFX_NET에 매칭되면
THEN Route Tag를 100으로 설정
```

Route-map도 마지막에 암묵적 deny가 존재합니다.

```cisco
route-map FILTER deny 10
 match tag 100
!
route-map FILTER permit 20
! match가 없는 permit = 나머지 모든 경로 허용
```

---

### Q6. Route-map의 OR/AND 조건은 어떻게 해석하는가?

👉 **핵심 답변:** 일반적으로 **하나의 match 명령에 나열된 복수 값은 OR**, 서로 다른 Match 유형은 AND로 평가합니다.

```cisco
route-map POLICY permit 10
 match tag 100 200
```

```properties
Tag 100 OR Tag 200
```

```cisco
route-map POLICY permit 20
 match tag 100
 match ip address prefix-list PFX_LIST
```

```properties
Tag 100 AND Prefix-list PFX_LIST
```

> 동일한 Match 유형을 여러 줄로 반복했을 때의 병합 방식은 명령 종류와 IOS 구현에 따라 다를 수 있습니다. OR 조건은 가능한 한 한 줄에 명시하거나 Route-map Sequence를 분리하는 것이 안전합니다.

---

### Q7. Offset-list는 필터링과 무엇이 다른가?

👉 **핵심 답변:** Offset-list는 경로를 제거하지 않고 **기존 Metric에 값을 더해 경로를 덜 선호하도록** 만듭니다.

```cisco
router eigrp 100
 offset-list 10 in 5000000 Serial0/1
```

| 도구 | 결과 |
|---|---|
| Distribute-list deny | 경로 차단 |
| Route-map deny | 경로 차단 |
| Route-map `set metric` | Metric 설정/변경 |
| Offset-list | 기존 Metric에 가산 |
| OSPF `ip ospf cost` | 인터페이스 Cost 변경 |

- RIP: Hop Count 증가
- EIGRP: Composite Metric 증가
- OSPF: Offset-list 미지원
- BGP: Offset-list 대신 Local Preference, MED, AS_PATH 등 사용

---

## 2. 🛠️ Prefix-list 표준 설정

### 2-1. 기본 문법

```cisco
ip prefix-list <NAME> [seq <SEQ>] {permit|deny} <PREFIX>/<LENGTH> [ge <MIN>] [le <MAX>]
```

정확한 Prefix:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16
```

모든 하위 Prefix:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 le 32
```

특정 Prefix Length:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
```

---

### 2-2. 실무 패턴

```cisco
! 10.0.0.0/8과 모든 하위 Prefix
ip prefix-list NET10 permit 10.0.0.0/8 le 32
!
! 100.0.0.0/8 범위의 /25~27
ip prefix-list NET100 permit 100.0.0.0/8 ge 25 le 27
!
! 182.160.0.0/16 범위의 정확한 /24
ip prefix-list NET182 permit 182.160.0.0/16 ge 24 le 24
!
! 172.16.0.0/21 및 모든 하위 Prefix
ip prefix-list NET172 permit 172.16.0.0/21 le 32
```

> 마지막 예시는 172.16.0.0/21 자체와 그 안의 `/22~32` Prefix까지 모두 허용합니다. 정확히 8개의 `/24`만 매칭하려면 다음과 같이 설정합니다.

```cisco
ip prefix-list NET172 permit 172.16.0.0/21 ge 24 le 24
```

---

### 2-3. Prefix-list 편집

```cisco
ip prefix-list NET172 permit 172.16.1.0/27
ip prefix-list NET172 permit 172.16.4.0/26
!
ip prefix-list NET172 seq 7 permit 172.16.2.0/27
!
no ip prefix-list NET172 seq 10
!
no ip prefix-list NET172
```

---

## 3. 🛡️ Bogon Route Filter

### 3-1. Bogon 필터의 정확한 역할

Bogon Prefix-list를 BGP Neighbor에 적용하면 다음을 제어합니다.

```properties
BGP Inbound  = ISP가 특수·비공개 Prefix를 경로로 광고하는 것 차단
BGP Outbound = 자기 AS가 잘못된 Prefix를 외부로 광고하는 것 방지
```

하지만 다음 기능은 수행하지 않습니다.

```properties
BGP Prefix-list ≠ 패킷 Source Address 필터
BGP Prefix-list ≠ Source Spoofing 직접 방어
```

Spoofed Source 패킷을 차단하려면 별도의 데이터 플레인 정책이 필요합니다.

- Infrastructure ACL
- Unicast RPF
- 인터페이스 ACL
- 방화벽 정책
- BCP 38/84 기반 Source Validation

---

### 3-2. 기본 Bogon Prefix-list

```cisco
ip prefix-list BOGON seq 10 deny 0.0.0.0/8 le 32
ip prefix-list BOGON seq 20 deny 10.0.0.0/8 le 32
ip prefix-list BOGON seq 30 deny 100.64.0.0/10 le 32
ip prefix-list BOGON seq 40 deny 127.0.0.0/8 le 32
ip prefix-list BOGON seq 50 deny 169.254.0.0/16 le 32
ip prefix-list BOGON seq 60 deny 172.16.0.0/12 le 32
ip prefix-list BOGON seq 70 deny 192.0.2.0/24 le 32
ip prefix-list BOGON seq 80 deny 192.168.0.0/16 le 32
ip prefix-list BOGON seq 90 deny 198.18.0.0/15 le 32
ip prefix-list BOGON seq 100 deny 198.51.100.0/24 le 32
ip prefix-list BOGON seq 110 deny 203.0.113.0/24 le 32
ip prefix-list BOGON seq 120 deny 224.0.0.0/4 le 32
ip prefix-list BOGON seq 130 deny 240.0.0.0/4 le 32
ip prefix-list BOGON seq 1000 permit 0.0.0.0/0 le 32
```

> 위 목록은 학습용 기본 목록이며 완전한 Fullbogon 목록이 아닙니다. IANA Special-Purpose Registry와 Team Cymru Fullbogon Feed를 기준으로 정기 갱신해야 합니다.

---

### 3-3. BGP Inbound 적용

```cisco
router bgp 65000
 neighbor <ISP-IP> prefix-list BOGON in
```

정책 변경 후 Inbound Soft Refresh:

```cisco
clear ip bgp <ISP-IP> soft in
```

> Route Refresh를 지원하는 일반적인 BGP 환경에서는 세션을 끊는 Hard Clear보다 Soft Refresh를 우선합니다.

---

### 3-4. BGP Outbound 권장 정책

Outbound는 “Bogon이 아니면 모두 광고”보다 **자기 AS가 소유하고 광고할 Prefix만 허용**하는 방식이 안전합니다.

```cisco
ip prefix-list OUR_PUBLIC_PREFIXES seq 10 permit 198.51.100.0/24
!
router bgp 65000
 neighbor <ISP-IP> prefix-list OUR_PUBLIC_PREFIXES out
```

> `198.51.100.0/24`는 문서용 주소이므로 실제 운영에서는 할당받은 공인 Prefix로 변경합니다.

Bogon Outbound 필터를 보조 방어선으로 함께 사용할 수 있지만, 하나의 Neighbor에 동일 방향 Prefix-list를 여러 개 직접 적용할 수 없으므로 Route-map 또는 하나로 통합된 Prefix-list를 사용합니다.

---

### 3-5. CGN 대역 주의

```properties
100.64.0.0/10 = Shared Address Space, RFC 6598
```

- 공인 인터넷 BGP 테이블에는 일반적으로 광고하지 않음
- ISP 내부 CGN 구간에서는 정상적으로 사용 가능
- 일반 인터넷 콘텐츠 서버는 NAT 변환 후의 공인 Source IP를 보게 됨
- 사설 인터커넥트 또는 ISP 내부 정책에서는 예외가 필요할 수 있음

---

## 4. 🔀 Distribute-list 표준 설정

### 4-1. ACL 기반

```cisco
access-list 1 permit 13.13.5.0 0.0.0.255
access-list 1 permit 13.13.15.0 0.0.0.255
access-list 1 permit 198.198.0.0 0.0.255.255
!
router eigrp 100
 distribute-list 1 in FastEthernet0/0
```

---

### 4-2. Prefix-list 기반

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 25 le 27
!
router eigrp 100
 distribute-list prefix NET172 in FastEthernet0/0
```

---

### 4-3. 특정 Prefix 제외 후 나머지 허용

```cisco
ip prefix-list FILTER seq 10 deny 172.16.0.0/16 le 32
ip prefix-list FILTER seq 100 permit 0.0.0.0/0 le 32
!
router eigrp 100
 distribute-list prefix FILTER in FastEthernet0/0
```

---

### 4-4. Route-map 기반

```cisco
ip prefix-list PFX_IN permit 13.13.5.0/24
ip prefix-list PFX_IN permit 13.13.15.0/24
ip prefix-list PFX_IN permit 198.198.0.0/16
!
route-map EIGRP_IN permit 10
 match ip address prefix-list PFX_IN
!
router eigrp 100
 distribute-list route-map EIGRP_IN in FastEthernet0/0
```

---

### 4-5. 프로토콜별 차이

| 프로토콜 | `in` | `out` | 주요 주의점 |
|---|---|---|---|
| RIP | 지원 | 지원 | Hop Count, 주기 업데이트 |
| EIGRP | 지원 | 지원 | Topology와 Successor 변화 확인 |
| OSPF | 로컬 RIB 설치 필터 | 제한적 | LSA Flooding은 유지 |
| BGP | Prefix-list/Route-map | Prefix-list/Route-map | Soft Refresh 사용 |

OSPF Inbound Route-map:

```cisco
route-map OSPF_FILTER deny 10
 match tag 777
!
route-map OSPF_FILTER permit 20
!
router ospf 1
 distribute-list route-map OSPF_FILTER in
```

> 이 정책은 해당 OSPF 경로가 로컬 RIB에 설치되는 것을 막지만 LSDB에서 LSA를 제거하거나 LSA Flooding을 차단하지 않습니다.

ABR Summary 필터:

```cisco
ip prefix-list AREA_FILTER deny 172.16.0.0/16 le 32
ip prefix-list AREA_FILTER permit 0.0.0.0/0 le 32
!
router ospf 1
 area 1 filter-list prefix AREA_FILTER in
```

> `in`은 ABR이 다른 Area의 Summary LSA를 Area 1로 들여보내는 방향입니다. 실행 위치와 Area 방향을 반드시 토폴로지 기준으로 확인합니다.

---

## 5. 🗺️ Route-map 표준 설정

### 5-1. 기본 구조

```cisco
route-map <NAME> {permit|deny} <SEQ>
 match <CONDITION>
 set <ACTION>
```

---

### 5-2. Tag 부여

```cisco
access-list 41 permit 199.171.8.0 0.0.7.255
!
route-map R4_TAG100 permit 10
 match ip address 41
 set tag 100
!
route-map R4_TAG100 permit 20
!
router eigrp 100
 distribute-list route-map R4_TAG100 out FastEthernet0/0
```

---

### 5-3. Tag 기반 필터링

```cisco
route-map TAG100_DENY deny 10
 match tag 100
!
route-map TAG100_DENY permit 20
!
router eigrp 100
 distribute-list route-map TAG100_DENY in FastEthernet0/0
```

---

### 5-4. 다단계 정책

```cisco
ip prefix-list PRIVATE permit 10.0.0.0/8 le 32
ip prefix-list PRIVATE permit 172.16.0.0/12 le 32
ip prefix-list PRIVATE permit 192.168.0.0/16 le 32
!
ip prefix-list NET199 permit 199.171.0.0/21 le 32
!
route-map EIGRP_FILTER deny 10
 match ip address prefix-list PRIVATE
!
route-map EIGRP_FILTER permit 20
 match ip address prefix-list NET199
 set tag 150
!
route-map EIGRP_FILTER permit 30
 set tag 100
!
router eigrp 100
 distribute-list route-map EIGRP_FILTER in FastEthernet0/0
```

---

### 5-5. 재분배 Loop 방지 Tagging

```cisco
route-map EIGRP_TO_OSPF permit 10
 set tag 100
!
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP_TO_OSPF
```

반대 방향 재유입 차단:

```cisco
route-map OSPF_TO_EIGRP deny 10
 match tag 100
!
route-map OSPF_TO_EIGRP permit 20
 set tag 200
!
router eigrp 100
 redistribute ospf 1 metric 100000 100 255 1 1500 route-map OSPF_TO_EIGRP
```

> Route Tag 보존 여부를 프로토콜과 플랫폼에 따라 검증해야 합니다. 단순한 빈 Route-map이 기존 Tag의 보존을 보장한다고 가정하지 말고 필요한 Tag를 명시적으로 설정합니다.

---

## 6. ⚖️ Offset-list 표준 설정

### 6-1. 기본 문법

```cisco
router <PROTOCOL>
 offset-list <ACL> {in|out} <OFFSET> [INTERFACE]
```

---

### 6-2. EIGRP 특정 경로 강등

```cisco
access-list 1 permit 13.13.15.0 0.0.0.255
!
router eigrp 100
 offset-list 1 in 700000 Serial1/1
```

광고 측에서 적용:

```cisco
access-list 1 permit 13.13.15.0 0.0.0.255
!
router eigrp 100
 offset-list 1 out 700000 Serial1/1
```

---

### 6-3. WAN Primary/Backup

```cisco
access-list 10 permit 10.0.0.0 0.255.255.255
!
router eigrp 100
 offset-list 10 in 10000000 Serial0/1
```

---

### 6-4. RIP Hop Count 조작

```cisco
access-list 20 permit 192.168.100.0 0.0.0.255
!
router rip
 offset-list 20 in 2 Serial0/0
```

> RIP Metric이 16 이상이 되면 Unreachable로 처리됩니다.

---

### 6-5. Offset 값 변경

```cisco
router eigrp 100
 no offset-list 1 in 100000 Serial1/1
 offset-list 1 in 700000 Serial1/1
```

> `no router eigrp 100`은 EIGRP 프로세스 전체를 삭제하는 명령이므로 Offset-list 한 줄을 수정할 목적으로 사용하면 안 됩니다.

---

## 7. 🔍 검증 명령어

### Prefix-list

```cisco
show ip prefix-list
show ip prefix-list BOGON
show ip prefix-list detail BOGON
show ip prefix-list BOGON 10.0.0.0/8
```

### Distribute-list

```cisco
show ip protocols
show running-config | section router eigrp
show ip route eigrp
show ip eigrp topology all-links
```

### Route-map/Tag

```cisco
show route-map
show route-map EIGRP_FILTER
show ip route 199.171.8.0
show ip eigrp topology | include tag
```

### Offset-list

```cisco
show ip protocols
show ip eigrp topology all-links
show ip route eigrp
show ip rip database
traceroute <DESTINATION> source <SOURCE>
```

### BGP

```cisco
show ip bgp neighbors <ISP-IP> received-routes
show ip bgp neighbors <ISP-IP> routes
show ip bgp neighbors <ISP-IP> advertised-routes
show ip bgp regexp
```

---

## 8. ⚠️ 실무 체크리스트

- [ ] 트래픽 ACL과 라우팅 Prefix-list를 구분했는가?
- [ ] Prefix-list의 `ge/le` 범위를 정확히 계산했는가?
- [ ] 마지막 암묵적 deny를 고려했는가?
- [ ] Distribute-list의 `in/out`을 현재 라우터 기준으로 판단했는가?
- [ ] Route-map 마지막에 필요한 Catch-all Permit이 있는가?
- [ ] Route-map Match 유형별 OR/AND 동작을 확인했는가?
- [ ] 재분배 경계에서 Tag를 명시적으로 설정하고 역유입을 차단했는가?
- [ ] Offset 적용 전후 Metric 차이를 수치로 비교했는가?
- [ ] OSPF 필터가 LSDB/LSA Flooding을 막는다고 오해하지 않았는가?
- [ ] BGP Outbound는 자기 Prefix만 명시적으로 허용했는가?
- [ ] Bogon/Fullbogon 목록을 정기적으로 갱신하는가?
- [ ] Source Spoofing 방어는 uRPF/ACL 등 데이터 플레인에서 별도로 구성했는가?
- [ ] 정책 변경 후 무조건 Neighbor를 Clear하지 않고 Soft Refresh를 우선했는가?

---

> 📌 **핵심 요약**
> - Prefix-list = Prefix + Prefix Length 매칭
> - Distribute-list = 경로 허용/차단
> - Route-map = Match + Set 정책 엔진
> - Offset-list = 기존 Metric 가산
> - Route Tag = 경로 출처 표시 및 재분배 Loop 방지
> - OSPF `distribute-list in` = 로컬 RIB 필터, LSA Flooding 차단 아님
> - Bogon BGP 필터 = 경로 광고 제어, Source Spoofing 직접 방어 아님
> - 관련: 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter · 3-2-2. ⛳  Distribute-list를 활용한 라우팅 필터링 · 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging · 3-2-4. ⚖️ Offset-list를 활용한 경로 조작 · 3-1. 🔄 Route Redistribution (재분배) with Route-Map Filtering
