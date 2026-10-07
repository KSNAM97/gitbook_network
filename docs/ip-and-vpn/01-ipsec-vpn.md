# IPsec VPN (IP Security)

> **Tag:** #network #vpn #ipsec #ike #esp #crypto #cisco
> **핵심 요약:** 공중망에서 트래픽을 암호화·인증하는 표준 보안 프로토콜. IKE 2단계 협상과 AH/ESP가 핵심

---

## 1. 핵심 기술 개념 (Concept)

### IPsec이 보장하는 것과 GRE만으로 부족한 이유

GRE는 캡슐화만 할 뿐 암호화가 없어 중간에서 스니핑하면 내용이 그대로 보인다. IPsec은 기밀성(암호화)·무결성(변조 방지)·인증(출처 확인)을 제공해 공중망을 지나는 트래픽을 실제로 보호한다.

Transport mode는 원본 IP 헤더를 유지하고 그 뒤 데이터만 보호한다(호스트 간). Tunnel mode는 원본 패킷 전체를 감싸고 New IP 헤더를 붙인다(Site-to-Site, 가장 일반적). 암호화 알고리즘은 대칭키(DES/3DES/AES, 빠름·데이터 암호화용), 비대칭키(RSA/DH, 키 교환·서명용), 해시(MD5/SHA, 무결성)로 나뉜다.

### AH와 ESP의 차이와 실무에서 ESP를 쓰는 이유

AH(프로토콜 51)는 인증/무결성만 제공하고 암호화가 없으며, IP 헤더 전체를 인증하므로 NAT 환경에서 못 쓴다. ESP(프로토콜 50)는 암호화+인증을 모두 제공하고 NAT와도 호환되어, 실무에서는 거의 ESP만 사용한다.

AH는 IP 헤더를 인증에 포함하므로 NAT가 출발지 IP를 바꾸면 무결성 검증이 깨진다. ESP는 페이로드만 보호하고 인증 데이터(ICV)로 무결성을 확인한다.

### IKE의 Phase 1과 Phase 2의 역할

Phase 1(IKE SA)은 두 피어 간 인증하고 안전한 관리 채널(ISAKMP SA)을 만든다. Phase 2(IPsec SA)는 그 안전한 채널 위에서 실제 데이터를 보호할 SA를 협상한다. 즉 Phase 1은 "안전한 협상 통로 확보", Phase 2는 "실제 데이터 암호화 규칙 합의"다.

Phase 1은 Main mode(6메시지, 안전) 또는 Aggressive mode(3메시지, 빠름·덜 안전)를 쓰며, Phase 2는 Quick mode(3메시지)를 쓴다. IKE는 UDP 500을 사용하므로 방화벽에서 UDP 500 허용이 필수다. 키가 오래 쓰이면 암호분석에 취약하므로 주기적으로 SA를 갱신한다(lifetime).

### IKE 협상 요약

| 구분 | 목적 | 모드 | 결과물 |
|------|------|------|--------|
| **Phase 1** | 피어 인증 + 안전한 채널 | Main / Aggressive | ISAKMP SA |
| **Phase 2** | 데이터 보호 규칙 협상 | Quick | IPsec SA |

---

## 2. 표준 설정 템플릿 (Configuration)

### IPsec Site-to-Site (crypto map 방식) - 5단계

```cisco
! === GIT-A (R1) ===
! [1단계] Phase 1 정책 (ISAKMP)
crypto isakmp policy 10
 authentication pre-share            ! 사전공유키 방식
 encryption 3des                     ! 암호화 알고리즘
 hash md5                            ! 해시(무결성)
 group 2                             ! DH 그룹(키 교환)
 lifetime 3600                       ! Phase1 SA 수명
!
! [2단계] 사전공유키 (상대 피어 IP와 매칭)
crypto isakmp key 0 solcisco address 121.160.20.2
!
! [3단계] Phase 2 변환 집합 (Transform-set)
crypto ipsec transform-set GIT_A esp-aes esp-sha-hmac   ! ESP+AES+SHA
!
! [4단계] Crypto Map (피어 + 변환집합 + 보호대상 결합)
crypto map IPSEC_GIT_A 10 ipsec-isakmp
 set peer 121.160.20.2               ! 상대 피어
 set transform-set GIT_A
 match address 100                   ! 보호할 트래픽(ACL)
!
! 보호 대상 정의 (양쪽 사설망)
access-list 100 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
! [5단계] 물리 인터페이스에 crypto map 적용
interface serial 1/0.10
 crypto map IPSEC_GIT_A
```

```cisco
! === GIT-B (R2) : ACL 방향 반대, peer 반대 ===
access-list 100 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
crypto isakmp policy 10
 authentication pre-share
 encryption 3des
 hash md5
 group 2
 lifetime 3600
crypto isakmp key 0 solcisco address 121.160.10.1
crypto ipsec transform-set GIT_B esp-aes esp-sha-hmac
crypto map IPSEC_GIT_B 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set GIT_B
 match address 100
interface serial 1/0.20
 crypto map IPSEC_GIT_B
```

### NAT와 IPsec 공존 시 (NAT 예외 처리 - 중요)

```cisco
! IPsec 보호 대상은 NAT에서 제외해야 함 (deny로 먼저 걸러냄)
GIT-A(config)# access-list 110 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255  ! VPN 트래픽 NAT 제외
GIT-A(config)# access-list 110 permit ip 192.168.10.0 0.0.0.255 any                    ! 나머지는 인터넷용 NAT
GIT-A(config)# ip nat inside source list 110 pool GIT_A overload
```

> 💡 이 순서가 틀려 VPN 트래픽이 NAT로 변환되면, 상대가 사설 IP를 기대하는데 공인 IP로 와서 **VPN이 깨집니다.** NAT 예외는 IPsec 실무의 필수 함정.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Router# show crypto isakmp sa            ! Phase1 SA 상태 (QM_IDLE=정상)
Router# show crypto isakmp policy        ! Phase1 정책 확인
Router# show crypto ipsec sa             ! Phase2 SA + 암/복호화 카운터
Router# show crypto ipsec transform-set  ! Phase2 변환집합 확인
Router# show crypto engine connections active   ! 실제 암/복호화 발생 확인
Router# show crypto isakmp peer          ! 연결된 피어 확인
Router# debug crypto isakmp              ! Phase1 협상 실시간 추적
Router# debug crypto ipsec               ! Phase2 협상 추적
```

### `show crypto isakmp sa` 판독

- **QM_IDLE** : Phase 1 정상 완료 (Quick Mode 대기 상태)
- **MM_NO_STATE / MM_KEY_EXCH** : Phase 1 협상 진행/실패 중

### Case 1: Phase 1이 안 올라옴 (SA 없음)

- **원인 파악:**
    1. **정책 불일치** — 양쪽 encryption/hash/group/authentication이 완전히 같아야 함.
    2. **사전공유키(PSK) 불일치** 또는 peer 주소 오지정.
    3. **UDP 500 차단** — 경로상 ACL/방화벽이 IKE를 막음.
- **해결:** `debug crypto isakmp`로 어느 파라미터에서 실패하는지 확인, 양쪽 정책·키·peer를 일치시킴.

### Case 2: Phase 1은 되는데 Phase 2가 안 됨 / 트래픽 암호화 안 됨

- **원인 파악:**
    1. **Interesting Traffic ACL 불일치** — 양쪽 match ACL이 서로 대칭(mirror)이어야 함.
    2. **transform-set 불일치** (ESP 알고리즘 다름).
    3. **crypto map을 인터페이스에 미적용.**
- **해결:** 양쪽 ACL을 정확히 반대로 맞추고, `show crypto ipsec sa`의 encrypt/decrypt 카운터가 증가하는지 확인.

### Case 3: VPN은 되는데 인터넷이 안 되거나, VPN이 간헐 끊김

- **원인 파악:** **NAT 예외 처리 누락.** VPN 대상 트래픽이 NAT로 변환되어 상대가 인식 못 함.
- **해결:** NAT ACL에서 VPN 트래픽을 `deny`로 먼저 제외.

---

]
