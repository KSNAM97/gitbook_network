# Cisco IOS ACL 명령어 퀵 레퍼런스

> **Tag:** #Network #ACL #QuickReference #StandardACL #ExtendedACL #ReflectiveACL #TimeRange
> **핵심 요약:** Standard·Extended·Reflective·Dynamic·Time-range ACL의 설정과 검증 명령어를 빠르게 조회하기 위한 암기 카드. 이해보다는 조회와 복붙이 목적이다.

---

## 1. ACL 번호 범위

| 유형 | 기본 번호 | 확장 번호 |
|---|---:|---:|
| Standard IP ACL | 1~99 | 1300~1999 |
| Extended IP ACL | 100~199 | 2000~2699 |

> 운영망에서는 번호보다 목적을 알기 쉬운 Named ACL을 권장합니다.

---

## 2. ACL 동작 3대 원칙

```properties
1. 위에서 아래로 순차 검사
2. 최초 일치 항목 적용 후 종료
3. 마지막에 암묵적 deny 존재
```
```properties
Standard ACL = Source IP
Extended ACL = Source + Destination + Protocol + Port
```

---

## 3. Standard ACL

### Numbered Standard ACL

```cisco
access-list <1-99> permit <SOURCE> <WILDCARD>
access-list <1-99> deny   <SOURCE> <WILDCARD>
access-list <1-99> permit any
```

예시:

```cisco
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 2 deny 192.168.20.0 0.0.0.255
access-list 2 permit any
```

### Named Standard ACL

```cisco
ip access-list standard <ACL_NAME>
 <SEQ> permit <SOURCE> <WILDCARD>
 <SEQ> deny   <SOURCE> <WILDCARD> [log]
```

예시:

```cisco
ip access-list standard ACL_VTY_ADMIN
 10 permit 10.10.10.0 0.0.0.255
 20 deny any log
```

### 인터페이스 적용

```cisco
interface <INTERFACE>
 ip access-group <ACL_NAME_OR_NUMBER> {in|out}
```

### VTY 적용

```cisco
line vty 0 4
 access-class <STANDARD_ACL> in
 transport input ssh
```

---

## 4. Extended ACL

### 기본 문법

```cisco
access-list <100-199> {permit|deny} <PROTOCOL> \
 <SOURCE> <SOURCE_WILDCARD> \
 <DESTINATION> <DESTINATION_WILDCARD>
```

### TCP/UDP 포트 지정

```cisco
access-list <100-199> {permit|deny} {tcp|udp} \
 <SOURCE> <SOURCE_WILDCARD> [eq <SOURCE_PORT>] \
 <DESTINATION> <DESTINATION_WILDCARD> [eq <DEST_PORT>]
```

### Named Extended ACL

```cisco
ip access-list extended <ACL_NAME>
 <SEQ> permit tcp <SOURCE> <WC> <DESTINATION> <WC> eq <PORT>
 <SEQ> permit udp <SOURCE> <WC> <DESTINATION> <WC> eq <PORT>
 <SEQ> deny ip any any log
```

예시:

```cisco
ip access-list extended ACL_WEB
 10 permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.10 eq 443
 20 deny ip any any log
```

---

## 5. 주소 키워드

|표현|의미|
|---|---|
|`any`|모든 주소|
|`host 10.1.1.1`|단일 호스트|
|`10.1.1.1 0.0.0.0`|단일 호스트|
|`10.1.1.0 0.0.0.255`|10.1.1.0/24|
|`172.16.0.0 0.0.7.255`|172.16.0.0/21|

```properties
Wildcard 0 = 비교
Wildcard 1 = 무시
Wildcard = 255.255.255.255 - Subnet Mask
```

---

## 6. 포트 연산자

|연산자|의미|
|---|---|
|`eq`|같음|
|`neq`|같지 않음|
|`lt`|미만|
|`gt`|초과|
|`range`|범위|

```cisco
permit tcp any host 192.0.2.10 eq 443
permit udp any host 192.0.2.53 eq 53
permit tcp any any range 1024 65535
```

---

## 7. 라우팅 프로토콜 예외

### RIPv2

```cisco
permit udp host <NEIGHBOR_IP> eq 520 host 224.0.0.9 eq 520
```

### EIGRP

```cisco
permit eigrp host <NEIGHBOR_IP> host 224.0.0.10
permit eigrp host <NEIGHBOR_IP> host <LOCAL_IP>
```

보수적 예시:

```cisco
permit eigrp host <NEIGHBOR_IP> any
```

### OSPF

```cisco
permit ospf host <NEIGHBOR_IP> host 224.0.0.5
permit ospf host <NEIGHBOR_IP> host 224.0.0.6
permit ospf host <NEIGHBOR_IP> host <LOCAL_IP>
```

보수적 예시:

```cisco
permit ospf host <NEIGHBOR_IP> any
```

### BGP

```cisco
permit tcp host <PEER_IP> eq 179 host <LOCAL_IP>
permit tcp host <PEER_IP> host <LOCAL_IP> eq 179
```

> ACL 방향과 세션 시작 방향에 따라 Source/Destination Port 179 위치가 달라질 수 있습니다.

---

## 8. TCP Established

```cisco
ip access-list extended ACL_OUTSIDE_IN
 permit tcp any 10.10.0.0 0.0.255.255 established
 deny ip any any log
```

```properties
established = TCP ACK 또는 RST 플래그 검사
Stateful Firewall 아님
UDP/ICMP 적용 불가
```

---

## 9. Reflective ACL

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
reflect  = 임시 역방향 ACE 생성
evaluate = 임시 ACE 평가
Default Global Timeout = 300초
```

---

## 10. Dynamic ACL — Lock-and-Key

```cisco
username <USER> secret <SECRET>
!
access-list 105 permit tcp any host <ROUTER_IP> eq 23
access-list 105 dynamic <DYNAMIC_NAME> timeout 5 \
 permit ip any <DESTINATION> <WILDCARD>
!
interface <OUTSIDE_INTERFACE>
 ip access-group 105 in
!
line vty 0 4
 login local
 transport input telnet
 autocommand access-enable host timeout 5
```

확인 및 삭제:

```cisco
show access-lists 105
clear access-template
```

> 레거시 기능. 신규 환경에서는 VPN/ZBF/전용 방화벽 권장.

---

## 11. ⏰ Time-range ACL

### 반복 시간

```cisco
time-range BIZ_HOURS
 periodic weekdays 08:00 to 18:00
 periodic weekend 10:00 to 16:00
```

### 절대 기간

```cisco
time-range PROJECT_WINDOW
 absolute start 09:30 01 July 2026 end 18:30 29 August 2026
```

### ACL 연결

```cisco
ip access-list extended ACL_TIME_BASED
 permit tcp any host 13.13.4.4 eq 443 time-range BIZ_HOURS
 deny ip any any log
```

### 시간 및 NTP

```cisco
clock timezone KST 9
ntp server <NTP_SERVER_IP>
```

---

## 12. Log / Log-input

```cisco
deny ip any any log
deny tcp any host 192.0.2.10 eq 23 log-input
```

|옵션|기록 내용|
|---|---|
|`log`|ACL, 동작, 프로토콜, 주소, 포트|
|`log-input`|기본 로그 + 유입 인터페이스·가능한 경우 Source MAC|

> 대량 매칭 ACE에 적용하면 CPU와 Syslog 부하가 증가할 수 있습니다.

---

## 13. Fragment

```cisco
ip access-list extended ACL_ANTI_FRAG
 deny icmp any host 13.13.4.4 fragments
 permit ip any any
```

TCP MSS 조정:

```cisco
interface Tunnel0
 ip tcp adjust-mss 1360
```

> MSS는 터널 종류와 Path MTU를 계산한 후 설정합니다.

---

## 14. ACL 편집

### 시퀀스 추가

```cisco
ip access-list extended ACL_WEB
 15 permit tcp 10.10.20.0 0.0.0.255 host 192.0.2.10 eq 443
```

### 시퀀스 삭제

```cisco
ip access-list extended ACL_WEB
 no 15
```

### 시퀀스 재정렬

```cisco
ip access-list resequence ACL_WEB 10 10
```

### Numbered ACL 구성 모드

```cisco
ip access-list standard 1
 15 permit 10.10.20.0 0.0.0.255
```

> 플랫폼과 IOS 버전에 따라 지원 여부를 확인합니다.

### 전체 삭제

```cisco
no ip access-list standard ACL_NAME
no ip access-list extended ACL_NAME
no access-list 1
no access-list 101
```

---

## 15. 검증 명령어

```cisco
show access-lists
show access-lists <NUMBER>
show ip access-lists
show ip access-lists <ACL_NAME>
show ip interface <INTERFACE>
show running-config | section ip access-list
show running-config | include access-group
show running-config | section line vty
```

라우팅 프로토콜:

```cisco
show ip route
show ip protocols
show ip ospf neighbor
show ip eigrp neighbors
```

고급 ACL:

```cisco
show ip access-lists REFLECTIVE_SESSIONS
show ip access-lists 105
show time-range
show clock detail
show ntp status
show ntp associations
```

로그:

```cisco
show logging | include SEC-6-IPACCESSLOG
```

---

## 16. 적용 위치 암기표

|ACL 유형|기본 배치 원칙|이유|
|---|---|---|
|Standard|목적지 가까이|출발지만 검사하여 범위가 넓음|
|Extended|출발지 가까이|정밀 식별 후 조기 차단 가능|
|Reflective|경계 인터페이스|내부 시작 세션의 응답만 허용|
|VTY Standard|`line vty`의 `access-class in`|관리 접속 Source 제한|

> 위치 원칙은 절대 규칙이 아닙니다. 비대칭 라우팅, NAT, 터널, 장비 자체 트래픽, 성능 및 운영 정책을 함께 고려합니다.

---

> 📌 **핵심 요약**
> 
> - Standard: `access-list 1 permit <SOURCE> <WC>`
> - Extended: `permit tcp <SA> <WC> <DA> <WC> eq <PORT>`
> - 인터페이스 적용: `ip access-group <ACL> in|out`
> - VTY 적용: `access-class <ACL> in`
> - 편집: Named ACL + Sequence Number
> - 검증: `show ip access-lists`, `show ip interface`
> - 관련: 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리 · 3-1-3. ⚙️  고급 ACL - Reflective & Dynamic & Time-range · 3-1-4. 🧩 ACL 통합 정리 · 3-1-5. 🚑 ACL 트러블슈팅 치트시트
