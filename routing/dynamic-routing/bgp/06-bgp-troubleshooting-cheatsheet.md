# BGP 트러블슈팅 치트시트

> **Tag:** #Routing #BGP #Troubleshooting #Peer #Multihop #SplitHorizon #NextHop #Confederation
> **핵심 요약:** BGP 장애는 **Peer(State) → Prefix 수신(PfxRcd) → BGP Table(Best `>`) → Routing Table(Next-hop 도달성)** 순서로 확인한다. `Idle/Active`면 Peer 설정·Multihop·도달성을, Established인데 경로가 없으면 **Split-horizon**과 **Next-hop**을 먼저 의심한다.

---

## 1. 빠른 진단 순서

```text
1) show ip bgp summary          → State/PfxRcd (숫자면 Established, Idle/Active면 Peer 문제)
2) show ip bgp                  → 경로가 BGP Table에 있는가? Best(>) 표시가 있는가?
3) show ip route <Next-hop>     → Next Hop IP가 Routing Table에 있는가? (Next-hop 문제)
4) show ip route bgp            → Routing Table에 B 경로로 등록됐는가?
5) show running-config | section bgp → neighbor/remote-as/update-source/multihop 대조
```

| 확인 지점 | 정상 | 이상 시 의심 |
|-----------|------|--------------|
| State | 숫자(수신 Prefix 수) | Idle / Active → Peer 설정, 도달성, Multihop |
| PfxRcd | 기대한 개수 | 0 → 상대가 `network` 광고 안 함 / Split-horizon |
| BGP Table | `*>` Best | `* i` Best 없음 → Next-hop 도달 불가 |
| Routing Table | `B [20/0]` / `[200/0]` | 없음 → Next-hop 도달성, 동기화 |

---

## 2. Peer State별 진단

| State | 의미 | 주요 원인 | 조치 |
|-------|------|-----------|------|
| Idle | `neighbor`로 지정한 Peer를 찾는 단계 | Peer IP로의 Routing 없음, eBGP Multihop 부족(Loopback), 한쪽만 neighbor 설정 | 도달성·`ebgp-multihop`·양쪽 설정 확인 |
| Active | Open 수신 실패 | `remote-as` 불일치, Peer IP 오입력, `update-source` 불일치 | 양쪽 neighbor IP와 `remote-as`, `update-source` 대조 |
| OpenSent | Open 전송 후 대기 | 상대가 응답 안 함 | 상대 설정 확인 |
| OpenConfirm | Open 수신 | 일시 상태 | Keepalive 교환 확인 |
| Established | 정상 | — | Prefix 수신/광고 확인 |

---

## 3. 증상별 즉시 대응표

### 3-1. Peer 문제

| 증상 | 원인 | 확인 | 조치 |
|------|------|------|------|
| Loopback eBGP Peer가 `Idle`, Up/Down `never` | eBGP Multihop 기본값 1 | `show ip bgp summary` | 양쪽 `neighbor X ebgp-multihop 3` |
| iBGP Loopback Peer 안 올라옴 | `update-source` 누락 / Loopback 도달성 없음 | `show run \| s bgp`, `show ip route <Lo>` | `update-source loopback 0`, IGP로 Loopback 광고 |
| `%BGP-3-NOTIFICATION ... peer in wrong AS` | `remote-as`와 상대 AS(또는 Confederation Sub-AS) 불일치 | 로그, `show ip bgp neighbors` | `remote-as` 수정 / `bgp confederation identifier` 설정 |
| 설정 변경 후 Peer Down/Up | `route-reflector-client` 변경 시 정상 Reset | `%BGP-5-ADJCHANGE ... RR client config change` | 잠시 대기 후 Up 확인 |
| DMZ 구성 후 Loopback eBGP 끊김 | Loopback 네트워크 도달 불가 | `show ip route` | 물리 Interface IP로 Peer 재구성 |

### 3-2. 경로 문제

| 증상 | 원인 | 확인 | 조치 |
|------|------|------|------|
| R3까지만 수신, R4·R5 미수신 | iBGP Split-horizon | `show ip bgp`(R4,R5) 비어 있음 | Full-mesh / Route-reflector / Confederation |
| BGP Table에 있으나 `* i`(Best 없음), Routing Table 없음 | Next-hop 도달 불가 (DMZ) | `show ip bgp`의 Next Hop, `show ip route <Next-hop>` | Static / IGP 재분배 / `next-hop-self` |
| 네트워크가 광고되지 않음 | `network`의 mask 누락(Subnetting된 네트워크) | `show run \| s bgp`, `show ip route connected` | `network X mask M` 으로 정확히 지정 |
| Routing Table에 보이나 통신 불가 | BGP 동기화(Blackhole) / 중간 Router 경로 없음 | `traceroute`, `show ip route` | 모든 Transit Router에 경로 확보, `no synchronization` 정책 점검 |

---

## 4. Case별 트러블슈팅 가이드

### Case 1. Loopback으로 연결한 eBGP Peer가 Idle

#### 증상
- `show ip bgp summary`의 State가 `Idle`, Up/Down이 `never`.
- 같은 Router의 iBGP Loopback Peer는 정상.

#### 원인
- iBGP는 Multihop 기본값 255, **eBGP는 기본값 1**. BGP에서 Connected는 1 hop이므로 Loopback 간 연결은 부족하다.

#### 조치
```cisco
router bgp 100
 neighbor 13.13.2.2 remote-as 200
 neighbor 13.13.2.2 update-source loopback 0
 neighbor 13.13.2.2 ebgp-multihop 3
!
```
양쪽 모두 설정 후 `show ip bgp summary`에서 숫자(Prefix 수)로 바뀌는지 확인한다.

---

### Case 2. iBGP Peer는 올라왔는데 R4·R5에 BGP 네트워크가 없음

#### 증상
- R2·R3에는 `10.1.1.0/24` 등이 보이지만 R4·R5의 `show ip route bgp`, `show ip bgp`가 비어 있음.

#### 원인
- 체인형 iBGP Peer(R2-R3-R4-R5)에서 **iBGP로 수신한 Route는 다른 iBGP Peer에게 재광고하지 않는다.**

#### 조치
| 방식 | 핵심 |
|------|------|
| Full-mesh | 모든 AS 200 Router 간 iBGP Peer 구성 |
| Route-reflector | R2에 `neighbor <R3/R4/R5> route-reflector-client` |
| Confederation | Sub-AS 분할 + `bgp confederation identifier 200` / `peers` |

---

### Case 3. Route-reflector 구성 직후 Peer가 순간 Down

#### 증상
- `%BGP-5-ADJCHANGE: neighbor 13.13.3.3 Down RR client config change` 이후 곧 `Up`.

#### 원인
- `route-reflector-client` 지정 변경 시 Peer가 Reset되는 정상 동작.

#### 조치
- Up 로그 확인 후 `show ip bgp`에서 AS 200 내부 네트워크(`192.168.X.0`)가 모든 Router에 보이는지 확인한다.

---

### Case 4. Confederation 구성 시 `peer in wrong AS`

#### 증상
- `%BGP-3-NOTIFICATION: sent to / received from neighbor ... 2/2 (peer in wrong AS)`

#### 원인
- R2가 `router bgp 64512`(Sub-AS)로 동작하며 R1에게도 64512로 보임 → R1의 `remote-as 200`과 불일치.

#### 조치
```cisco
router bgp 64512
 bgp confederation identifier 200
!
```
모든 Confederation Router(R2~R5)에 동일 Identifier(200)를 설정하고, Sub-AS 경계(R3 ↔ R4)에는 `bgp confederation peers`, Loopback이면 `ebgp-multihop`도 구성한다.

---

### Case 5. BGP Table에는 있는데 Routing Table에 등록되지 않음 (Next-hop)

#### 증상
- AS 200 내부 R3에서 `show ip bgp`에 `* i10.1.1.0/24 13.13.12.1`처럼 **Best(`>`) 표시가 없음**.
- `show ip route bgp`에 해당 경로 없음.

#### 원인
- eBGP 구간(13.13.12.0/24)이 **DMZ(IGP 미구성)** 이고 동일 AS 내 Next-hop은 변경되지 않아, R3는 `13.13.12.1`로 가는 경로를 모른다.

#### 조치 (택1)
```cisco
! 1) Static (R3, R4, R5)
ip route 13.13.12.0 255.255.255.0 <내부 Next-hop>

! 2) IGP 재분배 (R2)
route-map EIGRP_REDI permit 10
 match interface serial 1/0.12
router eigrp 100
 redistribute connected metric 1544 2000 255 1 1500 route-map EIGRP_REDI

! 3) next-hop-self (R2)
router bgp 64512
 neighbor 13.13.3.3 next-hop-self
```

---

### Case 6. DMZ 재분배 후 다른 Loopback까지 전파됨

#### 증상
- `redistribute connected` 후 R3에서 `100.10.2.0/24`, `200.20.2.0/24`까지 보임.

#### 원인
- Route-map 없이 `connected` 전체를 재분배.

#### 조치
- `match interface serial 1/0.12`만 Permit 하거나, Loopback 100/200을 Deny 후 나머지를 Permit 한다.

```cisco
route-map EIGRP_REDI deny 10
 match interface loopback100 loopback200
route-map EIGRP_REDI permit 20
```

---

### Case 7. 광고한 네트워크가 상대 Router에 보이지 않음

#### 증상
- R1에서 `network`를 입력했지만 R2 `show ip route bgp`에 없음.

#### 원인
- Subnetting된 네트워크(`10.1.1.0/24`, `172.16.1.0/24`)에 **`mask`를 생략**하면 Classful 경계(예: `10.0.0.0/8`)로 해석되어, Routing Table에 정확히 일치하는 경로가 없으므로 광고되지 않는다.

#### 조치
```cisco
router bgp 100
 network 10.1.1.0 mask 255.255.255.0
 network 172.16.1.0 mask 255.255.255.0
 network 192.168.1.0
!
```

---

## 5. Debug 및 확인 명령어

```cisco
Router# show ip bgp summary
Router# show ip bgp
Router# show ip bgp neighbors
Router# show ip route bgp
Router# show ip route connected
Router# debug ip bgp events        ! 운영 환경에서는 신중히 사용
Router# clear ip bgp *             ! Peer 재설정 (영향 범위 주의)
```

---

## 6. 변경 전후 체크리스트

### 변경 전
- [ ] 현재 `show ip bgp summary` / `show ip bgp` 결과 저장
- [ ] Peer IP 도달성(`ping`, IGP 경로) 확인
- [ ] 변경 영향 Peer 범위 파악 (RR client / Confederation 변경은 Peer Reset 발생)

### 변경 후
- [ ] 모든 Peer가 Established(숫자) 상태
- [ ] 기대한 Prefix 수신 (`PfxRcd`)
- [ ] 모든 Router에서 Best(`>`) 경로 확인
- [ ] Routing Table에 `B` 경로 등록 및 Next-hop 도달성 확인

---

## 7. 핵심 요약

- `Idle/Active` → Peer 설정, **eBGP Loopback은 Multihop**, 도달성
- Established인데 일부 Router만 미수신 → **Split-horizon** (Full-mesh / RR / Confederation)
- BGP Table `* i` Best 없음 → **Next-hop 도달성** (Static / 재분배 / next-hop-self)
- Subnetting된 `network`는 **mask 필수**
- `peer in wrong AS` → `remote-as` 또는 **Confederation identifier**
