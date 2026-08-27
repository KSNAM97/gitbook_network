# VPN 명령어 퀵 레퍼런스

> **Tag:** #Network #VPN #GRE #IPsec #DMVPN #Cisco  
> **핵심 요약:** GRE·mGRE·NHRP·IPsec·CBAC·GRE-over-IPsec·DMVPN·IPsec 이중화의 설정 명령과 검증 명령을 한 곳에서 확인하는 암기 카드입니다. GRE는 캡슐화만 제공하고 암호화는 IPsec이 담당하며, 모든 설정 블록 뒤에는 반드시 `show` 검증을 붙입니다. 마지막의 함정표와 빠른 진단 순서를 우선 암기하십시오.

---

## 1. 요구사항별 기술 선택

| 요구사항 | 사용 기술 |
|---|---|
| 사설 대역을 공중망 위로 전달 | GRE Tunnel |
| 원격 사설망 경로 자동 교환 | GRE + 동적 라우팅 |
| 터널 1개로 다수 Peer 수용 | mGRE (`tunnel mode gre multipoint`) |
| Tunnel IP ↔ 공인 IP 매핑 | NHRP |
| 암호화·무결성·인증 | IPsec (ESP) |
| 세션 상태 기반 방화벽 | CBAC (신규 설계는 ZBF) |
| 동적 라우팅과 Multicast 암호화 | GRE-over-IPsec |
| 다수 지사 확장 + Spoke 직접 통신 | DMVPN |
| 본사 VPN 장비 단일 장애점 제거 | IPsec 이중화 (HSRP 연동) |

```properties
GRE   = 캡슐화 (암호화 아님)
IPsec = 암호화 + 무결성 + 인증
NHRP  = 매핑 해석
DMVPN = mGRE + NHRP + IPsec Profile + 동적 라우팅
```

---

## 2. 기술 비교

| 항목 | GRE | IPsec | GRE-over-IPsec | DMVPN |
|---|:---:|:---:|:---:|:---:|
| 암호화 | X | O | O | O |
| Multicast | O | X | O | O |
| 동적 라우팅 | O | 직접 불가 | O | O |
| 확장성 | 낮음 | 낮음 | 중간 | 높음 |
| Peer 주소 사전 필요 | O | O | O | Spoke 불필요 |
| 오버헤드 | 작음 | 중간 | 큼 | 큼 |

| 프로토콜/포트 | 용도 |
|---|---|
| IP Protocol 47 | GRE |
| IP Protocol 50 | ESP |
| IP Protocol 51 | AH (NAT 통과 곤란) |
| UDP 500 | IKE 협상 |
| UDP 4500 | NAT-T |

---

## 3. GRE Static

```cisco
configure terminal
!
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel destination <PEER-IP>
 tunnel mode gre ip
 ip mtu 1400
 ip tcp adjust-mss 1360
!
ip route 192.168.20.0 255.255.255.0 Tunnel <TUNNEL-NUMBER>
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip interface brief
show ip route <PEER-IP>
show ip route
ping <IP-ADDRESS>
```

```properties
Tunnel Up 조건
[1] tunnel source Up
[2] tunnel destination 라우팅 가능
[3] 그 경로가 Tunnel 자신을 사용하지 않음
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 4. GRE Dynamic

```cisco
configure terminal
!
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source Loopback0
 tunnel destination <PEER-IP>
 ip mtu 1400
 ip tcp adjust-mss 1360
!
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface Tunnel <TUNNEL-NUMBER>
 network 172.16.100.0 0.0.0.255
 network 192.168.10.0
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip eigrp interfaces
show ip eigrp neighbors
show ip route eigrp
```

OSPF로 운용하는 경우:

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip ospf network point-to-multipoint
!
router ospf 1
 network 172.16.100.0 0.0.0.255 area 0
```

검증:

```cisco
show ip ospf neighbor
show ip ospf interface tunnel <TUNNEL-NUMBER>
show ip route ospf
```

Recursive Routing 방어:

```cisco
ip prefix-list DENY_UNDERLAY seq 5 deny 121.160.0.0/16 le 32
ip prefix-list DENY_UNDERLAY seq 10 permit 0.0.0.0/0 le 32
!
router eigrp 100
 distribute-list prefix DENY_UNDERLAY in Tunnel <TUNNEL-NUMBER>
```

검증:

```cisco
show ip route <PEER-IP>
show interface tunnel <TUNNEL-NUMBER>
show logging
```

```properties
%TUN-5-RECURDOWN
= tunnel destination 경로가 Tunnel 자신으로 선택됨
= Longest Prefix Match → AD → Metric 순서로 발생
```

---

## 5. mGRE & NHRP

Hub (NHS):

```cisco
configure terminal
!
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp map multicast dynamic
 no ip split-horizon eigrp 100
```

Spoke (NHC):

```cisco
configure terminal
!
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp nhs <IP-ADDRESS>
 ip nhrp map <IP-ADDRESS> <PEER-IP>
 ip nhrp map multicast <PEER-IP>
```

검증:

```cisco
show ip nhrp
show ip nhrp brief
show ip nhrp nhs detail
show interface tunnel <TUNNEL-NUMBER>
show ip route
```

| 명령 | 의미 |
|---|---|
| `tunnel mode gre multipoint` | mGRE 활성, `tunnel destination` 미지정 |
| `ip nhrp network-id` | NHRP 도메인 식별자, Hub/Spoke 일치 |
| `ip nhrp authentication` | NHRP 인증, Hub/Spoke 일치 |
| `ip nhrp nhs` | Spoke가 등록할 NHS(Hub) Tunnel IP |
| `ip nhrp map` | Tunnel IP ↔ NBMA IP 정적 매핑 |
| `ip nhrp map multicast` | Multicast/Broadcast 복제 대상 NBMA |
| `ip nhrp map multicast dynamic` | Hub가 Spoke 등록을 동적 수용 |

```properties
NHS = Next Hop Server = Hub
NHC = Next Hop Client = Spoke

show ip nhrp
static  = 수동 매핑
dynamic = 학습된 매핑
```

---

## 6. IPsec Site-to-Site (Crypto Map)

### 6-1. Phase 1

```cisco
configure terminal
!
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256
 hash sha256
 group 14
 lifetime 3600
!
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
crypto isakmp keepalive 10 3
```

검증:

```cisco
show crypto isakmp policy
show crypto isakmp sa
show crypto isakmp peer
```

### 6-2. Phase 2

```cisco
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec security-association lifetime seconds 3600
```

검증:

```cisco
show crypto ipsec transform-set
show crypto ipsec sa
```

### 6-3. Crypto ACL (관심 트래픽)

```cisco
ip access-list extended VPN_TRAFFIC
 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
```

상대 장비는 반대로 작성합니다.

```cisco
ip access-list extended VPN_TRAFFIC
 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```

검증:

```cisco
show access-lists VPN_TRAFFIC
show crypto map
```

### 6-4. Crypto Map 및 적용

```cisco
crypto map <NAME> 10 ipsec-isakmp
 set peer <PEER-IP>
 set transform-set <NAME>
 set pfs group14
 match address <ACL-NUMBER>
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show crypto map
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show crypto engine connections active
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

### 6-5. NAT 예외

```cisco
ip access-list extended NAT_EXCEPT
 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
!
ip nat inside source list NAT_EXCEPT interface <INTERFACE> overload
```

검증:

```cisco
show access-lists NAT_EXCEPT
show ip nat translations
show crypto ipsec sa
```

```properties
[1] VPN 대상 deny
[2] 나머지 permit
순서가 뒤바뀌면 VPN 트래픽이 변환되어 협상 실패
```

### 6-6. 알고리즘 강도

| 구분 | 최신 권장 | 레거시 |
|---|---|---|
| 암호화 | `encryption aes 256`, `esp-gcm 256` | `encryption des`, `encryption 3des` |
| 해시 | `hash sha256`, `esp-sha256-hmac` | `hash md5`, `esp-md5-hmac` |
| DH Group | `group 14` 이상 | `group 1`, `group 2` |

> 실제 지원 여부는 IOS 버전·License·플랫폼별 확인이 필요합니다. Packet Tracer 등 학습 도구는 IPsec 알고리즘 지원이 제한적일 수 있습니다.

### 6-7. IKEv2 개요

```cisco
crypto ikev2 proposal <NAME>
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy <NAME>
 proposal <NAME>
!
crypto ikev2 profile <NAME>
 match identity remote address <PEER-IP> 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local <NAME>
```

검증:

```cisco
show crypto ikev2 sa
show crypto ikev2 proposal
show crypto ikev2 profile
show crypto session
```

---

## 7. CBAC

```cisco
configure terminal
!
ip inspect name <NAME> tcp
ip inspect name <NAME> udp
ip inspect name <NAME> icmp
!
ip access-list extended ACL_IN
 permit udp host <PEER-IP> eq 500 any eq 500
 permit udp host <PEER-IP> eq 4500 any eq 4500
 permit esp host <PEER-IP> any
 permit gre host <PEER-IP> any
 permit ospf host <PEER-IP> host 224.0.0.5
 deny   ip any any log-input
!
interface <INTERFACE>
 ip access-group ACL_IN in
 ip inspect <NAME> out
```

검증:

```cisco
show ip inspect config
show ip inspect interfaces
show ip inspect session
show ip access-lists ACL_IN
```

임계치 기반 방어:

```cisco
ip inspect max-incomplete high 500
ip inspect max-incomplete low 400
ip inspect one-minute high 1000
ip inspect one-minute low 800
ip inspect tcp max-incomplete host 50 block-time 2
```

검증:

```cisco
show ip inspect config
show ip inspect session
show ip inspect all
```

```properties
ACL           = 상태 없는 정적 필터
Reflexive ACL = L3/L4 역방향 미러링
CBAC          = 세션 상태 추적 + Application 검사
ZBF           = Zone 기반 정책, 신규 설계 권장
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 8. GRE-Over-IPsec

### 8-1. GRE 터널 + 동적 라우팅

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel destination <PEER-IP>
 ip mtu 1400
 ip tcp adjust-mss 1360
!
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface Tunnel <TUNNEL-NUMBER>
 network 172.16.100.0 0.0.0.255
 network 192.168.10.0
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip eigrp neighbors
show ip route eigrp
```

### 8-2. IPsec으로 GRE 보호

```cisco
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256
 hash sha256
 group 14
 lifetime 7200
!
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
!
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode transport
!
access-list <ACL-NUMBER> permit gre host <IP-ADDRESS> host <PEER-IP>
!
crypto map <NAME> 10 ipsec-isakmp
 set peer <PEER-IP>
 set transform-set <NAME>
 match address <ACL-NUMBER>
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show crypto map
show crypto engine connections active
show ip eigrp neighbors
show ip route eigrp
```

```properties
적용 위치별 캡슐화
Tunnel IF + Tunnel Mode    = [New IP][GRE][New IP][ESP][Original]  → New IP 2개
물리 IF + Tunnel Mode      = [New IP][ESP][New IP][GRE][Original]  → New IP 2개
물리 IF + Transport Mode   = [New IP][ESP][GRE][Original]          → New IP 1개 (권장)
```

> Crypto ACL에 라우팅 프로토콜 Multicast(224.0.0.5 / 224.0.0.10)를 직접 매칭하면 `%CRYPTO-4-RECVD_PKT_NOT_IPSEC`와 함께 인접이 끊깁니다. GRE(Protocol 47)를 대상으로 지정하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 9. DMVPN

### 9-1. IPsec Profile

```cisco
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256
 hash sha256
 group 14
 lifetime 3600
!
! Hub : Spoke 주소를 사전에 몰라도 수용
crypto isakmp key 0 <PASSWORD> address 0.0.0.0 0.0.0.0
!
! Spoke : Hub 공인 IP 지정
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
!
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile <NAME>
 set transform-set <NAME>
```

검증:

```cisco
show crypto isakmp policy
show crypto ipsec profile
show crypto isakmp sa
```

### 9-2. Hub 터널

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp map multicast dynamic
 ip nhrp redirect
 no ip split-horizon eigrp 100
 no ip next-hop-self eigrp 100
 tunnel protection ipsec profile <NAME>
```

### 9-3. Spoke 터널

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip address <IP-ADDRESS> 255.255.255.0
 tunnel source <INTERFACE>
 tunnel mode gre multipoint
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp nhs <IP-ADDRESS>
 ip nhrp map <IP-ADDRESS> <PEER-IP>
 ip nhrp map multicast <PEER-IP>
 ip nhrp shortcut
 tunnel protection ipsec profile <NAME>
```

검증:

```cisco
show dmvpn
show dmvpn detail
show ip nhrp
show ip nhrp nhs detail
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show ip route eigrp
show ip cef <IP-ADDRESS>
```

### 9-4. Phase별 명령

| Phase | Hub 명령 | Spoke 명령 | 데이터 경로 |
|---|---|---|---|
| Phase 1 | `no ip split-horizon eigrp <AS>` | 기본 설정 | Hub 경유 |
| Phase 2 | `no ip next-hop-self eigrp <AS>` 추가 | 기본 설정 | Spoke 직접 |
| Phase 3 | `ip nhrp redirect` | `ip nhrp shortcut` | Shortcut으로 직접 |

```properties
Phase 판별
Next-Hop이 Hub Tunnel IP      → Phase 1 또는 Phase 3 초기
Next-Hop이 상대 Spoke Tunnel IP → Phase 2
Phase 3는 show ip route만으로 판단 불가 → show ip nhrp / show ip cef 병행
```

> Packet Tracer 등 학습 환경은 DMVPN과 NHRP Shortcut 지원이 제한적일 수 있습니다.

---

## 10. IPsec 이중화 (HSRP 연동)

### 10-1. 본사 HSRP

```cisco
interface <INTERFACE>
 standby 1 ip <IP-ADDRESS>
 standby 1 priority 120
 standby 1 preempt delay minimum 60
 standby 1 track <INTERFACE> 100
!
interface <INTERFACE>
 standby 2 ip <IP-ADDRESS>
 standby 2 priority 120
 standby 2 preempt delay minimum 60
 standby 2 track <INTERFACE> 100
 standby 2 name IPSEC_R
```

검증:

```cisco
show standby brief
show standby all
show ip interface brief
```

### 10-2. Crypto Map 연동

```cisco
crypto isakmp keepalive 10 3
!
crypto map <NAME> 10 ipsec-isakmp
 set peer <PEER-IP>
 set transform-set <NAME>
 match address <ACL-NUMBER>
!
interface <INTERFACE>
 no crypto map <NAME>
 crypto map <NAME> redundancy IPSEC_R
```

검증:

```cisco
show standby brief
show crypto isakmp sa
show crypto isakmp peer
show crypto ipsec sa
show crypto session
show crypto map
```

### 10-3. 원격 지사

```cisco
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
crypto isakmp keepalive 10 3
!
crypto map <NAME> 10 ipsec-isakmp
 set peer <PEER-IP>
 set transform-set <NAME>
 match address <ACL-NUMBER>
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show crypto session
```

```properties
원격지 Peer 주소 = 본사 HSRP Virtual IP
물리 IP를 Peer로 쓰면 절체 시 VPN 불통

재수렴 시간
= HSRP 감지 + Track 반영 + SA 정리(DPD) + IKE 재협상 + 라우팅 재수렴
```

Gateway 이중화 자체는 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)를 참고하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 11. MTU 및 MSS

```cisco
interface Tunnel <TUNNEL-NUMBER>
 ip mtu 1400
 ip tcp adjust-mss 1360
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip interface tunnel <TUNNEL-NUMBER>
ping <IP-ADDRESS> size 1400 df-bit
ping <IP-ADDRESS> size 1500 df-bit
```

```properties
캡슐화 순서
Original IP → GRE Header → IPsec ESP → New IP Header

오버헤드 개념
New IP Header    = 20 Byte
GRE Header       = 4 Byte 이상
ESP Header/Trailer = 알고리즘 의존
NAT-T UDP Header = 8 Byte
```

| 증상 | 의심 지점 |
|---|---|
| 작은 Ping 성공, 큰 패킷 실패 | MTU/MSS 미조정 |
| HTTPS Handshake 이후 정지 | MSS 미조정 |
| 파일 전송만 실패 | Fragmentation 또는 DF bit |
| CPU 상승과 함께 지연 | Fragmentation 처리 부하 |

---

## 12. 검증 명령어

### Tunnel

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip interface brief
show ip interface tunnel <TUNNEL-NUMBER>
show running-config interface tunnel <TUNNEL-NUMBER>
```

### NHRP / DMVPN

```cisco
show ip nhrp
show ip nhrp brief
show ip nhrp nhs detail
show dmvpn
show dmvpn detail
```

### IPsec

```cisco
show crypto isakmp sa
show crypto isakmp policy
show crypto isakmp peer
show crypto ipsec sa
show crypto ipsec transform-set
show crypto map
show crypto session
show crypto engine connections active
```

### 라우팅 / NAT / ACL

```cisco
show ip route
show ip route <PEER-IP>
show ip eigrp neighbors
show ip ospf neighbor
show ip nat translations
show access-lists <ACL-NUMBER>
```

### 이중화

```cisco
show standby brief
show standby all
```

### Log / Debug

```cisco
show logging
debug crypto isakmp
debug crypto ipsec
undebug all
```

> ⚠️ `debug crypto` 계열은 운영 장비에서 CPU 부하와 Console 폭주를 유발할 수 있습니다. 범위를 좁히고 Maintenance Window에서 수행하며 확인 즉시 해제하십시오.

---

## 13. 출력 판독

### `show crypto isakmp sa`

| State | 의미 |
|---|---|
| MM_NO_STATE | Main Mode 초기 또는 실패 |
| MM_SA_SETUP | SA 파라미터 합의 |
| MM_KEY_EXCH | DH 키 교환, PSK 불일치 시 정지 |
| MM_KEY_AUTH | 인증 단계 |
| AG_INIT_EXCH | Aggressive Mode 진행 |
| QM_IDLE | Phase 1 완료, 정상 |

### `show crypto ipsec sa`

```properties
# pkts encaps 증가 + #pkts decaps 증가 = 양방향 정상
# pkts encaps 증가 + #pkts decaps 0 = 상대측 반환 경로·ACL 문제
# pkts encaps 0 = Crypto ACL 미매칭 또는 Crypto Map 미적용

local ident / remote ident
= Proxy Identity, 양단이 서로 반대로 일치해야 함
```

### `show ip nhrp`

| 표시 | 의미 |
|---|---|
| static | 관리자 수동 매핑 |
| dynamic | NHS가 학습한 매핑 |
| NBMA address | Tunnel IP에 대응하는 공인 IP |

### `show dmvpn`

| 표시 | 의미 |
|---|---|
| Peer NBMA Addr | Peer 공인 IP |
| Peer Tunnel Add | Peer Tunnel IP |
| State UP | 세션 활성 |
| Attrb S | Static 매핑 |
| Attrb D | Dynamic 매핑 |

### `show standby brief`

```properties
Active  = Virtual IP 소유, IPsec SA 대표 처리
Standby = SA 없음(정상)
```

---

## 14. 안전한 변경 원칙

```properties
[1] 현재 확인 → [2] 백업 → [3] 경로 확인 → [4] 적용 → [5] 검증 → [6] 저장
```

변경 전:

```cisco
show running-config
show ip route
show interface tunnel <TUNNEL-NUMBER>
show crypto isakmp sa
show crypto ipsec sa
show crypto map
show ip nat translations
show access-lists
```

변경 후:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip nhrp
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show ip route
show ip eigrp neighbors
```

저장:

```cisco
copy running-config startup-config
```

Rollback 대비:

```cisco
reload in 10
! 변경 및 검증 수행
reload cancel
```

```properties
Rollback 순서
[1] 인터페이스에서 no crypto map <NAME>
[2] Tunnel shutdown 후 파라미터 원복
[3] ACL / NAT 원래 내용 재구성
[4] 추가한 Static Route 및 Distribute-List 제거
[5] show 명령으로 원상 복귀 확인
[6] 정상 확인 전에는 저장하지 않음
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 15. 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| GRE Tunnel 구성 완료 | 캡슐화만 적용, 암호화는 없음 |
| Tunnel Up/Up | Underlay 도달성만 확인, 사설망 통신 보장 아님 |
| `tunnel source`만 설정 | Tunnel은 Down 상태 유지 |
| mGRE에 `tunnel destination` 지정 | 잘못된 설정, mGRE는 지정하지 않음 |
| Tunnel 대역을 Overlay에서 광고 | Recursive Routing 위험 |
| AD만 비교해 경로 판단 | Longest Prefix Match가 우선 |
| Ping 성공 | MTU 문제가 없다는 증거가 아님 |
| `ip tcp adjust-mss` 설정 | TCP만 적용, UDP는 해당 없음 |
| AH 사용 | 암호화 없음, NAT 통과 곤란 |
| ESP 사용 | 암호화 제공, NAT-T로 NAT 통과 |
| Transport Mode | New IP Header를 추가하지 않음 |
| Tunnel Mode | New IP Header 추가로 오버헤드 증가 |
| Phase 1 QM_IDLE | Phase 2 성립을 의미하지 않음 |
| Crypto ACL 단방향 작성 | 양단 미러가 아니면 Phase 2 실패 |
| Crypto ACL에 Multicast 매칭 | 라우팅 인접 단절 유발 |
| Crypto Map 설정만 완료 | 인터페이스에 적용해야 동작 |
| NAT 문장에 VPN 대역 permit 선행 | VPN 트래픽이 변환되어 실패 |
| UDP 500만 허용 | NAT 환경에서는 UDP 4500도 필요 |
| CBAC 적용 | IKE/ESP/GRE를 명시 허용하지 않으면 VPN 불통 |
| DMVPN에 Crypto Map 사용 | Peer 주소 사전 필요, 확장성 저하 |
| Hub Split Horizon 유지 | Spoke 간 경로 학습 실패 |
| Tunnel 기본 OSPF Network Type | Point-to-Point, 다수 인접 불가 |
| HSRP Virtual IP만 Peer로 지정 | `redundancy` 연동 없으면 SA 승계 안 됨 |
| HSRP Group `name` 미지정 | `crypto map ... redundancy` 사용 불가 |
| Standby 장비에 SA 없음 | 정상 동작 |
| `debug crypto` 상시 실행 | 운영 부하 유발, 즉시 해제 필요 |
| 3DES / MD5 / DH Group 2 | 레거시, AES / SHA-2 / Group 14 이상 권장 |
| Packet Tracer 검증 성공 | 실장비 동작을 보장하지 않음 |

---

## 16. 빠른 장애 진단 순서

```properties
하부 라우팅 확인 (show ip route <PEER-IP>)
    ↓
Tunnel 상태 확인 (show interface tunnel <TUNNEL-NUMBER>)
    ↓
Recursive Routing 로그 확인 (show logging)
    ↓
NHRP 등록 확인 (show ip nhrp / show dmvpn)
    ↓
IKE Phase 1 확인 (show crypto isakmp sa → QM_IDLE)
    ↓
IKE Phase 2 확인 (show crypto ipsec sa)
    ↓
Crypto ACL 대칭 확인 (show crypto map / show access-lists)
    ↓
encaps / decaps 카운터 확인
    ↓
NAT 예외 확인 (show ip nat translations)
    ↓
MTU / MSS 확인 (ping size df-bit)
    ↓
Overlay 라우팅 확인 (show ip route / show ip eigrp neighbors)
    ↓
이중화 상태 확인 (show standby brief / show crypto isakmp peer)
```

---

## 17. 명령 한눈에 보기

| 목적 | 명령 |
|---|---|
| GRE 터널 생성 | `interface Tunnel <TUNNEL-NUMBER>` |
| 터널 출발지 | `tunnel source <INTERFACE>` |
| 터널 목적지 | `tunnel destination <PEER-IP>` |
| GRE 모드 | `tunnel mode gre ip` |
| mGRE 모드 | `tunnel mode gre multipoint` |
| Tunnel MTU | `ip mtu 1400` |
| TCP MSS | `ip tcp adjust-mss 1360` |
| NHRP 도메인 | `ip nhrp network-id 1234` |
| NHRP 인증 | `ip nhrp authentication <PASSWORD>` |
| NHS 지정 | `ip nhrp nhs <IP-ADDRESS>` |
| NHRP 매핑 | `ip nhrp map <IP-ADDRESS> <PEER-IP>` |
| Multicast 매핑 | `ip nhrp map multicast <PEER-IP>` |
| 동적 등록 수용 | `ip nhrp map multicast dynamic` |
| Phase 3 Hub | `ip nhrp redirect` |
| Phase 3 Spoke | `ip nhrp shortcut` |
| Split Horizon 해제 | `no ip split-horizon eigrp <AS>` |
| Next-Hop 유지 | `no ip next-hop-self eigrp <AS>` |
| OSPF 타입 변경 | `ip ospf network point-to-multipoint` |
| Phase 1 정책 | `crypto isakmp policy 10` |
| Pre-Shared Key | `crypto isakmp key 0 <PASSWORD> address <PEER-IP>` |
| DPD | `crypto isakmp keepalive 10 3` |
| Transform-set | `crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac` |
| Transport Mode | `mode transport` |
| Crypto Map | `crypto map <NAME> 10 ipsec-isakmp` |
| Peer 지정 | `set peer <PEER-IP>` |
| 관심 트래픽 | `match address <ACL-NUMBER>` |
| Crypto Map 적용 | `crypto map <NAME>` (인터페이스) |
| HSRP 연동 | `crypto map <NAME> redundancy <HSRP-GROUP>` |
| IPsec Profile | `crypto ipsec profile <NAME>` |
| 터널 보호 | `tunnel protection ipsec profile <NAME>` |
| CBAC 규칙 | `ip inspect name <NAME> tcp` |
| CBAC 적용 | `ip inspect <NAME> out` |

---

> 📌 **핵심 요약**
> - GRE 터널: `tunnel source` · `tunnel destination` · `tunnel mode gre ip`
> - mGRE: `tunnel mode gre multipoint` (destination 지정 없음)
> - NHRP: `ip nhrp network-id` · `ip nhrp authentication` · `ip nhrp nhs` · `ip nhrp map`
> - MTU/MSS: `ip mtu 1400` · `ip tcp adjust-mss 1360`
> - Phase 1: `crypto isakmp policy` (암호화·해시·인증·DH Group·Lifetime)
> - Phase 2: `crypto ipsec transform-set` + Crypto ACL + PFS
> - Crypto Map 적용: 물리 인터페이스에 `crypto map <NAME>`
> - GRE-over-IPsec: Crypto ACL을 `permit gre`로 지정하고 `mode transport` 사용
> - DMVPN: `crypto ipsec profile` + `tunnel protection ipsec profile`
> - DMVPN Phase: 1(Hub 경유) → 2(`no ip next-hop-self`) → 3(`ip nhrp redirect` / `ip nhrp shortcut`)
> - IPsec 이중화: HSRP `name` + `crypto map <NAME> redundancy <HSRP-GROUP>`
> - 상태 확인: `show crypto isakmp sa`(QM_IDLE) · `show crypto ipsec sa`(encaps/decaps)
> - 터널 확인: `show interface tunnel <TUNNEL-NUMBER>` · `show ip nhrp` · `show dmvpn`
> - GRE는 캡슐화이고 암호화는 IPsec이 담당한다.
> - AH는 암호화가 없고 NAT 통과가 곤란하므로 ESP를 사용한다.
> - Crypto ACL은 양단 미러, NAT는 VPN 대역 `deny` 선행이 필수다.
> - 관련: 5-3-1-1. 🚇 GRE Static · 5-3-1-2. 🔄 GRE Dynamic · 5-3-1-3. 🌐 mGRE & NHRP · 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE) · 5-3-3. 🛡️ CBAC (Stateful Firewall) · 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화) · 5-3-5. 🌐 DMVPN (동적 멀티포인트 VPN) · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동) · 5-3-7. 🧩 VPN 통합 정리 (GRE · IPsec · DMVPN) · 5-3-8. 🚑 VPN 트러블슈팅 치트시트 · 5-3-9. ⚡ VPN 명령어 퀵 레퍼런스
