# BGP 명령어 퀵 레퍼런스

> **Tag:** #Routing #BGP #QuickReference #iBGP #eBGP #RouteReflector #Confederation #CheatSheet
> **핵심 요약:** BGP Process, Peer(물리/Loopback/Multihop), `network` 광고, Split-horizon 해결(Full-mesh/Route-reflector/Confederation), Next-hop 해결, 검증 명령어를 빠르게 조회하기 위한 명령어 카드.

---

## 1. 기본 설정

### 1-1. BGP Process / Router-ID

```cisco
router bgp <AS>
 bgp router-id <X.X.X.X>
 no auto-summary
 no synchronization
```

### 1-2. 물리적 Interface Peer (eBGP / iBGP)

```cisco
router bgp 100
 neighbor 13.13.12.2 remote-as 200      ! AS가 다르면 eBGP
!
router bgp 200
 neighbor 13.13.23.3 remote-as 200      ! AS가 같으면 iBGP
```

### 1-3. Loopback Interface Peer

```cisco
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
```

### 1-4. eBGP Loopback Peer (Multihop)

```cisco
router bgp 100
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
 neighbor 13.13.2.2 ebgp-multihop 3      ! default 1, 최대 255
```

### 1-5. 네트워크 광고

```cisco
router bgp 100
 network 10.1.1.0 mask 255.255.255.0     ! Subnetting된 네트워크는 mask 필수
 network 172.16.1.0 mask 255.255.255.0
 network 192.168.1.0                     ! 원래 Class 경계 네트워크
```

### 1-6. BGP 설정 삭제

```cisco
no router bgp 100
no router bgp 200
```

---

## 2. Split-horizon 해결

### 2-1. Full-mesh

```cisco
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source loopback 0
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source loopback 0
 neighbor 13.13.5.5 remote-as 200
 neighbor 13.13.5.5 update-source loopback 0
```

### 2-2. Route-reflector

```cisco
router bgp 200
 neighbor 13.13.3.3 remote-as 200
 neighbor 13.13.3.3 update-source Loopback0
 neighbor 13.13.3.3 route-reflector-client
 neighbor 13.13.4.4 remote-as 200
 neighbor 13.13.4.4 update-source Loopback0
 neighbor 13.13.4.4 route-reflector-client
```

### 2-3. Confederation

```cisco
router bgp 64512                          ! Sub-AS
 bgp confederation identifier 200         ! 실제 AS 번호
 bgp confederation peers 64513            ! 다른 Sub-AS (Sub-AS 경계 Router)
 neighbor 13.13.4.4 remote-as 64513
 neighbor 13.13.4.4 update-source loopback 0
 neighbor 13.13.4.4 ebgp-multihop 3
```

---

## 3. Next-hop 해결

### 3-1. Static Route

```cisco
ip route 13.13.12.0 255.255.255.0 13.13.23.2
```

### 3-2. DMZ 구간 IGP 재분배 (Route-map으로 범위 제한)

```cisco
route-map EIGRP_REDI permit 10
 match interface serial 1/0.12
!
router eigrp 100
 redistribute connected metric 1544 2000 255 1 1500 route-map EIGRP_REDI
```

```cisco
route-map EIGRP_REDI deny 10
 match interface loopback100 loopback200
route-map EIGRP_REDI permit 20
!
router eigrp 100
 redistribute connected metric 1544 2000 255 1 1500 route-map EIGRP_REDI
```

### 3-3. next-hop-self

```cisco
router bgp 64512
 neighbor 13.13.3.3 next-hop-self
```

---

## 4. 핵심 파라미터

| 항목 | 값 |
|------|----|
| TCP Port | 179 |
| eBGP AD | 20 |
| iBGP AD | 200 |
| Keepalive | 60초 |
| eBGP Multihop 기본값 | 1 |
| iBGP Multihop 기본값 | 255 |
| Multihop 최대 | 255 |
| 2-Byte AS | 1 ~ 65535 (65535 예약) |
| Private AS | 64512 ~ 65534, 4200000000 ~ 4294967294 |
| 4-Byte AS | 지원 |

---

## 5. Peer State

| State | 설명 |
|-------|------|
| Idle | `neighbor` Peer를 찾는 시작 단계 |
| Connected | TCP 3-Way 완료 |
| OpenSent | Open 전송, 수신 대기 (미수신 시 Active) |
| OpenConfirm | Open 수신 |
| Established | Peer 성립, Keepalive만 교환 |

---

## 6. 검증 명령어

```cisco
show ip bgp summary                 ! Peer 상태 / 수신 Prefix 수 (State/PfxRcd)
show ip bgp                         ! BGP Table (*> Best, i = iBGP 수신)
show ip route bgp                   ! Routing Table의 BGP 경로 (B)
show ip route connected             ! Connected 확인
show ip bgp neighbors               ! Peer 상세
```

### BGP Table 표기

| 표기 | 의미 |
|------|------|
| `*>` | 유효 + Best (i 없음 = eBGP 수신 또는 로컬) |
| `*>i` | iBGP로 수신한 Best |
| `* i` | iBGP로 수신했으나 Best 아님 (Next-hop 도달 불가 등) |
| `Next Hop 0.0.0.0` | 해당 Router가 직접 광고한 네트워크 |
| `Weight 32768` | 로컬에서 생성한 네트워크 기본 Weight |
| Path `(64512) 100` | Confederation Sub-AS 경유 (괄호) |

---

## 7. Debug 및 초기화

```cisco
debug ip bgp events
undebug all
clear ip bgp *
```

---

## 8. 빠른 장애 대응

| 증상 | 즉시 확인 | 조치 |
|------|-----------|------|
| Peer `Idle` (eBGP Loopback) | `ebgp-multihop` | `neighbor X ebgp-multihop 3` |
| Peer `Active` | `remote-as`, neighbor IP, `update-source` | 설정 대조 |
| `peer in wrong AS` | `remote-as` / Confederation identifier | 수정 |
| R4·R5 미수신 | iBGP Split-horizon | Full-mesh / RR / Confederation |
| BGP Table `* i` Best 없음 | Next-hop 도달성 | Static / 재분배 / next-hop-self |
| 광고 안 됨 | `network` mask | `network X mask M` |

---

## 9. 암기 요약

- **TCP 179**, eBGP AD **20**, iBGP AD **200**
- Loopback Peer = `update-source`, eBGP는 **`ebgp-multihop`**
- Subnetting된 `network`는 **`mask`**
- Split-horizon = **Full-mesh / Route-reflector-client / Confederation**
- Next-hop = **Static / Redistribute / next-hop-self**
