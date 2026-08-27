# GRE-Over-IPsec (GRE + IPsec 결합)

> **Tag:** #network #vpn #gre #ipsec #dynamic-routing #transport-mode #cisco
> **핵심 요약:** GRE의 멀티캐스트 지원과 IPsec의 암호화를 결합해, 라우팅 프로토콜(EIGRP/OSPF)까지 암호화하는 VPN

---

## 1. 핵심 기술 개념 (Concept)

### IPsec만으로 충분한데 GRE와 결합하는 이유

순수 IPsec은 유니캐스트만 암호화할 수 있어 RIP/EIGRP/OSPF 같은 멀티캐스트 기반 라우팅 프로토콜을 보호하지 못한다. GRE는 멀티캐스트를 유니캐스트로 캡슐화할 수 있으므로, GRE로 라우팅을 실어 나르고 그 GRE를 IPsec으로 암호화하면 동적 라우팅까지 암호화 터널로 주고받을 수 있다.

순수 IPsec crypto map에 EIGRP를 매칭시키면 `%CRYPTO-4-RECVD_PKT_NOT_IPSEC` 에러와 함께 인접성이 끊긴다(멀티캐스트라 암호화 대상이 안 됨). GRE-over-IPsec에서는 IPsec의 보호 대상을 개별 트래픽이 아니라 GRE 프로토콜(proto 47) 전체로 지정하면, GRE 안에 담긴 EIGRP/실데이터가 통째로 보호된다.

### crypto map을 Tunnel 인터페이스에 걸 때와 물리 인터페이스에 걸 때의 차이

Tunnel 인터페이스에 걸면 IPsec이 먼저 암호화하고 그 위에 GRE가 캡슐화되어 New IP 헤더가 2개(GRE용 + IPsec용) 생겨 오버헤드가 크다. 물리 인터페이스에 걸면 GRE가 먼저 캡슐화한 뒤 IPsec이 보호해, 라우팅 프로토콜까지 자연스럽게 보호되고 헤더 낭비가 적다. 실무 표준은 물리 인터페이스 적용 + Transport 모드다.

물리 인터페이스에 걸 때는 ACL의 보호 대상을 `permit gre host [로컬공인] host [상대공인]`으로 지정하면 GRE 안의 모든 것(실데이터 + EIGRP)이 보호된다.

### GRE-over-IPsec에서 Transport 모드를 쓰는 이유

GRE가 이미 공인 IP로 캡슐화(터널링)를 했기 때문에, IPsec이 Tunnel 모드로 또 New IP 헤더를 붙이면 헤더가 중복된다. Transport 모드는 GRE가 만든 기존 외부 IP 헤더를 재사용하고 GRE 헤더부터 암호화하므로, 중복 캡슐화·오버헤드·Fragmentation을 줄인다.

역할 분담은 GRE가 터널링과 멀티캐스트 전달을, IPsec이 암호화·무결성·인증을 담당하는 구조다. GRE가 터널을 담당하므로 IPsec은 보안만 하면 되어 Transport가 최적이다. Transport 모드에선 외부 IP 헤더는 암호화 안 되지만, GRE 헤더부터 내부 전체가 보호된다.

### 캡슐화 순서 비교

| 적용 방식 | 캡슐화 구조 | New IP 헤더 |
|-----------|-------------|:-----------:|
| Tunnel IF + Tunnel모드 | `[New IP][GRE][New IP][ESP][원본]` | 2개 (비효율) |
| 물리 IF + Tunnel모드 | `[New IP][ESP][New IP][GRE][원본]` | 2개 |
| **물리 IF + Transport모드** | `[New IP][ESP][GRE][원본]` | **1개 (최적)** |

---

## 2. 표준 설정 템플릿 (Configuration)

### 1단계: GRE 터널 + 동적 라우팅(EIGRP) 구성

```cisco
! === GIT-A : GRE 터널 (공인 IP를 source/dest로) ===
interface tunnel 12
 ip address 192.168.12.1 255.255.255.0
 tunnel source 121.160.10.1
 tunnel destination 121.160.20.2
!
! === 터널 위에 EIGRP (사설망 + 터널 대역 포함) ===
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface tunnel 12          ! 터널로 EIGRP Hello 허용
 network 192.168.10.0
 network 192.168.12.0
```
### 2단계: crypto map을 물리 인터페이스에 적용 (권장) + GRE 보호

```cisco
! === GIT-A ===
! Phase 1
crypto isakmp policy 10
 authentication pre-share
 encryption 3des
 hash sha
 group 2
 lifetime 7200
crypto isakmp key 0 cisco address 121.160.20.2
!
! Phase 2 : Transport 모드 (GRE-over-IPsec 최적)
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
 mode transport                          ! Tunnel 모드가 아닌 Transport 모드
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
! 보호 대상 = GRE 프로토콜 전체 (안에 EIGRP+실데이터 모두 포함)
access-list 101 permit gre host 121.160.10.1 host 121.160.20.2
!
! crypto map은 물리 인터페이스에 적용
interface serial 1/0.10
 crypto map IPSEC_VPN
```

> ⚠️ **가장 흔한 실수 정정:** 원본의 `mode tranport`는 오타이며 올바른 명령은 `mode transport` 입니다.

### (참고) 최신 보안 강화 버전

```cisco
! 레거시(3DES/SHA1/DH2)는 취약 → 현대 표준으로 상향 권장
crypto isakmp policy 10
 encryption aes 256                      ! 3DES → AES-256
 hash sha256                             ! SHA1 → SHA-256
 group 14                                ! DH2(1024) → DH14(2048)
!
crypto ipsec transform-set IPSEC esp-gcm 256   ! AES-GCM (암호화+인증 결합, 고성능)
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Router# show crypto isakmp sa                  ! Phase1 SA (QM_IDLE=정상)
Router# show crypto engine connections active  ! 암/복호화 카운터 (핵심)
Router# show ip eigrp neighbors                 ! 터널 위 EIGRP 인접 확인
Router# show ip route eigrp                     ! 암호화된 채로 학습된 원격 사설망
Router# debug crypto ipsec                       ! Phase2 협상 추적
```

### 장애 시나리오 1 (핵심): IPsec 켜니 EIGRP/OSPF 인접이 끊김

- **증상:** crypto map 적용 후 `%CRYPTO-4-RECVD_PKT_NOT_IPSEC ... 224.0.0.10` 로그와 함께 인접성 down.
- **원인 파악:** IPsec 보호 대상 ACL에 **라우팅 프로토콜(멀티캐스트 224.0.0.x)을 직접 매칭**시킴. IPsec은 유니캐스트만 암호화하므로 멀티캐스트가 처리 실패.
- **해결:** ACL에서 라우팅 프로토콜을 빼고, **GRE 프로토콜 전체를 보호 대상**으로 지정 후 crypto map을 **물리 인터페이스**에 적용:
    
    ```cisco
    access-list 101 permit gre host [로컬공인] host [상대공인]
    interface serial 1/0.x
     crypto map IPSEC_VPN
    ```
    
    이렇게 하면 GRE 안에 담긴 EIGRP가 함께 암호화되어 인접이 유지됨.

### 장애 시나리오 2: 통신은 되는데 MTU 초과로 큰 패킷/일부 TCP만 실패

- **원인 파악:** Tunnel 모드로 IPsec을 걸어 **헤더가 중복 추가**되어 MTU를 초과 → Fragmentation/일부 TCP 세션 장애.
- **해결:** transform-set을 **`mode transport`** 로 변경하고 crypto map을 물리 인터페이스에 적용해 헤더 오버헤드를 최소화. 필요 시 `ip mtu`/`ip tcp adjust-mss` 조정.

### 장애 시나리오 3: Phase 1은 되는데 암호화 카운터가 안 오름

- **원인 파악:** 양쪽 `match address` ACL이 대칭이 아니거나(한쪽 ip/한쪽 gre), Tunnel/물리 적용 위치가 서로 다름.
- **해결:** 양쪽 모두 동일하게 `permit gre host ... host ...` + 물리 인터페이스 적용으로 통일. `show crypto engine connections active`의 encrypt/decrypt 증가 확인.

> 💡 **면접 한 줄 요약:** GRE-over-IPsec의 존재 이유는 "IPsec 단독으로는 암호화 못 하는 멀티캐스트 라우팅을 GRE로 감싸 암호화"하는 것이다. 실무 정석은 ① crypto map을 **물리 인터페이스**에 적용, ② 보호 대상을 **GRE 프로토콜 전체**로 지정, ③ **Transport 모드**로 헤더 오버헤드를 최소화하는 것이다.

---

> 관련: 5-3-1-1. 🚇 GRE Static · 5-3-1-2. 🔄 GRE Dynamic · 5-3-1-3. 🌐 mGRE & NHRP · 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE) · 5-3-3. 🛡️ CBAC (Stateful Firewall) · 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화) · 5-3-5. 🌐 DMVPN (동적 멀티포인트 VPN) · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동) · 5-3-7. 🧩 VPN 통합 정리 (GRE · IPsec · DMVPN) · 5-3-8. 🚑 VPN 트러블슈팅 치트시트 · 5-3-9. ⚡ VPN 명령어 퀵 레퍼런스