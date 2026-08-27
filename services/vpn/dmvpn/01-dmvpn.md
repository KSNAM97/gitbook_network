# DMVPN (Dynamic Multipoint VPN)

> **Tag:** #network #vpn #dmvpn #mgre #nhrp #ipsec-profile #cisco
> **핵심 요약:** mGRE + NHRP + IPsec + 라우팅을 결합한 확장성 최고의 대규모 Hub-Spoke VPN. Phase 1/2/3 이해가 핵심

> 🔗 선행: 5-3-1-3. 🌐 mGRE & NHRP · 5-3-4. 🔗 GRE-Over-IPsec

---

## 1. 핵심 기술 개념 (Concept)

### Site-to-Site VPN 대신 DMVPN을 쓰는 결정적 이유

일반 Site-to-Site VPN은 지점이 추가될 때마다 Hub에 터널·IPsec Peer를 일일이 추가해야 해서 지점 N개면 관리가 폭증한다. DMVPN은 Hub 설정 변경 없이 지점만 붙이면 자동으로 연결되고, 필요 시 지점끼리 직접(Spoke-to-Spoke) 통신까지 가능해 대규모 환경에서 확장성이 압도적이다.

DMVPN은 mGRE(터널 1개로 다수 Peer), NHRP(터널IP↔공인NBMA 매핑), IPsec(암호화), 동적 라우팅(경로 교환)의 4대 기술 결합이다. 멀티캐스트(라우팅 프로토콜)를 지원하고, 경로 장애 시 자동 재수렴하며, 최소 packet loss를 보장한다.

### DMVPN에서 IPsec을 crypto map이 아닌 IPsec Profile로 적용하는 이유

crypto map은 `set peer`로 상대 Peer 주소를 미리 알아야 하는데, DMVPN의 Spoke는 수시로 늘어나고 주소도 유동적이다. IPsec Profile은 Peer 주소를 몰라도 되며, NHRP 캐시로부터 Peer 정보를 동적으로 받아와 `tunnel protection ipsec profile`로 터널에 적용하므로 확장성이 뛰어나다.

Hub는 `crypto isakmp key ... address 0.0.0.0 0.0.0.0`으로 모든 Spoke의 Phase-1을 동적 수신 대기한다. 구성 요소는 IKE Phase1(관리채널) → Phase2(데이터 SA) → IPsec Profile(Phase2 정책) → Tunnel Protection(터널 전체 암호화) 순으로 이어진다.

### DMVPN Phase 1/2/3의 차이 (가장 중요한 시험 포인트)

Phase 1은 Spoke 간 통신이 항상 Hub 경유, Phase 2는 라우팅이 상대 Spoke를 넥스트홉으로 알려줘 Spoke끼리 직접 통신, Phase 3은 라우팅 넥스트홉은 Hub여도 되지만 NHRP Redirect/Shortcut으로 결국 직접 통신한다.

Phase 2에서는 Hub에서 `no ip next-hop-self eigrp`로 원래 Spoke 넥스트홉을 유지해야 Spoke가 상대 Spoke를 직접 가리킨다. Phase 3에서는 Hub에 `ip nhrp redirect`, Spoke에 `ip nhrp shortcut`을 설정해, Hub가 "직접 가라"고 Redirect를 보내면 Spoke가 Shortcut 경로를 생성한다. 경로 요약이 가능해 대규모에 최적이다. 공통적으로 Hub에서 `no ip split-horizon eigrp`가 필수인데, 한 Spoke 경로를 같은 터널로 다른 Spoke에 재광고하기 위함이다.

### DMVPN Phase 비교

| 구분 | Routing Next-hop | 실제 데이터 경로 | 핵심 명령 |
|------|:----------------:|:----------------:|-----------|
| **Phase 1** | Hub | Hub 경유 | no split-horizon |
| **Phase 2** | 상대 Spoke | 직접 | + no next-hop-self |
| **Phase 3** | Hub | 직접 (Shortcut) | Hub: nhrp redirect / Spoke: nhrp shortcut |

---

## 2. 표준 설정 템플릿 (Configuration)

### 1단계: mGRE + NHRP (Hub-Spoke)

```cisco
! === Hub (GIT-HQ) : NHRP Server ===
interface tunnel 123
 ip address 172.16.123.1 255.255.255.0
 tunnel source 121.160.14.1
 tunnel mode gre multipoint                ! mGRE
 ip nhrp network-id 1234
 ip nhrp authentication SOLnet
 ip nhrp map multicast dynamic             ! Spoke 동적 등록 수용

! === Spoke (GIT-A/B) : NHRP Client ===
interface tunnel 123
 ip address 172.16.123.2 255.255.255.0
 tunnel source 121.160.25.2
 tunnel mode gre multipoint
 ip nhrp network-id 1234
 ip nhrp authentication SOLnet
 ip nhrp nhs 172.16.123.1                  ! Hub(NHRP 서버) 터널IP
 ip nhrp map multicast 121.160.14.1        ! Hub 공인 IP 멀티캐스트 매핑
 ip nhrp map 172.16.123.1 121.160.14.1     ! Hub 터널IP↔공인IP 정적 매핑
```
### 2단계: 라우팅(EIGRP) - 터널 대역 포함 + split-horizon 해제

```cisco
! === Hub ===
router eigrp 100
 no passive-interface tunnel 123
 network 172.16.123.0 0.0.0.255
!
interface tunnel 123
 no ip split-horizon eigrp 100             ! Phase 1 필수
 no ip next-hop-self eigrp 100             ! Phase 2 추가

! === Spoke ===
router eigrp 100
 no passive-interface tunnel 123
 network 172.16.123.0 0.0.0.255
```

### 3단계: IPsec Profile로 터널 보호 (crypto map 아님)

```cisco
! === Hub : Peer 주소 몰라도 동적 수신 ===
crypto isakmp policy 10
 authentication pre-share
 encryption 3des
 hash sha
 group 2
 lifetime 900
!
crypto isakmp key 0 solcisco address 0.0.0.0 0.0.0.0   ! 모든 Spoke 동적 수신
!
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto ipsec profile DMVPN
 set transform-set IPSEC
!
interface tunnel 123
 tunnel protection ipsec profile DMVPN     ! 터널 전체를 IPsec으로 보호

! === Spoke : Hub 공인 IP를 key 주소로 ===
crypto isakmp key 0 solcisco address 121.160.14.1
! (나머지 profile/transform-set/tunnel protection은 Hub와 동일)
```

### Phase 3 (대규모 최적) 추가 설정

```cisco
! === Hub ===
interface tunnel 123
 ip nhrp redirect                          ! "직접 가라" 신호 발송
!
! === Spoke ===
interface tunnel 123
 ip nhrp shortcut                          ! Redirect 받아 직접 경로 생성
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Router# show ip nhrp                       ! 터널IP↔NBMA 매핑
Hub#    show ip nhrp dynamic               ! Hub가 동적 학습한 Spoke
Router# show ip route eigrp                 ! Phase별 넥스트홉 확인 (Hub vs Spoke)
Router# show crypto isakmp sa               ! Phase1 SA (QM_IDLE=정상)
Router# show crypto ipsec sa                ! 암/복호화 확인
Router# show ip cef [목적지]                 ! Phase 3 실제 전달 경로 확인
```

### Phase 판별법

- `show ip route eigrp`에서 원격 Spoke망의 넥스트홉이:
    - **Hub 터널IP(172.16.123.1)** → Phase 1 (또는 Phase 3의 초기 상태)
    - **상대 Spoke 터널IP(172.16.123.3)** → Phase 2
- Phase 3는 라우팅 테이블은 Hub를 가리켜도 실제 전달은 Shortcut → **`show ip route`만으로 판단 불가**, `show ip nhrp`/`show ip cef` 병행.

### Case 1: Spoke가 Hub에 NHRP 등록이 안 됨

- **원인 파악:** ① network-id/authentication 불일치, ② Spoke의 `ip nhrp nhs`(Hub 터널IP) 누락, ③ Hub의 `ip nhrp map multicast dynamic` 누락, ④ IPsec Phase1 실패로 터널 자체가 안 뜸.
- **해결:** 네 항목 순서대로 점검, `show ip nhrp dynamic`(Hub)에 Spoke가 뜨는지 확인.

### Case 2: Spoke끼리 직접 통신이 안 되고 계속 Hub만 경유 (Phase 2/3 기대인데)

- **원인 파악:** Hub에 **`no ip next-hop-self eigrp`(Phase 2)** 또는 **`ip nhrp redirect`(Phase 3)** 누락. Spoke에 `ip nhrp shortcut` 누락.
- **해결:** 원하는 Phase에 맞는 명령을 Hub/Spoke에 추가하고 `show ip route`의 넥스트홉 변화 확인.

### Case 3: 라우팅 프로토콜 경로가 Spoke끼리 안 배워짐

- **원인 파악:** Hub에서 **split-horizon**이 켜져 있어 한 Spoke 경로를 같은 터널의 다른 Spoke로 재광고 못 함.
- **해결:** Hub 터널에 `no ip split-horizon eigrp [AS]` 적용.

---

]
