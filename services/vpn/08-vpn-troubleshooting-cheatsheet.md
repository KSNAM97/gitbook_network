# VPN 트러블슈팅 치트시트

> **Tag:** #Network #VPN #GRE #IPsec #DMVPN #Cisco  
> **핵심 요약:** VPN 장애는 **하부 라우팅 → Tunnel 상태 → IKE Phase 1 → Phase 2 → Crypto ACL 대칭 → NAT 예외 → MTU → 라우팅** 순서로 확인합니다. Phase 1이 QM_IDLE인데 통신이 안 되면 Phase 2·Crypto ACL·NAT 예외를, Ping은 되는데 큰 패킷만 실패하면 MTU/MSS를 우선 의심합니다. GRE는 암호화를 제공하지 않으므로 터널이 올라왔다는 사실이 보안 적용을 의미하지 않습니다.

---

## 1. 공통 진단 순서

```properties
[1] 하부 라우팅(Underlay) : Tunnel destination 공인 IP에 도달 가능한가?
[2] Tunnel 상태           : Tunnel Interface가 Up/Up인가? Recursive Routing 로그가 있는가?
[3] IKE Phase 1           : show crypto isakmp sa가 QM_IDLE인가?
[4] IKE Phase 2           : show crypto ipsec sa에 SA가 형성됐는가?
[5] Crypto ACL 대칭       : 양단 match address가 서로 미러인가?
[6] NAT 예외              : VPN 대상 트래픽이 NAT에서 제외됐는가?
[7] MTU / MSS             : ip mtu와 ip tcp adjust-mss가 설정됐는가?
[8] 라우팅(Overlay)       : 원격 사설망 경로가 Tunnel을 향하는가?
```

계층을 섞지 않는 것이 핵심입니다.

```properties
Underlay 문제 = 공인 IP 도달성, 물리 회선, 공중망 라우팅
Overlay 문제  = Tunnel IP 통신, 터널 위 라우팅, 사설망 경로
Crypto 문제   = Phase 1/2 협상, Crypto ACL, Transform-set, Peer 파라미터

경로 선택 판단 = [1] Longest Prefix Match [2] AD [3] Metric
AD만 단독 비교하면 Recursive Routing 원인을 놓칩니다.
```

---

## 2. 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| Tunnel Down/Down | Tunnel destination 미도달 | `show ip route <PEER-IP>` | Underlay 라우팅 복구 |
| Tunnel Up/Down 반복 | Recursive Routing | `show logging` | Endpoint 대역 Overlay 광고 제외 |
| Tunnel IP는 되고 사설망 불통 | 원격 사설망 경로 누락 | `show ip route` | Static Route 또는 라우팅 추가 |
| Tunnel Up인데 EIGRP 인접 없음 | `passive-interface` 미해제 | `show ip eigrp interfaces` | Tunnel에 `no passive-interface` |
| OSPF 인접이 1개만 형성 | Tunnel Type Point-to-Point | `show ip ospf interface tunnel <TUNNEL-NUMBER>` | `ip ospf network point-to-multipoint` |
| Spoke끼리 경로 미학습 | Hub Split Horizon | Hub 설정 확인 | `no ip split-horizon eigrp <AS>` |
| Spoke NHRP 등록 실패 | network-id/authentication 불일치 | `show ip nhrp nhs detail` | Hub/Spoke 값 일치 |
| Phase 1 SA 없음 | 정책·PSK·Peer 불일치, UDP 500 차단 | `show crypto isakmp sa` | 파라미터 일치 및 경로 허용 |
| Phase 1 QM_IDLE, Phase 2 없음 | Crypto ACL 비대칭, Transform-set 불일치 | `show crypto ipsec sa` | ACL 미러 정정 |
| encaps만 증가, decaps 0 | 상대측 반환 경로·ACL 불일치 | `show crypto ipsec sa` | 상대 설정·경로 점검 |
| VPN은 되고 인터넷 불통 | NAT 예외 순서 오류 | `show access-lists` | VPN 대역 `deny` 선행 |
| Ping은 되고 큰 패킷 실패 | MTU/MSS 미조정 | `ping ... df-bit` | `ip mtu 1400`, `ip tcp adjust-mss 1360` |
| Crypto 적용 후 라우팅 단절 | Crypto ACL에 Multicast 직접 매칭 | `show logging` | GRE(47) 대상으로 변경 |
| NAT 뒤 Site에서 VPN 실패 | NAT-T 미허용, AH 사용 | `show crypto isakmp sa` | UDP 4500 허용, ESP 사용 |
| HSRP 절체 후 VPN 불통 | `redundancy` 연동 누락 | `show standby brief` | HSRP `name` + `redundancy` 적용 |
| Spoke 직접 통신 안 됨 | Phase 2/3 명령 누락 | `show dmvpn` | `no ip next-hop-self` 또는 redirect/shortcut |
| CBAC 적용 후 VPN 불통 | Inbound ACL에서 IKE/ESP 차단 | `show ip access-lists` | IKE·ESP·GRE 허용 |
| SA 주기적 단절 | Lifetime 불일치, DPD 부재 | `show logging` | Lifetime 정렬 및 DPD 설정 |

---

## 3. 핵심 진단 명령어

### Tunnel 및 NHRP

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip interface brief
show running-config interface tunnel <TUNNEL-NUMBER>
show ip nhrp
show ip nhrp nhs detail
show dmvpn
```

| 항목 | 확인 내용 |
|---|---|
| line protocol | Up/Up 여부 |
| Tunnel source / destination | 의도한 주소인지 |
| Tunnel protocol | GRE/IP 또는 multi-GRE/IP |
| ip mtu / transport MTU | 캡슐화 여유 확보 여부 |
| NHRP static / dynamic | 수동 매핑과 학습 매핑 구분 |
| NHS state RE | Registration 정상 완료 |

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

`show crypto isakmp sa` 판독:

| State | 의미 | 판단 |
|---|---|---|
| MM_NO_STATE | Main Mode 시작 또는 실패 | 정책·PSK·경로 확인 |
| MM_SA_SETUP | SA 파라미터 합의 | 진행 중 |
| MM_KEY_EXCH | DH 키 교환 | PSK 불일치 시 여기서 정지 |
| MM_KEY_AUTH | 인증 단계 | 인증 정보 확인 |
| AG_INIT_EXCH | Aggressive Mode 진행 | 모드 일치 확인 |
| QM_IDLE | Phase 1 완료 정상 | Phase 2로 진행 |

`show crypto ipsec sa` 판독:

```properties
# pkts encaps 증가 + #pkts decaps 증가 = 양방향 정상
# pkts encaps 증가 + #pkts decaps 0 = 송신만 됨, 상대 ACL·경로·ESP 차단 의심
# pkts encaps 0 = 관심 트래픽 미매칭, Crypto ACL·적용 인터페이스 확인

local ident / remote ident = Proxy Identity, 양단이 서로 반대로 일치해야 함
```

### 라우팅 · NAT · CBAC · Log

```cisco
show ip route <PEER-IP>
show ip eigrp neighbors
show ip ospf neighbor
show ip nat translations
show access-lists <ACL-NUMBER>
show ip inspect config
show ip inspect interfaces
show ip inspect session
show standby brief
show logging
debug crypto isakmp
debug crypto ipsec
```

> ⚠️ `debug crypto isakmp`와 `debug crypto ipsec`은 운영 장비에서 CPU 부하와 Console 폭주를 유발할 수 있습니다. 대상 Peer로 범위를 좁히고 Maintenance Window에서 수행하며 확인 즉시 `undebug all`로 해제하십시오.

---

## 4. 대표 장애 시나리오

### 시나리오 1. Tunnel Interface가 Down/Down

#### 원인 후보

1. `tunnel destination` 주소가 라우팅 테이블에 없음
2. `tunnel source` 인터페이스가 Down
3. `tunnel destination` 누락
4. Underlay 라우팅이 공인 대역을 학습하지 못함
5. mGRE인데 `tunnel destination`을 지정
6. Tunnel Interface가 `shutdown` 상태

#### 확인

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip route <PEER-IP>
show running-config interface tunnel <TUNNEL-NUMBER>
```

#### 조치

```cisco
interface Tunnel <TUNNEL-NUMBER>
 tunnel source <INTERFACE>
 tunnel destination <PEER-IP>
 tunnel mode gre ip
 no shutdown
```

검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip route <PEER-IP>
show ip interface brief
```

```properties
GRE Tunnel Up 조건
[1] source Up  [2] destination 라우팅 가능  [3] 그 경로가 Tunnel 자신이 아님
```

---

### 시나리오 2. Tunnel이 Up/Down을 반복하고 %TUN-5-RECURDOWN 발생

#### 증상

```properties
%TUN-5-RECURDOWN: TunnelX temporarily disabled due to recursive routing
```

#### 원인

Tunnel 위 라우팅 프로토콜이 `tunnel destination` 대역을 광고해 그 목적지 경로가 Tunnel 자신이 됩니다.

```properties
[1] Longest Prefix Match로 Tunnel 경로 선택
[2] destination을 Tunnel로 캡슐화해야 하는 모순
[3] Tunnel Down 후 재시도 반복
```

#### 확인

```cisco
show logging
show ip route <PEER-IP>
show ip protocols
```

#### 조치

```cisco
interface Tunnel <TUNNEL-NUMBER>
 tunnel source Loopback0
!
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

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 3. Tunnel IP는 통신되지만 사설망끼리 통신 안 됨

#### 핵심 판단

```properties
Tunnel Up/Up
≠ 원격 사설망 경로 자동 생성
≠ 상대 LAN 도달성 확보
```

#### 원인 후보

- 원격 사설망 Static Route 누락
- 라우팅 프로토콜에 Tunnel 대역 미포함
- 상대 라우터에도 역방향 경로 없음
- 사설망 Gateway 또는 SVI Down
- 중간 방화벽에서 사설 대역 차단

#### 확인

```cisco
ping <IP-ADDRESS>
show ip route
show ip route 192.168.20.0
```

#### 조치

```cisco
ip route 192.168.20.0 255.255.255.0 Tunnel <TUNNEL-NUMBER>
!
router eigrp 100
 no auto-summary
 no passive-interface Tunnel <TUNNEL-NUMBER>
 network 172.16.100.0 0.0.0.255
 network 192.168.10.0
```

검증:

```cisco
show ip route
show ip eigrp neighbors
show ip route eigrp
```

---

### 시나리오 4. Tunnel은 Up인데 라우팅 프로토콜 인접이 형성되지 않음

#### 원인 후보

1. `passive-interface default` 상태에서 Tunnel을 열지 않음
2. `network` 문에 Tunnel 대역 누락
3. EIGRP AS 번호 또는 OSPF Area 불일치
4. OSPF Network Type 불일치
5. mGRE에서 `ip nhrp map multicast` 누락으로 Hello 미전달
6. Tunnel 양단 Subnet Mask 불일치
7. 경로상 ACL 또는 CBAC이 라우팅 프로토콜 차단

#### 확인

```cisco
show ip eigrp interfaces
show ip eigrp neighbors
show ip ospf interface tunnel <TUNNEL-NUMBER>
show running-config interface tunnel <TUNNEL-NUMBER>
```

#### 조치

```cisco
router eigrp 100
 no passive-interface Tunnel <TUNNEL-NUMBER>
 network 172.16.100.0 0.0.0.255
!
interface Tunnel <TUNNEL-NUMBER>
 ip ospf network point-to-multipoint
```

검증:

```cisco
show ip eigrp neighbors
show ip ospf neighbor
show ip ospf interface tunnel <TUNNEL-NUMBER>
```

라우팅 프로토콜 예외 허용은 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리를 참고하십시오.

---

### 시나리오 5. Spoke가 Hub에 NHRP 등록되지 않음

#### 원인 후보

1. `ip nhrp network-id` 불일치
2. `ip nhrp authentication` 불일치
3. Spoke에 `ip nhrp nhs <IP-ADDRESS>` 누락
4. Spoke에 Hub Tunnel IP ↔ 공인 IP 정적 매핑 누락
5. Hub에 `ip nhrp map multicast dynamic` 누락
6. Hub 공인 IP에 Spoke가 도달하지 못함
7. IPsec Phase 1 실패로 보호 터널 미형성
8. 경로상 GRE(47) 또는 ESP(50) 차단

#### 확인

```cisco
show ip nhrp
show ip nhrp nhs detail
show crypto isakmp sa
show running-config interface tunnel <TUNNEL-NUMBER>
```

#### 조치

```cisco
! Hub
interface Tunnel <TUNNEL-NUMBER>
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp map multicast dynamic
!
! Spoke
interface Tunnel <TUNNEL-NUMBER>
 ip nhrp network-id 1234
 ip nhrp authentication <PASSWORD>
 ip nhrp nhs <IP-ADDRESS>
 ip nhrp map <IP-ADDRESS> <PEER-IP>
 ip nhrp map multicast <PEER-IP>
```

검증:

```cisco
show ip nhrp
show ip nhrp nhs detail
show dmvpn
```

---

### 시나리오 6. Phase 1이 형성되지 않음 (SA 없음 또는 MM_NO_STATE)

#### 원인 후보

1. Phase 1 파라미터 불일치(암호화·해시·인증 방식·DH Group)
2. Pre-Shared Key 불일치
3. `crypto isakmp key ... address <PEER-IP>`의 Peer 주소 오지정
4. 경로상 UDP 500 차단
5. NAT 환경인데 UDP 4500 차단
6. Crypto Map을 인터페이스에 적용하지 않음
7. 관심 트래픽이 발생하지 않아 협상이 시작되지 않음
8. 상대가 Virtual IP를 사용하는데 물리 IP를 Peer로 지정

#### 확인

```cisco
show crypto isakmp policy
show crypto isakmp sa
show crypto isakmp peer
show crypto map
show logging
```

#### 조치

```cisco
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256
 hash sha256
 group 14
 lifetime 3600
!
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
!
ip access-list extended ACL_IN
 permit udp host <PEER-IP> eq 500 any eq 500
 permit udp host <PEER-IP> eq 4500 any eq 4500
 permit esp host <PEER-IP> any
```

검증:

```cisco
show crypto isakmp sa
show crypto session
show ip access-lists ACL_IN
```

```properties
MM_KEY_EXCH에서 멈춤 → Pre-Shared Key 불일치 가능성이 높음
SA 자체가 없음        → UDP 500 도달성 또는 관심 트래픽 발생 여부 확인
```

---

### 시나리오 7. Phase 1은 QM_IDLE인데 Phase 2가 형성되지 않음

#### 원인 후보

1. Crypto ACL이 양단에서 미러가 아님
2. Transform-set 알고리즘 불일치
3. Mode 불일치(한쪽 Transport, 다른 쪽 Tunnel)
4. PFS Group 불일치
5. Crypto Map을 인터페이스에 적용하지 않음
6. 적용 위치가 서로 다름(한쪽 물리, 다른 쪽 Tunnel)
7. Phase 2 Lifetime 차이
8. 관심 트래픽이 실제로 발생하지 않음

#### 확인

```cisco
show crypto ipsec sa
show crypto map
show crypto ipsec transform-set
show access-lists <ACL-NUMBER>
```

#### 조치

```properties
본사 : permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
지사 : permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```

```cisco
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode tunnel
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show crypto ipsec sa
show crypto map
show crypto session
```

`show crypto ipsec sa`의 `local ident`와 `remote ident`가 상대 장비와 반대로 일치하는지 확인합니다.

---

### 시나리오 8. encaps 카운터만 증가하고 decaps가 0

#### 판독

```properties
# pkts encaps: 1520, #pkts encrypt: 1520
# pkts decaps: 0, #pkts decrypt: 0
        ↓
송신 암호화는 정상, 수신 복호화가 전혀 발생하지 않음
```

#### 원인 후보

1. 상대 Crypto ACL이 반대 방향을 포함하지 않음
2. 상대에 원격 사설망 경로가 없어 응답이 터널로 오지 않음
3. 경로상 ESP(50) 또는 UDP 4500 차단
4. 상대에서 NAT 예외 처리 누락
5. 비대칭 경로로 응답이 다른 장비로 되돌아옴
6. 상대 Crypto Map이 다른 인터페이스에 적용됨

#### 확인 및 조치

```cisco
show crypto ipsec sa
show crypto engine connections active
show access-lists <ACL-NUMBER>
show ip route
show ip nat translations
```

재협상이 필요하면 서비스 영향을 고려해 수행합니다.

```cisco
clear crypto sa
show crypto ipsec sa
```

> ⚠️ `clear crypto sa`는 기존 세션을 강제로 끊습니다. Maintenance Window에서 수행하십시오.

---

### 시나리오 9. VPN 트래픽이 NAT로 변환되어 통신 실패

#### 증상

VPN이 전혀 성립하지 않거나, 인터넷은 되는데 VPN만 실패합니다.

#### 원인

NAT ACL에서 VPN 대상을 제외하지 않아 출발지가 공인 IP로 변환되어 Crypto ACL과 매칭되지 않습니다.

#### 확인

```cisco
show ip nat translations
show access-lists
show crypto ipsec sa
```

#### 조치

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
[1] VPN 대상 deny (변환 제외)
[2] 나머지 permit (인터넷 변환)
deny가 뒤에 있으면 이미 permit에 매칭되어 변환됩니다.
```

NAT 동작은 5-2. 🔀 NAT & PAT (주소 변환)을 함께 확인하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 10. Ping은 되지만 큰 패킷과 일부 TCP 세션만 실패

#### 증상

- 소형 Ping 성공, 웹 페이지 일부만 로딩
- 파일 전송 또는 대용량 응답에서 정지
- HTTPS Handshake 이후 멈춤

#### 원인

캡슐화 오버헤드로 전송 가능한 크기가 줄었는데 MTU/MSS를 조정하지 않았습니다. 경로상 ICMP가 차단되면 Path MTU Discovery도 동작하지 않습니다.

#### 확인

```cisco
show interface tunnel <TUNNEL-NUMBER>
ping <IP-ADDRESS> size 1500 df-bit
ping <IP-ADDRESS> size 1400 df-bit
```

#### 조치

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
```

```properties
GRE 오버헤드       ≈ 24 Byte
IPsec ESP 오버헤드 = 알고리즘·Mode 의존
NAT-T 오버헤드     ≈ 8 Byte
        ↓
1400 / 1360은 여유를 둔 보수적 기준값
```

> `ip tcp adjust-mss`는 TCP에만 적용됩니다. UDP 기반 대용량 트래픽은 별도 검토가 필요합니다.

---

### 시나리오 11. Crypto Map 적용 후 EIGRP/OSPF 인접이 끊김

#### 증상

```properties
%CRYPTO-4-RECVD_PKT_NOT_IPSEC: Rec'd packet not an IPSEC packet
```

#### 원인

Crypto ACL에 라우팅 Multicast(224.0.0.5, 224.0.0.10)를 직접 매칭했습니다. IPsec은 Unicast를 대상으로 하므로 Multicast를 그대로 보호할 수 없습니다.

#### 확인

```cisco
show logging
show access-lists <ACL-NUMBER>
show crypto map
show ip eigrp neighbors
```

#### 조치

```cisco
no access-list <ACL-NUMBER>
access-list <ACL-NUMBER> permit gre host <IP-ADDRESS> host <PEER-IP>
!
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
 mode transport
!
interface <INTERFACE>
 crypto map <NAME>
```

검증:

```cisco
show access-lists <ACL-NUMBER>
show crypto ipsec sa
show ip eigrp neighbors
show ip route eigrp
```

```properties
GRE   = 캡슐화 및 Multicast 전달
IPsec = 암호화 및 무결성
        ↓
GRE 안에 담긴 라우팅 프로토콜이 함께 보호됨
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 12. NAT 뒤에 있는 Site에서 VPN이 성립하지 않음

#### 원인 후보

1. AH를 사용하고 있음(암호화 없음, NAT 통과 곤란)
2. 경로상 UDP 4500 차단으로 NAT-T 불가
3. NAT-T가 비활성화됨
4. 동일 공인 IP 뒤에 여러 Peer가 존재해 식별 충돌
5. Peer 식별을 IP로만 수행해 유동 주소 대응 불가

#### 확인

```cisco
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show logging
```

#### 조치

```cisco
crypto ipsec transform-set <NAME> esp-aes 256 esp-sha256-hmac
!
ip access-list extended ACL_IN
 permit udp host <PEER-IP> eq 500 any eq 500
 permit udp host <PEER-IP> eq 4500 any eq 4500
```

검증:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show ip access-lists ACL_IN
```

```properties
AH (51)  = 암호화 없음, IP Header 무결성 포함, NAT 통과 곤란
ESP (50) + NAT-T (UDP 4500) = 암호화 제공, NAT 통과 가능
```

---

### 시나리오 13. CBAC 적용 후 VPN 또는 라우팅이 끊김

#### 원인 후보

1. Inbound ACL이 IKE(UDP 500) 차단
2. Inbound ACL이 ESP(50) 차단
3. Inbound ACL이 GRE(47) 차단
4. Inbound ACL이 라우팅 Multicast 차단
5. `ip inspect` 적용 방향 오류(나가는 방향에 적용해야 상태 기록)
6. Half-open 임계치가 낮아 정상 트래픽까지 정리 대상이 됨

#### 확인

```cisco
show ip inspect interfaces
show ip inspect config
show ip inspect session
show ip access-lists ACL_IN
```

#### 조치

```cisco
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
show ip access-lists ACL_IN
show ip inspect interfaces
show ip inspect session
show crypto ipsec sa
```

CBAC은 레거시 방식이므로 신규 설계에서는 ZBF를 검토하십시오. 지원 여부는 IOS 버전·License·플랫폼별 확인이 필요합니다.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 14. HSRP 절체 후 VPN이 넘어가지 않음

#### 원인 후보

1. HSRP Group에 `name`을 지정하지 않음
2. `crypto map <NAME> redundancy <HSRP-GROUP>` 연동 누락
3. 원격지가 물리 IP를 Peer로 지정
4. `crypto isakmp key ... address`가 Virtual IP가 아님
5. DPD 미설정으로 기존 SA 정리가 늦음
6. Track 미설정으로 반대편 회선 장애를 감지하지 못함

#### 확인

```cisco
show standby brief
show standby all
show crypto isakmp sa
show crypto isakmp peer
show logging
```

#### 조치

```cisco
! 본사 양쪽 장비
interface <INTERFACE>
 standby 2 ip <IP-ADDRESS>
 standby 2 preempt delay minimum 60
 standby 2 track <INTERFACE> 100
 standby 2 name IPSEC_R
!
crypto isakmp keepalive 10 3
!
interface <INTERFACE>
 no crypto map <NAME>
 crypto map <NAME> redundancy IPSEC_R
!
! 원격지 : Peer를 Virtual IP로 통일
crypto isakmp key 0 <PASSWORD> address <PEER-IP>
```

검증:

```cisco
show standby brief
show crypto isakmp peer
show crypto ipsec sa
show crypto session
```

```properties
Standby 장비에 SA가 없는 것은 정상입니다.
재수렴 시간 = HSRP 감지 + Track 반영 + SA 정리(DPD) + IKE 재협상 + 라우팅 재수렴
```

Gateway 이중화 동작은 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy)를 참고하십시오.

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 15. DMVPN에서 Spoke 간 직접 통신이 되지 않고 계속 Hub를 경유

#### Phase 판별

```properties
Next-Hop이 Hub Tunnel IP        → Phase 1 또는 Phase 3 초기 상태
Next-Hop이 상대 Spoke Tunnel IP → Phase 2
Phase 3는 라우팅이 Hub를 가리켜도 실제 전달은 Shortcut
        ↓ show ip route만으로 판단 불가
show ip nhrp 및 show ip cef 병행 확인
```

#### 원인 후보

1. Phase 2 기대인데 Hub에 `no ip next-hop-self eigrp <AS>` 누락
2. Phase 3 기대인데 Hub에 `ip nhrp redirect` 누락
3. Spoke에 `ip nhrp shortcut` 누락
4. Spoke 간 NHRP 조회 실패
5. Spoke 간 공인 IP 도달성 없음(GRE/ESP 차단)
6. Spoke가 NAT 뒤에 있어 직접 터널 형성이 제한됨

#### 확인

```cisco
show dmvpn
show ip nhrp
show ip route eigrp
show ip cef <IP-ADDRESS>
```

#### 조치

```cisco
! Phase 2
interface Tunnel <TUNNEL-NUMBER>
 no ip split-horizon eigrp 100
 no ip next-hop-self eigrp 100
!
! Phase 3 Hub
interface Tunnel <TUNNEL-NUMBER>
 ip nhrp redirect
!
! Phase 3 Spoke
interface Tunnel <TUNNEL-NUMBER>
 ip nhrp shortcut
```

검증:

```cisco
show dmvpn
show ip nhrp
show ip route eigrp
show crypto ipsec sa
```

> Packet Tracer 등 학습 환경은 DMVPN Phase 2/3와 NHRP Shortcut 지원이 제한적일 수 있습니다.

---

### 시나리오 16. IPsec SA가 주기적으로 끊기고 재협상됨

#### 원인 후보

1. Phase 1/Phase 2 Lifetime이 양단에서 크게 다름
2. DPD 미설정으로 죽은 SA가 오래 유지됨
3. Underlay 회선 품질 저하 또는 간헐 단절
4. NAT Translation Timeout으로 NAT-T 세션 소멸
5. 중복 Peer 설정으로 SA가 서로 교체됨

#### 확인

```cisco
show crypto isakmp sa
show crypto ipsec sa
show logging
show interface <INTERFACE>
```

#### 조치

```cisco
crypto isakmp policy 10
 lifetime 3600
!
crypto isakmp keepalive 10 3
!
crypto ipsec security-association lifetime seconds 3600
```

검증:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show logging
```

```properties
[1] Underlay 회선 오류 카운터 확인
[2] Lifetime 양단 비교
[3] DPD 동작 확인
[4] NAT Translation Timeout 확인
[5] 로그 시각과 단절 시각 상관관계 확인
```

---

## 5. 안전한 변경 및 복구 절차

```properties
[1] 현재 확인 → [2] 백업 → [3] 경로 확인 → [4] 적용 → [5] 검증 → [6] 저장
```

변경 전 수집:

```cisco
show running-config
show ip route
show interface tunnel <TUNNEL-NUMBER>
show ip nhrp
show crypto isakmp sa
show crypto ipsec sa
show crypto map
show ip nat translations
show access-lists
show standby brief
```

변경 후 검증:

```cisco
show interface tunnel <TUNNEL-NUMBER>
show ip nhrp
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show ip route
show ip eigrp neighbors
```

Rollback 절차:

```properties
[1] 변경 전 running-config를 외부에 보관
[2] 인터페이스에서 no crypto map <NAME>
[3] Tunnel shutdown 후 파라미터 원복
[4] ACL / NAT 이전 내용 재구성 후 clear ip nat translation *
[5] 추가한 Static Route와 Distribute-List 제거
[6] show 명령으로 원상 복귀 확인
[7] 정상 확인 전에는 startup-config에 저장하지 않음
```

관리 연결 보호:

```cisco
reload in 10
! 변경 및 검증 수행
reload cancel
```

```properties
Crypto Map 적용, ACL·NAT·라우팅·Tunnel 변경
        ↓
관리 세션이 그 경로를 사용하면 즉시 단절 가능
대비 : Console/OOB 확보, 관리 대역 ACL 상단 허용, reload in 예약
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

> 📌 **핵심 요약**
> - VPN 장애는 하부 라우팅부터 확인하고 Overlay와 Crypto를 섞지 않는다.
> - Tunnel Up/Up은 Underlay 도달성만 의미하며 사설망 경로를 보장하지 않는다.
> - Recursive Routing은 Longest Prefix Match → AD → Metric 순서로 터널 경로가 선택될 때 발생한다.
> - Phase 1 판독은 MM_NO_STATE(실패·초기)와 QM_IDLE(정상 완료)을 기준으로 한다.
> - Phase 1이 정상인데 통신이 안 되면 Crypto ACL 대칭과 Transform-set을 먼저 본다.
> - `show crypto ipsec sa`의 encaps만 증가하면 상대측 반환 경로와 ACL을 점검한다.
> - VPN 대상 트래픽은 NAT ACL에서 반드시 먼저 `deny`로 제외한다.
> - Ping은 되는데 큰 패킷만 실패하면 MTU/MSS와 Fragmentation을 의심한다.
> - Crypto ACL에 라우팅 Multicast를 직접 매칭하면 인접이 끊기므로 GRE(47)를 대상으로 지정한다.
> - AH는 암호화를 제공하지 않고 NAT 통과가 곤란하므로 ESP와 NAT-T를 사용한다.
> - CBAC/ZBF Inbound 정책에서 IKE·ESP·GRE를 허용하지 않으면 VPN이 성립하지 않는다.
> - HSRP 절체 시 VPN 승계는 HSRP Group `name`과 `crypto map ... redundancy` 연동이 필요하다.
> - DMVPN Phase 3는 라우팅 테이블만으로 판단할 수 없어 `show ip nhrp`와 `show ip cef`를 병행한다.
> - `debug crypto` 계열은 운영 부하가 크므로 범위를 좁히고 즉시 해제한다.
> - 변경은 현재 확인 → 백업 → 경로 확인 → 적용 → 검증 → 저장 순서를 지키고 Rollback을 준비한다.
> - 관련: 5-3-1-1. 🚇 GRE Static · 5-3-1-2. 🔄 GRE Dynamic · 5-3-1-3. 🌐 mGRE & NHRP · 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE) · 5-3-3. 🛡️ CBAC (Stateful Firewall) · 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화) · 5-3-5. 🌐 DMVPN (동적 멀티포인트 VPN) · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동) · 5-3-7. 🧩 VPN 통합 정리 (GRE · IPsec · DMVPN) · 5-3-8. 🚑 VPN 트러블슈팅 치트시트 · 5-3-9. ⚡ VPN 명령어 퀵 레퍼런스
