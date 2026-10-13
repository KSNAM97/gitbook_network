# BGP 통합 정리 — AS·Peer·Split-horizon·Next-hop 한눈에

> **Tag:** #Routing #BGP #EGP #PathVector #iBGP #eBGP #SplitHorizon #RouteReflector #Confederation #NextHop #Summary
> **핵심 요약:** BGP는 **AS 사이에서 Routing 정보를 교환하는 Path Vector 방식의 EGP**다. TCP 179번으로 Peer(iBGP/eBGP)를 맺고, `AS_PATH`로 AS 단위 Loop를 방지하며, Attribute 기반 Policy로 Best Path를 고른다. 실무에서 가장 자주 마주치는 세 가지 문제는 **① Peer 연결(Loopback/Multihop) ② iBGP Split-horizon ③ DMZ 구간 Next-hop** 이며, 각각 `update-source`·`ebgp-multihop` / Full-mesh·Route-reflector·Confederation / Static·Redistribute·`next-hop-self`로 해결한다.

---

## 1. 핵심 기술 개념

### Q1. BGP는 IGP와 무엇이 다른가?

👉 **핵심 답변:** IGP는 **하나의 AS 내부**에서 Metric(Hop/Cost/Composite)으로 빠르게 최적 경로를 구하지만, BGP는 **AS와 AS 사이**에서 `AS_PATH`, `LOCAL_PREF`, `MED`, `NEXT_HOP` 등 **Attribute와 관리자가 설정한 Policy**로 Best Path를 결정한다.

| 구분 | IGP (RIP/EIGRP/OSPF/IS-IS) | EGP (BGP) |
|------|----------------------------|-----------|
| 범위 | 하나의 AS 내부 | 서로 다른 AS 사이 |
| 목적 | 빠른 경로 계산·수렴 | Policy 기반 경로 제어, 확장성 |
| 경로 선택 | Metric (Hop Count / Cost / Composite) | Attribute (AS_PATH, LOCAL_PREF, MED, NEXT_HOP …) |
| Algorithm | Distance Vector / Link-State | Path Vector |
| Transport | 자체 / IP Protocol | TCP 179 |

---

### Q2. BGP의 핵심 특징은?

- **Path Vector** : 목적지까지 거쳐온 AS를 `AS_PATH`에 기록, 자신의 AS가 포함된 Update는 Loop로 판단하여 폐기
- **Classless** : VLSM, CIDR 지원
- **Incremental Update** : 변경분만 UPDATE Message로 전달 (주기적 전체 Table 전송 없음)
- **TCP 179** : TCP 3-Way Handshake 완료 후 BGP Peer 연결 진행
- **AD** : eBGP 20 / iBGP 200 (Cisco 기준)
- **Best Path 1개** : 기본적으로 목적지당 하나의 Best Path 선택 (필요 시 Multipath)
- **4-Byte AS** 지원 (2-Byte : 1 ~ 65535, Private : 64512 ~ 65534 / 4200000000 ~ 4294967294)

---

### Q3. BGP Message와 Peer State는?

| Message | 역할 |
|---------|------|
| Open | TCP 3-Way 후 BGP 연결 개시 |
| Update | 라우팅 업데이트 정보 |
| Keepalive | Peer 유지 점검 (60초 주기) |
| Notification | 오류 발생 시 사용 (예: `peer in wrong AS`) |

```text
Idle → Connected → OpenSent → OpenConfirm → Established
                        ↓
                     Active (Open 수신 실패 — 설정 오류 가능성 높음)
```

| State | 의미 | 정체 시 확인 |
|-------|------|--------------|
| Idle | `neighbor`로 지정한 Peer를 찾는 시작 단계 | neighbor IP, Peer로의 Routing, Multihop |
| Connected | TCP 3-Way 완료 | — |
| OpenSent | Open 전송, 상대 Open 대기 | 상대 `remote-as`/neighbor 설정 |
| Active | Open 수신 실패 | BGP 설정 오류 |
| OpenConfirm | Open 수신 | — |
| Established | Peer 성립, Keepalive만 교환 | 정상 |

---

### Q4. Peer를 물리/Loopback Interface로 연결할 때 차이는?

👉 **핵심 답변:** Loopback으로 연결하려면 `update-source loopback 0`이 필요하고, **iBGP는 Multihop 기본값이 255라 그대로 연결되지만 eBGP는 기본값 1이라 `ebgp-multihop`을 반드시 조정**해야 한다. (BGP에서 Connected는 1 hop)

```cisco
router bgp 100
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
 neighbor 13.13.2.2 ebgp-multihop 3
```

---

### Q5. 광고(`network`)는 어떻게 하며 AD는 얼마인가?

- IGP와 동일하게 `network` 사용, Subnetting된 네트워크는 **`mask` 필수**

```cisco
router bgp 100
 network 10.1.1.0 mask 255.255.255.0       ! A class를 /24로 Subnetting
 network 172.16.1.0 mask 255.255.255.0     ! B class를 /24로 Subnetting
 network 192.168.1.0                       ! 원래 C class
```

- 수신 경로: eBGP `[20/0]`, iBGP `[200/0]` (Metric 정보 없음)

---

### Q6. iBGP Split-horizon이란, 어떻게 해결하는가?

👉 **핵심 답변:** **iBGP로 수신한 Route는 다른 iBGP Peer에게 다시 광고하지 않는다.** eBGP는 `AS_PATH`로 Loop를 막지만 iBGP Peer끼리는 `AS_PATH`가 같아 Loop 방지가 불가능하므로 iBGP는 "한 번만" 전달되도록 제한된다.

| 해결 방식 | 핵심 설정 | 특징 |
|-----------|-----------|------|
| Full-mesh | 동일 AS 내 모든 Router 간 iBGP Peer | 단순, Peer 수 급증 (N×(N-1)/2) |
| Route-reflector | `neighbor ... route-reflector-client` | RR이 Client에게 반사, Peer 수 감소 |
| Confederation | `bgp confederation identifier <AS>` + `bgp confederation peers <Sub-AS>` | Sub-AS 간 논리적 eBGP, `AS_PATH`에 `(Sub-AS)` 표기 |

---

### Q7. BGP Next-hop 문제는 언제 생기고 어떻게 해결하는가?

👉 **핵심 답변:** 동일 AS 내에서는 Next-hop이 변경되지 않으므로, **eBGP 구간이 DMZ(IGP 미구성)** 이면 AS 내부 Router가 외부 AS Next-hop IP에 도달하지 못해 BGP 경로가 Routing Table에 등록되지 않는다.

| 방법 | 설정 | 비고 |
|------|------|------|
| Static Route | 내부 Router마다 `ip route <DMZ> <mask> <내부 Next-hop>` | 수동, 확장성 낮음 |
| IGP 재분배 | Border Router에서 `redistribute connected ... route-map` | DMZ 노출 범위를 Route-map으로 제어 |
| next-hop-self | `neighbor <iBGP-Peer> next-hop-self` | BGP 자체 기능, 권장 |

---

### Q8. BGP 동기화(Synchronization)란?

- BGP로 수신한 네트워크는 **IGP로도 확인되어야** 한다는 규칙. 동기화가 되지 않으면 Routing Table에는 보이지만 통신이 안 되는 **Blackhole** 현상이 생긴다.
- BGP Peer 연결 자체는 **NLRI**(IP 도달성)만 확립되면 중간 Router에 BGP가 없어도 가능하다.
- 실습에서는 모든 BGP Router에 `no synchronization`을 설정했다.

---

### Q9. BGP를 써야 할 때와 쓰지 말아야 할 때는?

| 필요 | 부적합 |
|------|--------|
| 다중 AS 연결, Multi-homing, Policy 적용, 대량 Routing 정보 교환, 유입/유출 경로 제어, Internet Routing, MPLS L3 VPN(MP-BGP), 대규모 DC | 단일 ISP 연결 소규모 망, Default Route만으로 충분, Policy 불필요, Router 자원 부족 → **Static Default Route** 권장 |

> **Customer AS는 Transit AS가 되어서는 안 된다.** (Transit은 ISP/Backbone 역할)

---

## 2. 실습 흐름 요약

```text
① Pre-config(EIGRP로 Loopback/Serial 도달성 확보)
② eBGP Peer (R1 AS100 ↔ R2 AS200) : 물리 → Loopback + ebgp-multihop
③ iBGP Peer (R2~R5 AS200) : 물리 → Loopback + update-source
④ network 광고 (R1: 10.1.1.0/24, 172.16.1.0/24, 192.168.1.0/24)
⑤ R4·R5 미수신 확인 → Split-horizon
⑥ Full-mesh / Route-reflector / Confederation 으로 해결 (각 실습 후 BGP 삭제)
⑦ eBGP 구간 DMZ화 → Next-hop 문제 확인
⑧ Static / Redistribute / next-hop-self 로 해결
```

---

## 3. 핵심 요약

- BGP = **AS 간 Path Vector EGP**, TCP **179**, AD **eBGP 20 / iBGP 200**
- `AS_PATH`로 AS Loop 방지, **iBGP는 한 번만 전달** (Split-horizon)
- Loopback Peer: `update-source` 필수, **eBGP는 `ebgp-multihop`** 필수 (iBGP는 기본 255)
- Split-horizon 해결 = **Full-mesh / Route-reflector / Confederation**
- Next-hop 문제 해결 = **Static / IGP 재분배 / next-hop-self**
- 검증 = `show ip bgp summary` → `show ip bgp` → `show ip route bgp`
