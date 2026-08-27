# ⚡ 라우팅 필터링 & 정책 제어 퀵 레퍼런스

> **Tag:** #Network #QuickReference #PrefixList #DistributeList #RouteMap #OffsetList #Bogon
> **핵심 요약:** Prefix-list·Distribute-list·Route-map·Offset-list의 문법과 검증 명령어를 빠르게 조회하기 위한 암기 카드. Prefix-list는 매칭, Distribute-list는 필터링, Route-map은 복합 정책, Offset-list는 Metric 가산에 사용한다.

---

## 1. 🧠 도구 선택표

| 요구사항 | 도구 |
|---|---|
| 주소 범위 단순 매칭 | ACL |
| Prefix Length 매칭 | Prefix-list |
| 경로 허용·차단 | Distribute-list |
| Match + 속성 변경 | Route-map |
| 기존 Metric 증가 | Offset-list |
| 경로 출처 표시 | Route Tag |
| BGP 경로 정책 | Prefix-list + Route-map |
| OSPF 경로 선호도 | `ip ospf cost` |
| Source Spoofing 방어 | ACL/uRPF/Firewall |

---

## 2. 📍 Prefix-list 문법

```cisco
ip prefix-list <NAME> [seq <SEQ>] {permit|deny} \
 <PREFIX>/<LENGTH> [ge <MIN>] [le <MAX>]
```

정확한 Prefix:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16
```

모든 하위 Prefix:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 le 32
```

`/24~27`:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 27
```

정확한 `/24`:

```cisco
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
```

정확히 `/0~32` 모든 IPv4 Prefix:

```cisco
ip prefix-list ANY permit 0.0.0.0/0 le 32
```

Default Route만:

```cisco
ip prefix-list DEFAULT_ONLY permit 0.0.0.0/0
```

---

## 3. ✏️ Prefix-list 편집

추가:

```cisco
ip prefix-list NET172 seq 7 permit 172.16.2.0/27
```

라인 삭제:

```cisco
no ip prefix-list NET172 seq 7
```

전체 삭제:

```cisco
no ip prefix-list NET172
```

Sequence 재정렬:

```cisco
ip prefix-list NET172 resequence 10 10
```

> Resequence 명령 지원 여부와 문법은 IOS/IOS XE 버전을 확인합니다.

---

## 4. 🛡️ Bogon 기본 목록

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

BGP Inbound:

```cisco
router bgp 65000
 neighbor <ISP-IP> prefix-list BOGON in
```

BGP Outbound 권장:

```cisco
ip prefix-list OUR_PUBLIC_PREFIXES permit <OUR_PREFIX>/<LENGTH>
!
router bgp 65000
 neighbor <ISP-IP> prefix-list OUR_PUBLIC_PREFIXES out
```

Soft Refresh:

```cisco
clear ip bgp <ISP-IP> soft in
clear ip bgp <ISP-IP> soft out
```

---

## 5. ⛳ Distribute-list 문법

ACL 기반:

```cisco
router <PROTOCOL>
 distribute-list <ACL> {in|out} [INTERFACE]
```

Prefix-list 기반:

```cisco
router <PROTOCOL>
 distribute-list prefix <PREFIX_LIST> {in|out} [INTERFACE]
```

Route-map 기반:

```cisco
router <PROTOCOL>
 distribute-list route-map <ROUTE_MAP> {in|out} [INTERFACE]
```

---

## 6. ⛳ Distribute-list 예시

특정 Prefix만 수신:

```cisco
ip prefix-list PFX_IN permit 13.13.5.0/24
ip prefix-list PFX_IN permit 13.13.15.0/24
!
router eigrp 100
 distribute-list prefix PFX_IN in FastEthernet0/0
```

특정 대역 제외:

```cisco
ip prefix-list FILTER seq 10 deny 172.16.0.0/16 le 32
ip prefix-list FILTER seq 100 permit 0.0.0.0/0 le 32
!
router eigrp 100
 distribute-list prefix FILTER in FastEthernet0/0
```

RIP:

```cisco
router rip
 distribute-list prefix FILTER in
 distribute-list prefix FILTER out Serial0/0
```

OSPF 로컬 RIB 필터:

```cisco
router ospf 1
 distribute-list prefix FILTER in
```

> OSPF LSA Flooding은 차단하지 않습니다.

OSPF ABR Summary 필터:

```cisco
router ospf 1
 area 1 filter-list prefix FILTER in
```

---

## 7. 🗺️ Route-map 문법

```cisco
route-map <NAME> {permit|deny} <SEQ>
 match <CONDITION>
 set <ACTION>
```

마지막 Catch-all Permit:

```cisco
route-map FILTER permit 100
```

Route-map 전체 삭제:

```cisco
no route-map FILTER
```

특정 Sequence 삭제:

```cisco
no route-map FILTER permit 20
```

---

## 8. 🔎 Route-map Match

| 명령 | 매칭 대상 |
|---|---|
| `match ip address <ACL>` | ACL |
| `match ip address prefix-list <NAME>` | Prefix-list |
| `match tag <TAG>` | Route Tag |
| `match interface <IF>` | Route Outgoing Interface |
| `match ip next-hop prefix-list <NAME>` | Next-hop |
| `match ip route-source prefix-list <NAME>` | Route Source |
| `match metric <VALUE>` | Metric |
| `match route-type <TYPE>` | Route Type |

OR:

```cisco
match tag 100 200 300
```

AND:

```cisco
match tag 100
match ip address prefix-list PFX_LIST
```

---

## 9. 🔧 Route-map Set

| 명령 | 동작 |
|---|---|
| `set tag <VALUE>` | Route Tag 설정 |
| `set metric <VALUE>` | Metric 설정 |
| `set metric-type type-1` | OSPF E1 |
| `set metric-type type-2` | OSPF E2 |
| `set local-preference <VALUE>` | BGP Local Preference |
| `set metric <VALUE>` | BGP MED |
| `set as-path prepend <AS>` | AS_PATH 추가 |
| `set community <VALUE>` | BGP Community |
| `set ip next-hop <IP>` | PBR Next-hop |

---

## 10. 🏷️ Route Tagging

Tag 부여:

```cisco
ip prefix-list NET199 permit 199.171.0.0/21 le 32
!
route-map SET_TAG permit 10
 match ip address prefix-list NET199
 set tag 100
!
route-map SET_TAG permit 20
```

Tag 차단:

```cisco
route-map DENY_TAG deny 10
 match tag 100
!
route-map DENY_TAG permit 20
```

재분배 Loop 방지:

```cisco
route-map OSPF_TO_EIGRP deny 10
 match tag 100
!
route-map OSPF_TO_EIGRP permit 20
 set tag 200
```

---

## 11. ⚖️ Offset-list

문법:

```cisco
router <PROTOCOL>
 offset-list <ACL> {in|out} <OFFSET> [INTERFACE]
```

EIGRP:

```cisco
access-list 10 permit 10.0.0.0 0.255.255.255
!
router eigrp 100
 offset-list 10 in 5000000 Serial0/1
```

RIP:

```cisco
access-list 20 permit 192.168.100.0 0.0.0.255
!
router rip
 offset-list 20 in 2 Serial0/0
```

삭제:

```cisco
router eigrp 100
 no offset-list 10 in 5000000 Serial0/1
```

```properties
RIP Metric 16 이상 = Unreachable
EIGRP Offset = Composite Metric 증가
OSPF = Offset-list 미지원
BGP = Offset-list 미지원
```

---

## 12. 🌲 OSPF 대체 Metric 조정

```cisco
interface GigabitEthernet0/0
 ip ospf cost 100
```

Reference Bandwidth:

```cisco
router ospf 1
 auto-cost reference-bandwidth 100000
```

> Reference Bandwidth는 OSPF Domain 전체에서 일관되게 설정합니다.

---

## 13. 🔍 검증 명령어

Prefix-list:

```cisco
show ip prefix-list
show ip prefix-list <NAME>
show ip prefix-list detail <NAME>
show ip prefix-list <NAME> <PREFIX>/<LENGTH>
```

Route-map:

```cisco
show route-map
show route-map <NAME>
```

Routing Protocol:

```cisco
show ip protocols
show ip route
show ip route <PREFIX>
show ip eigrp topology all-links
show ip rip database
show ip ospf database
```

BGP:

```cisco
show ip bgp <PREFIX>
show ip bgp neighbors <PEER> routes
show ip bgp neighbors <PEER> advertised-routes
```

실제 경로:

```cisco
traceroute <DESTINATION> source <SOURCE>
```

---

## 14. ⚠️ 암기용 함정표

| 설정 | 실제 의미 |
|---|---|
| `172.16.0.0/16` | 정확히 `/16` |
| `172.16.0.0/16 le 24` | `/16~24` |
| `172.16.0.0/16 ge 24 le 24` | 정확한 `/24` |
| `0.0.0.0/0` | Default Route만 |
| `0.0.0.0/0 le 32` | 모든 IPv4 Prefix |
| 빈 Route-map Permit | 나머지 모두 허용 |
| OSPF Distribute-list In | 로컬 RIB 설치 필터 |
| BGP Prefix-list | 경로 광고 필터 |
| uRPF/ACL | 데이터 패킷 Source 검증 |
| Offset-list | Metric 증가만 수행 |

---

## 15. 🔄 정책 변경 원칙

```properties
Prefix-list 변경
    ↓
Route-map Match 확인
    ↓
Distribute-list/BGP Neighbor 연결 확인
    ↓
Soft Refresh 또는 자연 업데이트
    ↓
RIB/Topology 확인
    ↓
Traceroute 검증
```

BGP:

```cisco
clear ip bgp <PEER> soft in
clear ip bgp <PEER> soft out
```

> `clear ip bgp *` 또는 Routing Neighbor 전체 Clear는 최후 수단으로 사용합니다.

---

> 📌 **핵심 요약**
> - Prefix-list: `ip prefix-list NAME permit PREFIX/LENGTH ge X le Y`
> - Distribute-list: `distribute-list prefix NAME in|out IF`
> - Route-map: `match` + `set`
> - Offset-list: 기존 Metric에 값 추가
> - 모든 경로 허용: `permit 0.0.0.0/0 le 32`
> - Default만 허용: `permit 0.0.0.0/0`
> - 관련: 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter · 3-2-2. ⛳  Distribute-list를 활용한 라우팅 필터링 · 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging · 3-2-4. ⚖️ Offset-list를 활용한 경로 조작 · 3-2-5. 🧩 라우팅 필터링 & 정책 제어 통합 정리 · 3-2-6. 🚑 라우팅 필터링 & 정책 제어 트러블슈팅 치트시트
