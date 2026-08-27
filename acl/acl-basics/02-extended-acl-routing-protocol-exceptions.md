# 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리

> **Tag:** #Network #Cisco #Routing #StaticRoute #CEF #ACL #Security #AccessControl
> **핵심 요약:** Extended ACL(100~199)의 5-Tuple 기반 정밀 필터링과
> RIP/EIGRP/OSPF 등 라우팅 프로토콜 트래픽 예외 처리 실무 표준

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Extended ACL을 출발지 인터페이스 in 방향에 적용해야 하는 이유

Extended ACL은 **출발지+목적지+프로토콜+포트**까지 세분화된 조건으로 트래픽을 정밀 식별할 수 있기 때문에, **출발지 라우터의 진입 인터페이스(in)에서 즉시 폐기**하는 것이 대역폭과 CPU를 아끼는 최적 지점이다.

어차피 차단할 트래픽을 백본 회선 끝까지 옮긴 뒤 목적지 근처에서 버리면 WAN 대역폭과 중간 라우터 CPU가 낭비된다. 다만 라우팅 프로토콜(RIP/EIGRP/OSPF)의 제어 패킷은 반드시 예외 permit을 걸어야 하며, 이를 놓치면 Neighbor가 끊어져 라우팅 테이블이 통째로 사라진다.

### `established` 옵션의 용도와 한계

내부→외부로 나간 TCP 세션의 **응답 트래픽만 허용**하고, 외부에서 임의로 시작하는 신규 TCP 세션은 차단하기 위해 사용한다. TCP 헤더의 `ACK` 또는 `RST` 비트가 세팅된 패킷만 통과시키는 옵션이다.

`established`는 TCP에만 적용 가능하며, UDP/ICMP에는 쓸 수 없다. UDP/ICMP까지 상태 기반으로 처리하려면 **Reflective ACL** 또는 **ZBF(Zone-Based Firewall)**를 사용해야 한다. 또한 진짜 상태 추적(stateful)이 아니라 TCP 플래그 검사만 하는 것이므로 정교한 스푸핑 공격에는 취약하다. 방화벽 대체품이 아님을 반드시 인지해야 한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Extended ACL 문법 요약

```cisco
! [문법 1] 프로토콜 + SA/DA만 지정
access-list <100-199> {permit|deny} <protocol> <SA> <SA-wildcard> <DA> <DA-wildcard>
!
! [문법 2] 포트 번호까지 지정 (TCP/UDP 전용)
access-list <100-199> {permit|deny} <tcp|udp> <SA> <SA-wildcard> [eq <src-port>] <DA> <DA-wildcard> [eq <dst-port>]
!
! [문법 3] host 키워드 (=단일 호스트, wildcard 0.0.0.0)
access-list 100 permit tcp host 3.3.3.3 host 1.1.1.1 eq 23
```
### 📌 프로토콜별 예외 처리 시그니처 (반드시 암기)

| 라우팅 프로토콜 | ACL 표기                             | 멀티캐스트 주소              | 비고         |
| -------- | ---------------------------------- | --------------------- | ---------- |
| RIPv2    | `permit udp any eq 520 any eq 520` | 224.0.0.9             | UDP 520    |
| EIGRP    | `permit eigrp any any`             | 224.0.0.10            | IP 프로토콜 88 |
| OSPF     | `permit ospf any any`              | 224.0.0.5 / 224.0.0.6 | IP 프로토콜 89 |
| BGP      | `permit tcp any any eq 179`        | (유니캐스트)               | TCP 179    |
### 📌 Named Extended ACL - 실무 표준 템플릿
```cisco
! [Named Extended ACL] 이름 기반 + 시퀀스 번호로 라인 편집 가능
!
ip access-list extended ACL_WAN_IN
 !
 ! (1) 라우팅 프로토콜 예외 - 반드시 최상단 permit (Neighbor 유지)
 permit ospf host 13.13.12.2 host 224.0.0.5
 !
 ! (2) 원하는 서비스 명시적 허용
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23    ! Telnet
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 80    ! HTTP
 permit icmp 13.13.30.0 0.0.0.255 host 1.1.1.1         ! ICMP
 !
 ! (3) 나머지 명시적 차단 (감사 로그 목적)
 deny ip any any log
!
interface Serial1/0
 ip access-group ACL_WAN_IN in
!
```
### 📌 라우팅 프로토콜별 예외 처리 상세

```cisco
! ===== RIPv2 사용 시 =====
ip access-list extended ACL_WITH_RIP
 permit udp host 13.13.12.2 eq 520 host 224.0.0.9 eq 520
! Neighbor의 소스 IP + 멀티캐스트 224.0.0.9 + UDP 520
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
!
!
! ===== EIGRP 사용 시 (Hello=멀티캐스트 / Update=유니캐스트) =====
ip access-list extended ACL_WITH_EIGRP
 permit eigrp host 13.13.12.2 host 224.0.0.10   ! Hello (멀티캐스트)
 permit eigrp host 13.13.12.2 host 13.13.12.1   ! Update/Query/Reply (유니캐스트)
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
!
!
! ===== OSPF 사용 시 =====
ip access-list extended ACL_WITH_OSPF
 permit ospf host 13.13.12.2 host 224.0.0.5     ! AllSPFRouters
 permit ospf host 13.13.12.2 host 224.0.0.6     ! DR/BDR (멀티엑세스 환경)
 permit ospf host 13.13.12.2 host 13.13.12.1    ! Virtual-link (필요 시)
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
!
```

### 📌 established 옵션 (준-상태 검사)

```cisco
! 시나리오: 내부→외부 TCP는 자유, 외부→내부는 응답만 허용
!
ip access-list extended ACL_OUTSIDE_IN
 ! 라우팅 프로토콜 예외 먼저
 permit ospf any any
 !
 ! 내부에서 나간 세션의 응답만 통과 (ACK/RST 비트 검사)
 permit tcp any 10.10.0.0 0.0.255.255 established
 !
 ! 나머지 외부발 신규 세션은 전부 차단
 deny ip any any log
!
interface Serial1/0
 ip access-group ACL_OUTSIDE_IN in
!
```
### 📌 IP Fragment 공격 방어

```cisco
! 시나리오: ICMP 분할 패킷을 이용한 DoS 공격 방어
!
ip access-list extended ACL_ANTI_FRAG
 deny icmp any host 13.13.4.4 fragments   ! 조각화된 ICMP만 차단
 permit ip any any
!
interface FastEthernet0/0
 ip access-group ACL_ANTI_FRAG in
!
```

### 📌 log-input 옵션 (침해 대응/포렌식)

```cisco
! deny 발생 시 어느 인터페이스로 들어왔는지까지 syslog에 기록
!
ip access-list extended ACL_AUDIT
 deny tcp host 150.1.13.4 13.13.12.0 0.0.0.255 eq 23 log-input
 deny tcp host 150.3.13.5 13.13.12.0 0.0.0.255 eq 80 log-input
 permit ip any any
!
! 출력 예시:
! %SEC-6-IPACCESSLOGP: list ACL_AUDIT denied tcp 150.1.13.4(44475)
!  (Serial1/0.123 ) -> 13.13.12.2(23), 1 packet
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어
```cisco
! ACL 룰과 라인별 hit count 확인 (매칭이 실제 발생하는지)
show access-lists
show ip access-lists ACL_WAN_IN

! 인터페이스 ACL 바인딩 확인
show ip interface Serial1/0 | include access list

! 라우팅 Neighbor 상태 (ACL로 끊어졌는지 최우선 검증)
show ip route
show ip eigrp neighbors
show ip ospf neighbor

! Syslog 확인 (log / log-input 결과)
show logging | include SEC-6-IPACCESSLOG
```

### 🚨 트러블슈팅 시나리오 1: ACL 적용 후 라우팅 테이블이 사라졌다

- **증상:** Serial in에 Extended ACL 적용 → `show ip route`에서 원격지 경로가 전부 사라짐 → Ping/서비스 전면 중단.
- **원인 파악:**
    1. `show ip ospf neighbor` (또는 `show ip eigrp neighbors`) → Neighbor가 Down/미형성 상태.
    2. `show access-lists` → 라우팅 프로토콜 permit 라인이 없거나, 암묵적 `deny any` 앞에 위치하지 않음.
- **해결법:**
    
    ```cisco
    ip access-list extended ACL_WAN_IN
     5 permit ospf host <neighbor-IP> host 224.0.0.5   ! 최상단에 삽입
    ```
    
    1. `clear ip ospf process` 또는 `clear ip eigrp neighbors` 로 이웃 재수립.
- **재발 방지:** ACL 설계 시 **라우팅 프로토콜 permit → 서비스 permit → deny any 순서**를 템플릿화.

### 🚨 트러블슈팅 시나리오 2: EIGRP는 되는데 특정 이웃만 안 붙는다

- **증상:** EIGRP Hello는 오는데 Neighbor 형성 후 즉시 Down 반복 (Stuck-in-Active).
- **원인 파악:**
    1. EIGRP는 Hello(멀티캐스트) + Update/Query/Reply(유니캐스트) 두 종류의 트래픽을 사용.
    2. ACL에 멀티캐스트(`224.0.0.10`)만 permit 하고 유니캐스트를 빠뜨렸을 가능성 높음.
- **해결법:**
    
    ```cisco
    ip access-list extended ACL_WAN_IN
     permit eigrp host 13.13.12.2 host 224.0.0.10   ! Hello
     permit eigrp host 13.13.12.2 host 13.13.12.1   ! Update/Query/Reply (★ 놓치기 쉬움)
    ```
    

### 🚨 트러블슈팅 시나리오 3: 특정 서비스만 지연/타임아웃 (Fragment 문제)

- **증상:** Ping은 되는데 대용량 데이터 전송(FTP, 대형 HTTP 응답)만 타임아웃.
- **원인 파악:**
    1. MTU 초과로 IP Fragment가 발생 → ACL이 후속 조각(첫 조각 이후)을 L4 정보 없이 판단.
    2. Extended ACL은 첫 조각의 L4 헤더만 검사하고 이후 조각은 프로토콜 수준으로만 매칭 → 의도치 않은 차단 발생.
- **해결법:**
    - 필요 시 `permit tcp any any` 같은 넓은 룰을 두거나, MTU/MSS 조정(`ip tcp adjust-mss 1360`)으로 분할 자체를 회피.
    - Anti-fragment ACL은 반드시 `fragments` 키워드로 명확히 구분.

### ⚠️ Extended ACL 실무 체크리스트

- [ ]  **출발지 인터페이스 in 방향**에 적용했는가?
- [ ]  라우팅 프로토콜 permit이 **ACL 최상단**에 위치하는가?
- [ ]  EIGRP는 **멀티캐스트 + 유니캐스트** 모두 permit 했는가?
- [ ]  OSPF는 멀티엑세스 환경일 경우 **224.0.0.5 + 224.0.0.6** 둘 다 열었는가?
- [ ]  `deny ip any any log` 를 명시해 차단 로그를 확보했는가?
- [ ]  상태 기반 필터링이 필요하면 `established` 대신 **Reflective ACL** 또는 **ZBF** 를 검토했는가?

---

> 관련: 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-3. ⚙️  고급 ACL - Reflective & Dynamic & Time-range · 3-1-4. 🧩 ACL 통합 정리 · 3-1-5. 🚑 ACL 트러블슈팅 치트시트 · 3-1-6. ⚡ ACL 명령어 퀵 레퍼런스