# BGP 이론 (BGPv4)

> **Tag:** #Network #Cisco #BGP #Routing #EGP #PathVector #AS #iBGP #eBGP
> **핵심 요약:** **AS(Autonomous System) 사이에서 Routing 정보를 교환하는 대표적인 EGP** — Path Vector 방식으로 AS_PATH, LOCAL_PREF, MED, NEXT_HOP 등 Attribute 기반의 Routing Policy에 따라 Best Path를 결정하며, TCP 179번으로 BGP Peer(iBGP/eBGP)를 연결한다.

---

## 1. 핵심 기술 개념 (Concept)

### IGP와 EGP의 차이

- RIPv1/v2, EIGRP, OSPF, IS-IS 등은 **IGP(Interior Gateway Protocol)** 계열 Routing Protocol이며 하나의 AS 내부에서 네트워크 정보를 교환하기 위해 사용한다.
- IGP는 하나의 조직이나 관리 영역 내부에서 빠른 경로 계산과 빠른 수렴을 목적으로 사용한다.
- IGP는 각 Routing Protocol에서 사용하는 Metric을 이용하여 최적 경로를 결정한다.

| IGP | Metric |
|-----|--------|
| RIP | Hop Count |
| OSPF | Cost |
| EIGRP | Bandwidth, Delay 등을 이용한 Composite Metric |

- **EGP(Exterior Gateway Protocol)** 계열 Routing Protocol은 서로 다른 AS(Autonomous System) 사이에서 네트워크 정보를 교환하기 위해 사용한다.
- 현재 인터넷에서 사용되는 대표적인 EGP Routing Protocol은 **BGP**이다.
- BGP는 ISP와 ISP, ISP와 기업, 대규모 Data Center 및 대규모 Network 연결 등에 사용된다.
- BGP는 IGP처럼 단순한 Metric 값을 이용해서 최적 경로를 선택하지 않는다.
- BGP는 `AS_PATH`, `LOCAL_PREF`, `MED`, `NEXT_HOP` 등의 다양한 Attribute를 이용하여 관리자가 설정한 Routing Policy에 따라 최적 경로를 결정한다.
- BGP Routing Update에는 목적지 Network 정보뿐만 아니라 해당 Network에 도달하기 위한 여러 BGP Attribute 정보가 포함된다.

---

### BGPv4 (Border Gateway Protocol) 핵심 특징

- BGP는 서로 다른 AS 사이에서 Routing 정보를 교환하기 위한 대표적인 EGP Routing Protocol이다.
- BGP는 **Path Vector** 방식의 Routing Protocol이다.
- BGP Network는 목적지까지 거쳐온 AS 정보를 `AS_PATH`에 기록한다.
- `AS_PATH`에 자신의 AS 번호가 포함된 Routing Update를 수신하면 Loop가 발생한 것으로 판단하여 해당 경로를 사용하지 않는다.
- **Classless** Routing Protocol이다. (VLSM 지원, CIDR 지원)
- BGP는 Network에 변경이 발생했을 경우 전체 Routing Table을 주기적으로 전송하는 것이 아니라 변경된 Routing 정보에 대해서 **UPDATE Message**를 전달한다.
- BGP는 AS(Autonomous System)를 기반으로 동작한다.
- BGP는 과거에는 2-Byte(16 Bit) AS 번호를 사용하였지만 현재는 4-Byte(32 Bit) AS 번호를 지원한다.

| 구분 | AS 번호 범위 | 비고 |
|------|--------------|------|
| 2-Byte AS | 1 ~ 65535 | 65535는 특수 용도로 예약되어 있으므로 일반적인 AS 번호로 사용하지 않는다. |
| Private AS | 64512 ~ 65534 | 2-Byte Private AS |
| Private AS | 4200000000 ~ 4294967294 | 4-Byte Private AS |

- BGP 인접 관계를 **BGP Peer 또는 BGP Neighbor**라고 한다.
- BGP Routing Update는 BGP Peer 사이에서 교환된다.
- BGP Peer는 크게 iBGP와 eBGP로 구분한다.

| 구분 | 의미 |
|------|------|
| iBGP (Internal BGP) | 동일한 AS 내부에서 BGP Peer를 연결 |
| eBGP (External BGP) | 서로 다른 AS 사이에서 BGP Peer를 연결 |

- BGP는 **TCP Port 179번**을 이용하여 BGP Session을 연결한다.
- BGP는 TCP를 사용하기 때문에 TCP 3-Way Handshake가 먼저 완료된 후 BGP Peer 연결 과정이 진행된다.
- Cisco Router 기준 기본 AD 값은 **eBGP : 20 / iBGP : 200** 이다.
- BGP는 기본적으로 하나의 목적지 Network에 대해서 하나의 Best Path를 선택한다.
- 필요한 경우 BGP Multipath 등의 기능을 이용하여 여러 경로를 사용할 수도 있다.
- BGP는 다양한 BGP Attribute를 이용하여 Best Path를 선출한다.
- BGP는 `AS_PATH`를 이용하여 AS 단위의 Routing Loop를 방지한다.
- iBGP에서는 추가적으로 **"iBGP Peer에게 학습한 Route는 다른 iBGP Peer에게 전달하지 않는다."** 라는 규칙이 적용된다.
- Cisco IOS의 일반적인 구성에서는 하나의 Router에서 하나의 BGP AS Process를 구성한다.

```cisco
Router(config)# router bgp 65000
```

---

### AS (Autonomous System)

- AS는 하나의 관리 정책 아래에서 운영되는 Router와 Network들의 집합을 의미한다.
- Router, 네트워크들이 모여서 생성한 지역 도메인이다.
- 각 AS는 다른 AS와 BGP를 이용하여 Routing 정보를 교환할 수 있다.
- 수많은 AS가 서로 연결되어 전 세계 Internet을 구성한다. (AS와 AS들이 연결되어 통신망의 집합체를 인터넷이라고 한다.)
- AS 번호는 **ASN(Autonomous System Number)** 이라고 한다.
- 현재 BGP는 4-Byte AS 번호를 지원한다.
- 과거 2-Byte 기준 AS 범위는 1 ~ 65535이며, 사설 AS는 64512 ~ 65535 범위를 사용한다. (65535는 예약 번호이므로 실제 일반 Private AS는 64512 ~ 65534)
- Private AS는 `64512 ~ 65534`, `4200000000 ~ 4294967294` 이다.
- **Public AS**는 ISP, 대기업, Cloud 사업자 등에서 사용하며 일반적으로 RIR을 통해 할당받는다.
- **Private AS**는 조직 내부, Lab, 사설 BGP 환경 등에 사용할 수 있다.
- Private AS 번호가 Global Internet으로 그대로 전파되지 않도록 주의해야 한다.
- BGP에서는 AS와 AS 사이의 기술적인 경로뿐만 아니라 ISP와 Customer 사이의 계약이나 Routing Policy도 매우 중요하다.
- **Customer AS는 절대 Transit AS가 되면 안 된다.** (ISP, Backbone이 Transit AS 수행)
- BGP는 AS와 AS 간 이윤 관계와 민감하다.

---

### BGP 사용이 필요한 경우 / 적당하지 않은 경우

| BGP 사용이 필요한 경우 | BGP 사용이 적당하지 않은 경우 |
|-----------------------|-------------------------------|
| 여러 AS와 연결해야 할 경우 | 하나의 ISP와 단순하게 연결된 소규모 Network |
| 둘 이상의 ISP와 연결되는 Multi-homing 환경을 구성할 경우 | Default Route 하나만으로 외부 통신이 가능한 Network |
| Routing Policy를 적용해야 할 경우 | Routing Policy가 필요하지 않은 단순한 Network |
| AS와 AS 사이에서 대량의 Routing 정보를 교환해야 할 경우 | Router의 CPU 또는 Memory 자원이 부족한 경우 |
| 특정 Traffic의 유입 또는 유출 경로를 제어해야 할 경우 | BGP Routing Table을 유지해야 할 필요가 없는 경우 |
| Internet Routing을 구성할 경우 | |
| MPLS L3 VPN에서 MP-BGP를 구성할 경우 | |
| 대규모 Data Center에서 BGP 기반 Routing을 구성할 경우 | |

> 이러한 환경(오른쪽)에서는 **Static Default Route** 등을 사용하는 것이 더 간단할 수 있다.

---

### BGP 테이블

원문에서는 "BGP 4 가지 테이블"로 표기되어 있으나, 확인 명령어는 아래 2개가 명시되어 있다.

| 구분 | 확인 명령어 |
|------|-------------|
| BGP 상태 Table | `show ip bgp summary` |
| BGP 포워딩 Table | `show ip bgp` |

---

### BGP 메시지

BGP는 TCP 179번을 사용하여 Session을 연결한다.

| Message | 역할 |
|---------|------|
| Open | TCP 3-Way 통신 후 BGP를 연결하는 Message |
| Update | BGP 라우팅 업데이트 정보 |
| Keepalive | BGP Peer 연결 후 인접 관계 유지 점검, 1분(60초) 주기로 교환 실시 |
| Notification | BGP 상태 연결 문제 시 사용 및 해결 |

---

### BGP Peer 단계 (State)

```text
Idle → Connected → OpenSent → OpenConfirm → Established
                       ↓
                    Active (Open 수신 실패 시)
```

| State | 설명 |
|-------|------|
| Idle | BGP가 시작하는 단계, BGP Process에서 `neighbor` command로 지정한 Peer를 찾는 단계 |
| Connected | TCP 3-Way 핸드셰이킹을 끝낸 상태 |
| OpenSent | BGP Peer에게 Open 메시지를 전달하고, 자기는 Open 메시지 수신을 대기하는 상태 |
| OpenConfirm | Open 수신 상태 |
| Established | BGP Peer 성립, Keepalive 메시지만 교환 (60초 주기) |
| Active | OpenSent에서 BGP Peer에게 Open을 수신하지 못했다면 전환 (BGP 관련 설정이 잘못된 경우가 많음) |

- OpenSent 단계에서 BGP Peer에게 Open을 수신하게 되면 `OpenConfirm` 상태로 전환된다.
- OpenSent 단계에서 BGP Peer에게 Open 수신을 못했다면 `Active` 상태로 전환된다. (BGP 관련 설정이 잘못된 경우가 많음)

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. BGP Process 기본 구성

```cisco
router bgp 100                                      ! AS 번호 지정 (Router당 하나의 BGP AS Process)
 bgp router-id 1.1.1.1                              ! BGP Router-ID 지정
 no auto-summary                                    ! Classful 자동 요약 비활성화
 no synchronization                                 ! BGP 동기화 비활성화
 neighbor 13.13.12.2 remote-as 200                  ! BGP Peer 지정 (AS 번호가 다르면 eBGP, 같으면 iBGP)
!
```

> 실제 Peer 연결(물리/논리 Interface, Multihop)과 `network` 광고는 **BGP Peer 연결** 문서를 참고한다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```cisco
R1# show ip bgp summary          ! BGP Peer 상태 및 수신 Prefix 수 확인 (State/PfxRcd)
R1# show ip bgp                  ! BGP Table (Forwarding Table) 확인
R1# show ip route bgp            ! Routing Table에 등록된 BGP 경로 확인
```

### 3-2. BGP 구성 시 주의사항

#### 1) BGP 동기화 (BGP Synchronization)

- BGP로 수신한 네트워크는 **IGP로 확인되어야 한다.**
- BGP 동기화가 되지 않으면 Routing Table에서는 확인되지만 통신은 불가능한 **Blackhole-point** 현상이 발생한다.
- BGP는 경로상의 중간 Router에 BGP가 동작하지 않아도 BGP Peer를 연결할 수 있다. (BGP Peer 연결은 **NLRI**가 확립된 환경에서만 가능)
- **NLRI (Network Layer Reachability Information)** : 네트워크 계층(IP)의 정보가 확인되면 BGP Peer를 연결할 수 있다.
- 동일 AS 내의 Router 중 BGP가 동작하지 않는 Router가 포함되면 BGP 동기화 문제에 의해서 Blackhole-point 현상이 발생할 수 있다.

#### 2) BGP Split-horizon

- iBGP Peer에게 수신한 네트워크는 **iBGP Peer에게 송신할 수 없다.** (BGP는 국가망 또는 광역 네트워크를 연결 시 사용하는 Protocol이므로 강력한 Split-horizon이 적용된다.)
- BGP Split-horizon 문제를 해결하기 위해서 3가지 방식을 사용할 수 있다.
  - BGP **Full-mesh** 방식
  - BGP **Route-reflector** 방식
  - BGP **Confederation** 방식

> 해결 방법은 **BGP Split-horizon 문제 해결** 문서를 참고한다.

#### 3) BGP Next-hop

- BGP로 라우팅하는 네트워크는 **동일 AS 내에서 Next-hop이 변경되지 않는다.**
- BGP의 Next-hop은 **다른 AS로 라우팅 시에만** 변경된다.
- BGP Next-hop 문제는 **eBGP Peer 구간이 DMZ인 환경**에서 문제가 발생한다.

> 해결 방법은 **BGP Next-hop 문제 해결** 문서를 참고한다.

### 3-3. Case별 트러블슈팅 가이드

#### Case A. **Peer가 Active/Idle에서 Established로 올라오지 않음**

- **원인 파악:** `neighbor` IP 주소 또는 `remote-as` 오입력, 상대 Router의 `neighbor` 미설정, Peer IP 간 Routing(NLRI) 미확립 등 BGP 관련 설정 오류가 대부분이다.
- **확인:** `show ip bgp summary`의 State/PfxRcd 컬럼에서 Idle / Active 상태 여부를 확인한다.
- **해결:** 양쪽 Router의 `neighbor` IP와 `remote-as`를 대조하고, Peer IP 주소 간 통신이 가능한지 확인한다.

#### Case B. **BGP Table에는 보이지만 통신이 되지 않음 (Blackhole)**

- **원인 파악:** BGP 동기화 문제 — BGP로 수신한 네트워크가 IGP로 확인되지 않는 경우(동일 AS 내 BGP 미동작 Router 포함) 발생한다.
- **해결:** IGP 경로 확립 또는 모든 Transit Router에 BGP 구성 및 `no synchronization` 정책 점검.
