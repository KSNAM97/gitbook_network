# 💾 DHCP (Dynamic Host Configuration Protocol)

> **Tag:** #network #dhcp #ip-service #dora #relay #cisco
> **핵심 요약:** IP·Subnet·Gateway·DNS를 단말에 자동 할당하는 프로토콜. DORA 4단계와 excluded-address 설계가 핵심

> 🔗 연계 문서: 05-02_통합실습_부가기술 (L3 스위치 DHCP 서버 설정)

---

## 1. 🎯 핵심 기술 개념 (Concept)

### DHCP DORA 4단계와 Broadcast를 쓰는 이유

단말은 부팅 시 자기 IP가 없어(0.0.0.0) 유니캐스트가 불가능하므로 Broadcast로 서버를 찾는다. 순서는 Discover(단말→전체, 서버 탐색) → Offer(서버→단말, IP 제안) → Request(단말→전체, 특정 서버 선택 공표) → ACK(서버→단말, 최종 확정)이다.

Discover와 Request가 모두 Broadcast인 이유는, Request가 여러 서버 중 어느 서버 것을 택했는지 나머지 서버에게도 알려 나머지 Offer를 회수시키기 위함이다. Offer 전 서버는 ICMP Ping / ARP로 해당 IP가 이미 쓰이는지(중복) 확인 후 제안하며, DHCP는 UDP 67(서버) / 68(클라이언트) 포트를 사용한다.

### DHCP 임대(Lease) 갱신 시점 (T1/T2 타이머)

클라이언트는 임대 시간의 50%(T1) 시점에 할당해준 서버로 유니캐스트 갱신을 시도하고, 실패하면 87.5%(T2) 시점에 아무 서버에게나 Broadcast로 갱신을 시도한다. 그래도 실패해 임대가 만료되면 IP를 반납하고 DORA를 처음부터 다시 시작한다.

이 단계적 갱신 덕분에 원 서버가 잠깐 죽어도 백업 서버가 있으면 임대를 이어갈 수 있다. `lease infinite`(무제한)는 프린터·서버처럼 IP가 고정돼야 하는 장비에, 짧은 임대는 게스트망처럼 회전율 높은 곳에 적합하다.

### 단말과 DHCP 서버가 다른 서브넷에 있을 때의 동작

DHCP Discover는 Broadcast라 라우터를 넘지 못한다. 따라서 단말 쪽 게이트웨이(라우터/L3 스위치)에 `ip helper-address`(DHCP Relay)를 설정해, Broadcast를 원격 DHCP 서버로 유니캐스트 변환·전달해야 한다.

Relay가 없으면 서브넷마다 DHCP 서버를 둬야 하지만, Relay를 쓰면 중앙 집중형 DHCP 서버 1대로 전 서브넷을 커버할 수 있다.

### 📊 DORA 4단계 요약

| 단계 | 방향 | 출발지 IP | 방식 | 역할 |
|------|------|-----------|------|------|
| **D**iscover | Client→ | 0.0.0.0 | Broadcast | 서버 탐색 |
| **O**ffer | Server→ | 서버IP | (u/b)cast | IP 제안 (중복검사 후) |
| **R**equest | Client→ | 0.0.0.0 | Broadcast | 특정 서버 선택 공표 |
| **A**CK | Server→ | 서버IP | (u/b)cast | 최종 확정 |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 DHCP 서버 기본 구성

```cisco
! === 서버 인터페이스(게이트웨이 역할) ===
R1(config)# interface fastethernet 0/1
R1(config-if)# no shutdown
R1(config-if)# ip address 192.168.10.254 255.255.255.0
!
! === 임대에서 제외할 주소 (게이트웨이/서버/고정IP 반드시 제외!) ===
R1(config)# ip dhcp excluded-address 192.168.10.250 192.168.10.254   ! 범위 제외
!
! === DHCP 풀 정의 ===
R1(config)# ip dhcp pool CCNA
R1(dhcp-config)# network 192.168.10.0 255.255.255.0    ! 할당할 대역
R1(dhcp-config)# default-router 192.168.10.254          ! 게이트웨이 (SVI/라우터 IP와 일치)
R1(dhcp-config)# dns-server 192.168.10.252 192.168.10.253  ! DNS (Primary, Secondary)
R1(dhcp-config)# lease 10                               ! 임대기간 (일) / infinite=무제한

### 📌 excluded-address 설계 (실무 핵심)

```cisco
! 단일 주소 제외
R1(config)# ip dhcp excluded-address 192.168.1.254        ! 게이트웨이 1개
R1(config)# ip dhcp excluded-address 192.168.1.200        ! 서버 자신
!
! 비연속 용도 주소 대역 제외
R1(config)# ip dhcp excluded-address 192.168.1.221 192.168.1.224   ! 타 용도 예약 대역
!
! === 임대기간 세부 지정: lease [Day] [Hours] [Minutes] ===
R1(dhcp-config)# lease infinite    ! 무제한 (프린터/서버용)
R1(dhcp-config)# lease 0 12 0      ! 12시간 (게스트망용)
```

### 📌 DHCP 클라이언트 (라우터를 PC로 흉내)

```cisco
PC1(config)# interface fastethernet 0/1
PC1(config-if)# no shutdown
PC1(config-if)# ip address dhcp                          ! DHCP로 IP 자동 획득
```

### 📌 DHCP Relay (서버가 다른 서브넷일 때)

```cisco
! 단말 쪽 게이트웨이에 설정 → Broadcast를 원격 서버로 전달
GW(config)# interface vlan 10
GW(config-if)# ip helper-address 192.168.100.1          ! 원격 DHCP 서버 IP
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
! --- 서버 측 ---
R1# show ip dhcp binding          ! 실제 임대된 IP ↔ MAC 매핑 (가장 중요)
R1# show ip dhcp pool             ! 풀 사용률/가용 주소 수
R1# show ip dhcp conflict         ! IP 충돌 기록 (Ping/ARP 검사 실패분)
R1# debug ip dhcp server events   ! 실시간 DORA 과정 추적

! --- 클라이언트 측 ---
PC1# show ip interface brief      ! 할당받은 IP 확인
PC1# show dhcp lease              ! 임대 상세 (IP/게이트웨이/DNS/T1·T2)
```

### 🚨 장애 시나리오 1: 단말이 IP를 못 받음 (169.254.x.x가 뜸)

- **증상:** 클라이언트가 APIPA 주소(169.254.x.x)를 받음 = DHCP 서버 응답 없음.
- **원인 파악 (순서대로):**
    1. **서버-단말 사이에 라우터가 있음** → Broadcast가 안 넘어감. `ip helper-address` 필요.
    2. 풀의 `network` 대역과 서버 인터페이스 대역이 불일치.
    3. 풀의 가용 주소 고갈 — `show ip dhcp pool`에서 사용률 확인.
- **해결:** 같은 서브넷이면 서버 도달성 확인, 다른 서브넷이면 Relay 설정, 고갈이면 대역 확장/임대 단축.

### 🚨 장애 시나리오 2: IP는 받았는데 인터넷/타 대역 통신 불가

- **원인 파악:** `default-router`(게이트웨이)를 잘못 지정했거나 `dns-server`가 틀림. 단말은 IP를 받아도 게이트웨이가 엉뚱하면 외부로 못 나감.
- **해결:** `show dhcp lease`로 단말이 받은 게이트웨이/DNS 확인 → 풀의 `default-router`가 실제 게이트웨이 IP와 일치하는지 점검.

### 🚨 장애 시나리오 3: IP 충돌 발생 (show ip dhcp conflict에 기록)

- **원인 파악:** **게이트웨이/서버/고정IP 장비 주소를 `excluded-address`로 제외하지 않아서** DHCP가 이미 쓰이는 IP를 임대함.
- **해결:** 충돌 IP를 `excluded-address`에 추가하고, `clear ip dhcp conflict *`로 기록 초기화 후 재임대.

> 💡 **면접 한 줄 요약:** DHCP의 핵심은 DORA(전부 도달 위해 Broadcast 활용)와 T1/T2 갱신 타이머이며, 실무 설계의 90%는 "게이트웨이·서버·고정장비를 excluded로 빼는 것"과 "서브넷이 다르면 ip helper-address로 Relay하는 것"으로 요약된다.

---

> ⚠️ **원본 실습 오류 정정:** 첫 번째 실습 preconfig의 `dns-server 192.168.10.252 192.168.10.254`는 조건(252, 253)과 다른 오타입니다. 정답은 `dns-server 192.168.10.252 192.168.10.253` 입니다. (하단 정상 설정본 기준)

---

> 관련: 5-1-1. 🧩 DHCP 통합 정리 · 5-1-2. 🚑 DHCP 트러블슈팅 치트시트 · 5-1-3. ⚡ DHCP 명령어 퀵 레퍼런스