# Cisco IOS ACL 통합 정리 — Standard·Extended·Stateful 한눈에

> **Tag:** #Network #ACL #StandardACL #ExtendedACL #ReflectiveACL #DynamicACL #TimeRange #Security
> **핵심 요약:** Cisco IOS ACL은 패킷을 **위에서 아래로 순차 검사하고, 최초 일치 시 즉시 종료하며, 마지막에는 암묵적 deny가 존재**한다. Standard ACL은 출발지만 보므로 목적지 가까이, Extended ACL은 출발지·목적지·프로토콜·포트를 구분하므로 출발지 가까이 배치하는 것이 기본이다. 운영망에서는 Named ACL과 시퀀스 번호를 사용하고, 상태 기반 제어가 필요하면 Reflective ACL보다 ZBF 또는 전용 방화벽을 우선 검토한다.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. ACL을 관통하는 가장 중요한 동작 원리는?

👉 **핵심 답변:** ACL은 **위에서 아래로 순차 검사 → 최초 일치 항목 적용 → 검사 종료** 방식으로 동작합니다. 어떤 항목에도 일치하지 않으면 마지막의 암묵적 `deny`에 의해 차단됩니다.

```properties
[1] Top-down 순차 검사
[2] First Match 적용 후 즉시 종료
[3] 마지막에 Implicit Deny 존재
[4] ACL은 생성만 해서는 동작하지 않으며 적용 지점이 필요
```
- 구체적인 조건은 위에, 넓은 조건은 아래에 배치
- `permit any` 또는 `permit ip any any`를 너무 위에 두면 이후 항목은 동작하지 않음
- 암묵적 deny는 일반적으로 로그와 hit count를 남기지 않으므로 감사 목적이라면 명시적 deny 사용

```cisco
ip access-list extended ACL_EXAMPLE
 10 permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.10 eq 443
 20 deny   ip 10.10.10.0 0.0.0.255 any log
 30 permit ip any any
```

---

### Q2. Standard ACL과 Extended ACL의 결정적 차이는?

👉 **핵심 답변:** Standard ACL은 **출발지 IP만** 확인하고, Extended ACL은 출발지·목적지·프로토콜·포트 등으로 트래픽을 정밀하게 구분합니다.

|구분|Standard ACL|Extended ACL|
|---|---|---|
|번호 범위|1~99, 1300~1999|100~199, 2000~2699|
|검사 조건|출발지 IP|SA·DA·Protocol·Port|
|기본 배치|목적지 가까이|출발지 가까이|
|대표 용도|VTY, NAT 매칭, 단순 분류|서비스별 트래픽 필터링|
|권장 형식|Named Standard|Named Extended|

> Standard는 조건이 넓으므로 출발지 가까이 적용하면 해당 출발지의 정상 트래픽까지 과도하게 차단할 수 있습니다.

---

### Q3. ACL의 `in`과 `out`은 어떻게 판단하는가?

👉 **핵심 답변:** 방향은 네트워크 전체 기준이 아니라 **ACL이 적용된 인터페이스 기준**입니다.

```properties
in  = 패킷이 해당 인터페이스로 라우터에 들어오는 방향
out = 라우팅 결정 후 해당 인터페이스로 나가는 방향
```

```text
PC ── Gi0/0 [Router] Gi0/1 ── Server

PC → Server 트래픽 기준
Gi0/0 = in
Gi0/1 = out
```

- Standard ACL은 목적지 가까이 적용하는 것이 원칙
    - 목적지 측 인터페이스의 `out`인 경우가 많지만 토폴로지에 따라 `in`일 수도 있음
- Extended ACL은 출발지 가까이 적용하는 것이 원칙
    - 출발지 LAN 인터페이스의 `in`인 경우가 많지만 항상 그런 것은 아님

---

### Q4. Numbered ACL과 Named ACL 중 무엇을 사용해야 하는가?

👉 **핵심 답변:** 운영망에서는 목적을 식별하기 쉽고 시퀀스 기반 편집이 편리한 **Named ACL을 권장**합니다.

```cisco
ip access-list standard ACL_VTY_ADMIN
 10 permit 10.10.10.0 0.0.0.255
 20 deny any log
```

> 최신 IOS/IOS XE 중에는 Numbered ACL도 아래처럼 Named ACL 구성 모드로 진입하여 시퀀스 편집이 가능한 플랫폼이 있습니다.

```cisco
ip access-list standard 1
 15 permit 10.10.20.0 0.0.0.255
 no 20
```

하지만 다음과 같은 old-style 명령은 전체 ACL을 삭제할 수 있으므로 주의합니다.

```cisco
no access-list 1
no access-list 101
```

---

### Q5. `established`, Reflective ACL, ZBF의 차이는?

|기술|동작|지원 범위|한계|
|---|---|---|---|
|`established`|TCP ACK/RST 비트 검사|TCP|실제 세션 상태 추적 아님|
|Reflective ACL|임시 역방향 ACE 생성|TCP·UDP·ICMP 등|애플리케이션 인식 부족|
|ZBF/전용 방화벽|Stateful Inspection|다양한 프로토콜|설계 및 자원 필요|

```cisco
permit tcp any 10.10.0.0 0.0.255.255 established
```

- `established`는 ACK 또는 RST 플래그 검사
- 패킷 플래그 위조 가능성이 있어 방화벽 대체 불가
- Reflective ACL은 주소·프로토콜·포트 등을 뒤집어 임시 ACE 생성
- FTP처럼 세션 중 포트가 바뀌는 프로토콜은 Passive FTP 등이 필요할 수 있음

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. Standard ACL — 일부 출발지만 허용

```cisco
! 192.168.3.0/24 출발 트래픽 허용
access-list 1 permit 192.168.3.0 0.0.0.255
!
interface FastEthernet0/0
 ip access-group 1 out
!
! 목적지 측 인터페이스가 Fa0/0이고 해당 트래픽이 out으로 나가는
! 토폴로지라는 전제에서 사용하는 구성
```

---

### 2-2. Standard ACL — 일부 출발지만 차단

```cisco
access-list 2 deny 192.168.1.0 0.0.0.255
access-list 2 permit any
!
interface FastEthernet0/0
 ip access-group 2 out
```

> `permit any`가 없으면 암묵적 deny에 의해 나머지 출발지도 모두 차단됩니다.

---

### 2-3. Wildcard Mask로 연속 대역 집약

```cisco
! 172.16.0.0/24 ~ 172.16.7.0/24 차단
access-list 3 deny 172.16.0.0 0.0.7.255
access-list 3 permit any
!
interface FastEthernet0/0
 ip access-group 3 out
```

```properties
Wildcard 0 = 해당 비트를 반드시 비교
Wildcard 1 = 해당 비트를 무시
0.0.7.255 = 172.16.0.0/21 범위 매칭
```

> 집약 전에 시작 주소가 블록 경계에 맞는지 확인해야 합니다.

---

### 2-4. Named Standard ACL — 실무 권장

```cisco
ip access-list standard ACL_ALLOW_MGMT
 10 permit 10.10.10.0 0.0.0.255
 20 permit 10.10.20.0 0.0.0.255
 30 deny any log
!
interface FastEthernet0/0
 ip access-group ACL_ALLOW_MGMT out
```

특정 위치에 항목 추가:

```cisco
ip access-list standard ACL_ALLOW_MGMT
 25 permit 10.10.30.0 0.0.0.255
```

특정 항목 삭제:

```cisco
ip access-list standard ACL_ALLOW_MGMT
 no 25
```

시퀀스 번호 재정렬:

```cisco
ip access-list resequence ACL_ALLOW_MGMT 10 10
```

---

### 2-5. VTY 접근 제어

```cisco
ip access-list standard ACL_VTY_ADMIN
 10 permit 10.10.10.0 0.0.0.255
 20 deny any log
!
line vty 0 4
 access-class ACL_VTY_ADMIN in
 transport input ssh
 login local
```

> 인터페이스 패킷 필터는 `ip access-group`, VTY 접근 제어는 `access-class`를 사용합니다.

---

### 2-6. Named Extended ACL — 기본 서비스 제어

```cisco
ip access-list extended ACL_WAN_IN
 !
 ! 라우팅 프로토콜 예외가 실제로 필요한 인터페이스/방향일 때만 구성
 permit ospf host 13.13.12.2 any
 !
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 80
 permit icmp 13.13.30.0 0.0.0.255 host 1.1.1.1
 !
 deny ip any any log
!
interface Serial1/0
 ip access-group ACL_WAN_IN in
```

> `permit ospf host <neighbor> any`는 OSPF의 멀티캐스트와 유니캐스트 패킷을 함께 고려한 보수적인 예시입니다. 실제 환경에서는 인터페이스 유형과 Neighbor 주소에 맞춰 최소 권한으로 제한합니다.

---

### 2-7. 라우팅 프로토콜 예외 처리

|프로토콜|IP Protocol/Port|주요 목적지|
|---|---|---|
|RIPv2|UDP 520|224.0.0.9|
|EIGRP|IP Protocol 88|224.0.0.10 또는 Unicast|
|OSPF|IP Protocol 89|224.0.0.5/6 또는 Unicast|
|BGP|TCP 179|Peer Unicast IP|

#### RIPv2

```cisco
ip access-list extended ACL_WITH_RIP
 permit udp host 13.13.12.2 eq 520 host 224.0.0.9 eq 520
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
```

#### EIGRP

```cisco
ip access-list extended ACL_WITH_EIGRP
 permit eigrp host 13.13.12.2 host 224.0.0.10
 permit eigrp host 13.13.12.2 host 13.13.12.1
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
```

#### OSPF

```cisco
ip access-list extended ACL_WITH_OSPF
 permit ospf host 13.13.12.2 host 224.0.0.5
 permit ospf host 13.13.12.2 host 224.0.0.6
 permit ospf host 13.13.12.2 host 13.13.12.1
 permit tcp 13.13.30.0 0.0.0.255 host 1.1.1.1 eq 23
```

> OSPF 유니캐스트 패킷은 Virtual Link에만 한정되지 않습니다. 인터페이스 유형과 동작 과정에 따라 유니캐스트가 사용될 수 있으므로 Neighbor 패킷 전체를 고려해야 합니다.

---

### 2-8. TCP `established`

```cisco
ip access-list extended ACL_OUTSIDE_IN
 permit ospf any any
 permit tcp any 10.10.0.0 0.0.255.255 established
 deny ip any any log
!
interface Serial1/0
 ip access-group ACL_OUTSIDE_IN in
```

> ACK/RST 비트만 확인하므로 Stateful Firewall로 간주하면 안 됩니다.

---

### 2-9. Fragment 제어

```cisco
ip access-list extended ACL_ANTI_FRAG
 deny icmp any host 13.13.4.4 fragments
 permit ip any any
!
interface FastEthernet0/0
 ip access-group ACL_ANTI_FRAG in
```

> `fragments`는 일반적으로 L4 헤더가 없는 **비초기 조각(non-initial fragment)**을 대상으로 합니다. PMTUD에 필요한 ICMP까지 과도하게 차단하면 대용량 전송 장애가 발생할 수 있습니다.

TCP MSS 조정 예시:

```cisco
interface Tunnel0
 ip tcp adjust-mss 1360
```

> `ip tcp adjust-mss`는 인터페이스를 통과하는 TCP SYN의 MSS를 조정하는 기능입니다. 실제 터널 오버헤드와 MTU를 계산해 값을 결정해야 합니다.

---

### 2-10. `log-input`

```cisco
ip access-list extended ACL_AUDIT
 deny tcp host 150.1.13.4 13.13.12.0 0.0.0.255 eq 23 log-input
 deny tcp host 150.3.13.5 13.13.12.0 0.0.0.255 eq 80 log-input
 permit ip any any
```

- `log`: 매칭 패킷의 기본 정보 기록
- `log-input`: 유입 인터페이스와 가능한 경우 Source MAC 정보까지 기록
- 대량 트래픽에 무분별하게 사용하면 CPU와 Syslog 부하가 증가할 수 있음

---

### 2-11. Reflective ACL

```cisco
ip access-list extended IN_TO_OUT
 permit tcp any any reflect REFLECTIVE_SESSIONS
 permit udp any any reflect REFLECTIVE_SESSIONS
 permit icmp any any reflect REFLECTIVE_SESSIONS
!
ip access-list extended OUT_TO_IN
 permit ospf any any
 evaluate REFLECTIVE_SESSIONS
 deny ip any any log
!
interface Serial1/0
 ip access-group IN_TO_OUT out
 ip access-group OUT_TO_IN in
!
ip reflexive-list timeout 120
```

```properties
reflect  = 내부에서 시작한 세션을 보고 임시 ACE 생성
evaluate = 반대 방향 패킷을 임시 ACE와 비교
```

- 외부 인터페이스에 구성하는 예시
- `evaluate`는 `deny ip any any`보다 위에 있어야 함
- Reflective ACL 자체는 인터페이스에 직접 적용하지 않고 Named Extended ACL 안에 중첩
- 기본 global timeout은 300초

---

### 2-12. Dynamic ACL — Lock-and-Key

> 신규 운영망에서는 권장하지 않는 레거시 방식입니다. Telnet 인증 의존성과 IP Spoofing 위험이 있으므로 VPN, AAA, ZBF 또는 전용 방화벽을 우선 검토합니다.

```cisco
username soladmin secret <STRONG_SECRET>
!
access-list 105 permit tcp any host 13.13.3.3 eq 23
access-list 105 permit ospf any any
access-list 105 dynamic SOL_DYNAMIC timeout 5 permit ip any 13.13.15.0 0.0.0.255
!
interface Serial1/0
 ip access-group 105 in
!
line vty 0 4
 login local
 transport input telnet
 autocommand access-enable host timeout 5
```

> 원본의 비밀번호 평문은 `secret` 및 플레이스홀더로 변경했습니다. Dynamic ACL의 Trigger는 전통적으로 Telnet을 사용하며, 모든 VTY 라인의 설정을 일관되게 구성해야 합니다.

---

### 2-13. Time-range

```cisco
time-range BIZ_HOURS
 periodic weekdays 08:00 to 18:00
 periodic weekend 10:00 to 16:00
!
time-range PROJECT_WINDOW
 absolute start 09:30 01 July 2026 end 18:30 29 August 2026
!
ip access-list extended ACL_TIME_BASED
 permit ospf any any
 permit tcp any host 13.13.4.4 eq 23 time-range BIZ_HOURS
 permit tcp any host 13.13.4.4 eq 80 time-range BIZ_HOURS
 permit icmp any host 13.13.4.4 time-range PROJECT_WINDOW
 deny ip any any log
!
interface FastEthernet0/0
 ip access-group ACL_TIME_BASED in
```

NTP 및 시간대 설정:

```cisco
clock timezone KST 9
!
ntp server 203.248.240.140
ntp server 210.98.16.100
```

> 실제 운영에서는 조직에서 관리하는 NTP 서버 또는 검증된 NTP 서버 주소를 사용합니다.

---

## 3. 검증 및 트러블슈팅 (Verification)

### 3-1. 필수 검증 명령어

```cisco
show access-lists
show access-lists 1
show ip access-lists ACL_WAN_IN
show ip interface FastEthernet0/0
show running-config | section ip access-list
show running-config | include access-group
show running-config | section line vty
```

### 3-2. 라우팅 프로토콜 확인

```cisco
show ip route
show ip protocols
show ip eigrp neighbors
show ip ospf neighbor
```

### 3-3. 고급 ACL 확인

```cisco
show ip access-lists REFLECTIVE_SESSIONS
show ip access-lists 105
show time-range
show clock detail
show ntp status
show ntp associations
show logging | include SEC-6-IPACCESSLOG
```

---

## 4. 실무 설계 체크리스트

- [ ]  ACL의 목적과 적용 트래픽 방향을 먼저 도식화했는가?
- [ ]  구체적인 ACE를 위에, 넓은 ACE를 아래에 배치했는가?
- [ ]  암묵적 deny에 의해 정상 트래픽이 차단되지 않는가?
- [ ]  Standard ACL을 목적지 가까이 배치했는가?
- [ ]  Extended ACL을 출발지 가까이 배치했는가?
- [ ]  라우팅 프로토콜·DHCP·DNS·NTP 등 제어 트래픽을 확인했는가?
- [ ]  Named ACL과 시퀀스 번호를 사용했는가?
- [ ]  변경 전 기존 ACL과 hit count를 백업했는가?
- [ ]  `log` 사용에 따른 CPU/Syslog 부하를 검토했는가?
- [ ]  상태 기반 제어가 필요하면 ZBF 또는 전용 방화벽을 검토했는가?
- [ ]  Time-range 사용 전 NTP와 timezone을 검증했는가?

---

> 📌 **핵심 요약**
> 
> - ACL은 **Top-down, First Match, Implicit Deny**
> - Standard는 출발지만 검사 → **목적지 가까이**
> - Extended는 SA·DA·Protocol·Port 검사 → **출발지 가까이**
> - 방향은 인터페이스 기준의 `in/out`
> - 운영망은 Named ACL + Sequence Number 권장
> - `established`는 TCP Flag 검사일 뿐
> - Reflective/Dynamic ACL은 레거시 또는 제한적 세션 필터 → 신규 환경은 ZBF/전용 방화벽 검토
> - 관련: 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리 · 3-1-3. ⚙️  고급 ACL - Reflective & Dynamic & Time-range · 1. 🧮 IPv4 Addressing & Subnetting & VLSM & Summary & Wildcard Mask
