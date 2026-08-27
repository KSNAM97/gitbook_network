# 🧩 라우팅 재분배 통합 정리 — Redistribution 설계 한눈에

> **Tags:** #Routing #Redistribution #OSPF #EIGRP #RIPv2 #RouteMap #PrefixList #Summary  
> **핵심 요약:** 재분배는 서로 다른 라우팅 프로토콜의 경로를 다른 프로토콜로 주입하는 기능이다. 주요 위험은 **라우팅 루프·차선 경로·의도하지 않은 Prefix 광고·Seed Metric 오류**다. 실무에서는 Prefix-list로 대상을 분류하고, Route-map으로 허용·차단·Metric·Tag 정책을 적용한다.

---

## 1. 🎯 핵심 기술 개념

### Q1. 재분배에서 가장 먼저 통제해야 할 것은?

👉 **핵심 답변:** 양방향 또는 다중 지점 재분배에서 발생할 수 있는 **라우팅 피드백과 차선 경로**다.

한 프로토콜에서 다른 프로토콜로 전달한 경로가 다시 원래 프로토콜로 돌아오면 다음 문제가 발생할 수 있다.

- 동일 Prefix의 반복 재분배
- 원래 경로와 재분배 경로 간 AD 경쟁
- 비대칭 경로 또는 차선 경로
- 장애 시 예상하지 못한 우회
- 라우팅 루프 또는 Blackhole

표준 방어 패턴:

```text
A → B 재분배 시 Tag 부착
        ↓
B → A 재분배 시 동일 Tag 차단
```

```cisco
route-map A->B permit 10
 set tag 100
!
route-map B->A deny 5
 match tag 100
!
route-map B->A permit 10
```

> [!IMPORTANT]
> `set tag`만 설정해서는 루프가 차단되지 않는다. 반대 방향 재분배 정책에서 반드시 `match tag`로 차단해야 한다.

---

### Q2. Route-map과 Prefix-list를 함께 사용하는 이유는?

- **Prefix-list:** 어떤 Prefix를 처리할지 분류
- **Route-map:** 분류한 경로를 허용·차단하거나 속성을 변경
- **Tag:** 경로의 출처를 표시해 재유입 차단
- **Metric:** 대상 프로토콜에 맞는 Seed Metric 설정

```text
Prefix-list = 경로 선택
Route-map   = 정책 적용
Redistribute = 대상 프로토콜에 주입
```

> [!NOTE]
> Prefix-list의 `permit`은 그 자체로 재분배를 허용한다는 뜻이 아니다. Route-map의 `match`에 사용할 Prefix를 선택한다는 의미다.

---

### Q3. Connected 경로도 자동으로 재분배되는가?

일반적으로 다음 설정은 RIB에서 **RIP 경로로 학습된 경로**를 OSPF로 재분배한다.

```cisco
router ospf 1
 redistribute rip subnets
```

RIP의 `network` 명령으로 활성화된 인터페이스의 Connected 경로가 모두 자동으로 `redistribute rip` 대상이 되는 것은 아니다.

Connected 경로를 재분배하려면 일반적으로 별도 설정이 필요하다.

```cisco
router ospf 1
 redistribute connected subnets route-map CONN->OSPF
```

단, 일부 경로는 프로토콜 동작이나 플랫폼에 따라 다르게 보일 수 있으므로 다음 명령으로 원본 경로의 Source Code를 확인한다.

```cisco
show ip route <prefix>
```

---

### Q4. OSPF E1과 E2의 차이는?

| 유형 | 경로 선택 방식 | 적합한 환경 |
|---|---|---|
| E1 | 외부 Metric + ASBR까지의 내부 OSPF Cost | 내부 Cost를 경로 선택에 적극 반영할 때 |
| E2 | 외부 Metric 우선, 동률 시 내부 Cost를 Tie-break로 사용 | 외부 Metric 중심으로 선택할 때 |

OSPF 외부 경로의 기본 유형은 E2이며, 명시적인 Metric이 없을 때 일반적인 기본 외부 Metric은 20이다.

```cisco
set metric-type type-1
set metric 100
```

> [!CAUTION]
> “Dual-ASBR이면 무조건 E1, Single-ASBR이면 무조건 E2”는 절대 규칙이 아니다. 내부 OSPF Cost를 외부 경로 선택에 얼마나 반영할지에 따라 결정한다.

---

## 2. 🧱 재분배 설계 구조

### 2-1. 재분배 4단계

```text
① 원본 경로 확인
        ↓
② Prefix-list로 대상 분류
        ↓
③ Route-map으로 Permit/Deny/Metric/Tag 적용
        ↓
④ 대상 프로토콜에서 redistribute 실행
```

### 2-2. 주요 위험과 방어 수단

| 위험 | 방어 수단 |
|---|---|
| 경로 재유입 | `set tag` + 반대 방향 `match tag deny` |
| 불필요한 Prefix 광고 | Prefix-list + Route-map |
| 전 경로 차단 | Route-map Sequence와 미매칭 동작 확인 |
| 잘못된 Seed Metric | 대상 프로토콜 Metric 지정 |
| 차선 경로 | E1/E2, Metric, AD 및 재분배 지점 검토 |
| RFC1918 외부 유출 | Export 정책에서 RFC1918 명시적 차단 |

---

## 3. 🔢 주요 AD 및 Metric

### 3-1. Administrative Distance

| Protocol | 기본 AD |
|---|---:|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| EIGRP Internal | 90 |
| OSPF | 110 |
| RIP | 120 |
| EIGRP External | 170 |
| iBGP | 200 |

> AD는 동일한 Prefix와 Prefix Length를 제공하는 서로 다른 Route Source 중 어떤 경로를 RIB에 설치할지 결정한다.

---

### 3-2. 방향별 Seed Metric

| 대상 프로토콜 | 주요 요구사항 |
|---|---|
| OSPF | `subnets` 권장/필수 취급, 기본 외부 Metric 20 |
| RIP | 1~15 범위의 Metric 지정 또는 `default-metric` |
| EIGRP | 5개 Metric 지정 또는 `default-metric` |
| BGP | BGP 정책과 Origin/Metric/Tag 등을 별도로 검토 |

EIGRP Metric 순서:

```text
Bandwidth Delay Reliability Load MTU
```

예:

```cisco
metric 100000 10 255 1 1500
```

> [!NOTE]
> Connected 또는 Static 경로를 EIGRP로 재분배할 때는 플랫폼과 경로 특성에 따라 Metric 처리 예외가 있을 수 있다. 운영 설정에서는 명시적인 Metric 또는 `default-metric` 사용이 안전하다.

---

## 4. 🛠️ 표준 설정 템플릿

### 4-1. Connected → OSPF

```cisco
! R2: 13.13.12.0/24만 OSPF로 재분배

ip prefix-list ALLOW_CONN permit 13.13.12.0/24
!
route-map CONN->OSPF permit 10
 match ip address prefix-list ALLOW_CONN
!
router ospf 1
 redistribute connected subnets route-map CONN->OSPF
```

Route-map에 매치되지 않은 Connected 경로는 재분배되지 않는다.

---

### 4-2. RIPv2 → OSPF

```cisco
! RFC1918 차단용 분류
ip prefix-list PRIVATE permit 10.0.0.0/8 le 32
ip prefix-list PRIVATE permit 172.16.0.0/12 le 32
ip prefix-list PRIVATE permit 192.168.0.0/16 le 32
!
! E1 적용 대상
ip prefix-list METRIC_E1 permit 150.1.13.0/24
ip prefix-list METRIC_E1 permit 13.13.4.0/24
ip prefix-list METRIC_E1 permit 13.13.14.0/24
!
! Tag 적용 대상
ip prefix-list TAG199 permit 199.171.0.0/21 le 32
!
route-map RIP->OSPF deny 10
 match ip address prefix-list PRIVATE
!
route-map RIP->OSPF permit 20
 match ip address prefix-list METRIC_E1
 set metric-type type-1
 set metric 100
!
route-map RIP->OSPF permit 30
 match ip address prefix-list TAG199
 set tag 199
!
route-map RIP->OSPF permit 40
!
router ospf 1
 redistribute rip subnets route-map RIP->OSPF
```

`permit 40`은 미매칭 RIP 경로도 재분배하려는 경우에만 필요하다.

미매칭 경로를 차단하려면 `permit 40`을 제거한다.

---

### 4-3. OSPF → RIPv2

```cisco
router rip
 version 2
 no auto-summary
 redistribute ospf 1 metric 4
```

또는 기본 Seed Metric을 지정한다.

```cisco
router rip
 default-metric 4
 redistribute ospf 1
```

RIP에서 Metric 16은 도달 불가이므로 재분배 Metric은 1~15 범위로 설계한다.

---

### 4-4. RIP Offset-list 보정

```cisco
access-list 1 permit 13.13.1.0 0.0.0.255
access-list 1 permit 13.13.9.0 0.0.0.255
access-list 1 permit 13.13.10.0 0.0.0.255
!
router rip
 offset-list 1 out 3 FastEthernet0/0
```

- ACL 1과 일치하는 RIP 광고 Metric에 3을 추가
- 방향과 인터페이스를 반드시 확인
- 증가 결과가 16 이상이면 도달 불가로 처리될 수 있음

> [!CAUTION]
> Offset-list는 재분배 자체가 아니라 RIP/EIGRP Metric을 추가하는 도구다. Metric 보정 목적과 대상 경로를 명확히 구분한다.

---

### 4-5. EIGRP → OSPF

```cisco
ip prefix-list EIGRP_E2 permit 4.1.1.0/24
ip prefix-list EIGRP_E2 permit 198.198.0.0/16 le 32
!
route-map EIGRP->OSPF permit 10
 match ip address prefix-list EIGRP_E2
 set metric-type type-2
 set metric 200
!
route-map EIGRP->OSPF permit 20
 set metric-type type-1
 set tag 300
!
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP->OSPF
```

Sequence 20에는 `match`가 없으므로 이전 Sequence에 매치되지 않은 모든 경로가 허용된다.

---

### 4-6. OSPF → EIGRP

```cisco
ip prefix-list TAG100 permit 199.171.0.0/21 le 32
ip prefix-list DENY199 permit 199.171.0.0/16 le 32
!
route-map OSPF->EIGRP permit 10
 match ip address prefix-list TAG100
 set tag 100
!
route-map OSPF->EIGRP deny 20
 match ip address prefix-list DENY199
!
route-map OSPF->EIGRP permit 30
!
router eigrp 100
 redistribute ospf 1 metric 100000 10 255 1 1500 route-map OSPF->EIGRP
```

또는 EIGRP Process에 기본 Metric을 설정할 수 있다.

```cisco
router eigrp 100
 default-metric 100000 10 255 1 1500
 redistribute ospf 1 route-map OSPF->EIGRP
```

---

### 4-7. Tag 기반 재유입 차단

```cisco
! A → B: Tag 부착
route-map A->B permit 10
 set tag 100
!
! B → A: Tag 100 경로 차단
route-map B->A deny 5
 match tag 100
!
route-map B->A permit 10
```

실제 설계에서는 방향마다 고유한 Tag를 사용하는 것이 좋다.

```text
OSPF → EIGRP : Tag 100
EIGRP → OSPF : Tag 300
RIP → OSPF   : Tag 199
```

---

## 5. Prefix-list 범위

| 표기 | 의미 |
|---|---|
| `10.0.0.0/8` | 정확히 10.0.0.0/8 |
| `10.0.0.0/8 le 32` | /8 내부의 /8~ /32 |
| `10.0.0.0/8 ge 24` | /8 내부의 /24~ /32 |
| `10.0.0.0/8 ge 16 le 24` | /8 내부의 /16~ /24 |

```cisco
ip prefix-list EXAMPLE permit 199.171.0.0/21 le 32
```

위 항목은 199.171.0.0/21 자체와 그 안에 포함된 더 긴 Prefix를 선택한다.

---

## 6. 🔍 검증 명령어

```cisco
show ip route
show ip route ospf
show ip route eigrp
show ip route rip
show ip route <prefix>
show route-map
show route-map RIP->OSPF
show ip prefix-list
show ip protocols
```

OSPF External LSA와 Tag 확인:

```cisco
show ip ospf database external
show ip ospf database external <prefix>
show ip ospf database nssa-external
```

EIGRP External 경로 확인:

```cisco
show ip route eigrp
show ip eigrp topology
show ip eigrp topology <prefix>/<length>
```

---

## 7. 📌 핵심 정리

- 재분배 전 원본 RIB에 경로가 존재하는지 먼저 확인
- Prefix-list `permit`은 Route-map에서 처리할 경로를 선택
- Route-map 미매칭 경로는 재분배 정책에서 거부됨
- Catch-all Permit은 미매칭 경로도 허용하려는 경우에만 추가
- OSPF 재분배에는 `subnets`를 명시
- RIP Metric은 1~15 범위로 설정
- EIGRP는 Metric 5개 또는 `default-metric` 설정
- Tag는 반대 방향의 `match tag deny`와 함께 사용
- AD 변경은 Tag 및 정책 기반 제어가 불가능할 때 최후 수단으로 검토

> - 관련: 3-1. 🔄 Route Redistribution (재분배) with Route-Map Filtering · 3-3. 🚑 라우팅 재분배 트러블슈팅 치트시트 · 3-4. ⚡ 라우팅 재분배 퀵 레퍼런스