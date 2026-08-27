# 🧩 통합 실습 부가기술 (DHCP / NTP / Anti-Fragment ACL)

> **Tag:** #network #dhcp #ntp #acl #l3switch #cisco
> **핵심 요약:** L3 스위치 통합 구성 시 함께 쓰이는 서비스 기술 (DHCP 서버, NTP 시간 동기화, 조각 공격 차단)

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Q. L3 스위치를 DHCP 서버로 쓸 때 반드시 챙겨야 할 것은?

- 👉 **핵심 답변:** 게이트웨이로 쓰는 **SVI(또는 라우티드 IP)를 반드시 `excluded-address`로 제외**해야 합니다. 이걸 빼먹으면 DHCP가 게이트웨이 IP를 단말에 임대해버려 IP 충돌이 발생합니다. 또한 해당 VLAN의 SVI가 up 상태여야 DHCP 풀이 정상 동작합니다.
- **꼬리 질문 대비 설명:**
  - DHCP 풀에는 network/default-router/dns-server/lease를 지정. `default-router`는 그 VLAN의 게이트웨이(SVI IP)와 일치해야 함.
  - 단말이 다른 서브넷의 DHCP 서버를 써야 하면 라우터에 `ip helper-address`(DHCP Relay)를 설정.

### Q. NTP에서 Stratum과 인증은 왜 중요한가요?

- 👉 **핵심 답변:** **Stratum은 시간 소스의 신뢰 계층**으로, 서버가 stratum 2면 그 클라이언트는 stratum 3이 됩니다(숫자가 낮을수록 정확). **인증(authentication)** 은 지정된 신뢰 키를 가진 정당한 NTP 서버로부터만 시간을 받게 해, 악의적 시간 변조(로그 조작·인증서 무효화 공격)를 방지합니다.
- **꼬리 질문 대비 설명:**
  - 시간 동기화는 로그 상관분석, 인증서 검증, Kerberos 등에서 필수. 시간이 틀어지면 보안 체계 전체가 흔들림.
  - `ntp master N`은 자신을 stratum N의 기준 시계로 만듦.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 L3 스위치 DHCP 서버

```cisco
SW1(config)# ip routing
!
! === 게이트웨이 IP는 임대 대상에서 제외 (필수!) ===
SW1(config)# ip dhcp excluded-address 13.13.100.254
!
SW1(config)# ip dhcp pool VLAN100
SW1(dhcp-config)# network 13.13.100.0 255.255.255.0      ! 임대할 대역
SW1(dhcp-config)# default-router 13.13.100.254            ! 게이트웨이(SVI IP)
SW1(dhcp-config)# dns-server 168.126.63.1                 ! DNS 서버
SW1(dhcp-config)# lease 5 0 0                             ! 임대기간 5일 (일 시 분)
```

### 📌 NTP 인증 기반 시간 동기화

```cisco
! === NTP Server (R2) ===
R2(config)# clock set 15:07:30 11 jun 2026               ! 먼저 정확한 시간 수동 설정
R2(config)# ntp authenticate                             ! 인증 활성화
R2(config)# ntp authentication-key 1 md5 cisco1234       ! 인증 키 정의
R2(config)# ntp trusted-key 1                            ! 신뢰 키 지정
R2(config)# ntp master 2                                 ! stratum 2 기준 시계로 동작

! === NTP Client (SW1/R1 등) ===
Client(config)# ntp authenticate
Client(config)# ntp authentication-key 1 md5 cisco1234   ! 서버와 동일 키
Client(config)# ntp trusted-key 1
Client(config)# ntp server 121.160.20.2 key 1            ! 지정 서버로부터만 key1로 동기화
```

### 📌 IP Fragment 기반 DoS 차단 ACL

```cisco
! 특정 서버로 향하는 조각화(fragment) 패킷을 차단하여 조각 재조합 DoS 방어
R1(config)# ip access-list extended ANTI-FRAG
R1(config-ext-nacl)# deny ip any host 100.10.1.1 fragments   ! 조각 패킷만 차단
R1(config-ext-nacl)# permit ip any any                       ! 정상 트래픽 허용
!
R1(config)# interface fastethernet 0/0
R1(config-if)# ip access-group ANTI-FRAG in                  ! 인바운드 적용
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
! --- DHCP ---
SW1# show ip dhcp binding              ! 실제 임대된 IP-MAC 매핑 확인
SW1# show ip dhcp pool                 ! 풀 사용률/남은 주소 확인
SW1# show ip dhcp conflict             ! IP 충돌 기록

! --- NTP ---
Client# show ntp status                ! 동기화 여부 + 자신의 stratum 확인
Client# show ntp associations          ! NTP 서버 연결 상태 (* = 동기화됨)

! --- ACL ---
R1# show access-lists ANTI-FRAG        ! ACL 매칭 카운터 확인
R1# show ip interface fa0/0            ! ACL 적용 여부 확인
```

### 🚨 장애 시나리오 1: 단말이 DHCP로 IP를 못 받음

- **원인 파악:**
    1. 해당 VLAN **SVI가 down** — VLAN에 활성 Access 포트가 없으면 SVI가 안 뜨고 풀도 죽음.
    2. `default-router`를 게이트웨이와 다르게 설정 — 단말이 IP는 받아도 외부 통신 불가.
    3. 단말이 다른 서브넷에 있음 — 라우터에 `ip helper-address` 누락.
- **해결:** SVI up 확인 → 풀의 default-router가 SVI IP와 일치하는지 확인 → `show ip dhcp binding`으로 임대 확인.

### 🚨 장애 시나리오 2: NTP 클라이언트가 동기화 안 됨

- **증상:** `show ntp status`에 `unsynchronized` 표시.
- **원인 파악:**
    1. **인증 키 불일치** — 서버/클라이언트의 `authentication-key`가 다름.
    2. **라우팅 미비** — 클라이언트가 NTP 서버 IP로 도달 불가.
    3. 동기화에는 수 분이 걸릴 수 있음 (즉시 반영되지 않음).
- **해결:** 양쪽 키 일치 확인 → 서버로 Ping 확인 → `show ntp associations`에서 `*` 표시(동기화) 확인 후 stratum 값 검증.

> 💡 **면접 한 줄 요약:** L3 스위치 통합 구성에서 DHCP는 게이트웨이 제외가 핵심, NTP는 인증으로 신뢰 소스만 허용하는 것이 핵심, Fragment ACL은 `fragments` 키워드로 재조합 DoS를 초기 차단하는 것이 핵심이다.


---

]