# 🚑 Cisco IOS ACL 트러블슈팅 치트시트

> **Tag:** #Network #ACL #Troubleshooting #StandardACL #ExtendedACL #ReflectiveACL #DynamicACL
> **핵심 요약:** ACL 장애는 **적용 위치/방향 오류, 암묵적 deny, ACE 순서 오류, 라우팅 프로토콜 예외 누락, Return Traffic 차단**에서 가장 많이 발생한다. 진단은 `ACL 바인딩 → ACE 순서 → Hit Count → 제어 트래픽 → 왕복 경로` 순서로 진행한다.

---

## 1. 🎯 ACL 장애 진단 순서

```properties
[1] ACL이 어느 인터페이스의 어느 방향에 적용됐는가?
[2] 패킷의 실제 출발지와 목적지는 무엇인가?
[3] 어느 ACE에 최초로 매칭되는가?
[4] 정상 트래픽 permit이 암묵적 deny보다 앞에 있는가?
[5] 라우팅 프로토콜·DHCP·DNS 등 제어 트래픽이 차단되지 않았는가?
[6] 요청뿐 아니라 응답 경로도 허용되어 있는가?
```
```cisco
show ip interface <INTERFACE>
show ip access-lists <ACL_NAME>
show access-lists
show ip route <DESTINATION>
ping <DESTINATION> source <SOURCE>
traceroute <DESTINATION>
```

---

## 2. 🛠️ 증상별 즉시 대응표

|증상|주요 원인|확인|조치|
|---|---|---|---|
|ACL 적용 후 전체 통신 단절|암묵적 deny|`show ip access-lists`|필요한 permit 추가|
|차단해야 할 트래픽이 통과|상단의 넓은 permit|ACE 순서·hit count|구체적 deny를 위로 이동|
|ACL이 전혀 동작하지 않음|미적용/방향 오류|`show ip interface`|올바른 in/out으로 적용|
|OSPF/EIGRP Neighbor Down|제어 패킷 차단|Neighbor·ACL count|프로토콜 permit 추가|
|요청은 나가지만 응답 없음|Return Traffic 차단|역방향 ACL|응답 permit/Reflective 검토|
|SSH 접속 불가|VTY ACL 오류|`show run|section line vty`|
|Time-range 오동작|시간/NTP 오류|`show clock`, `show time-range`|timezone/NTP 수정|
|ACL 수정 후 전체 정책 삭제|`no access-list` 사용|Running config|백업에서 복원|
|Ping은 되지만 대용량 전송 실패|MTU/Fragment/PMTUD|DF Ping·ACL|MTU/MSS/ICMP 정책 수정|
|CPU/Syslog 부하 증가|과도한 `log`|CPU·logging|필요한 deny에만 제한 사용|

---

## 3. 🔍 핵심 확인 명령어

### ACL 내용 및 Hit Count

```cisco
show access-lists
show access-lists 1
show ip access-lists ACL_WAN_IN
```

### 인터페이스 바인딩

```cisco
show ip interface FastEthernet0/0
show ip interface Serial1/0 | include access list
show running-config | include access-group
```

### VTY 접근 제어

```cisco
show running-config | section line vty
show ip access-lists ACL_VTY_ADMIN
```

### 라우팅 프로토콜

```cisco
show ip route
show ip protocols
show ip ospf neighbor
show ip eigrp neighbors
```

### Reflective/Dynamic/Time-range

```cisco
show ip access-lists REFLECTIVE_SESSIONS
show ip access-lists 105
show time-range
show clock detail
show ntp status
show ntp associations
```

### 로그

```cisco
show logging | include SEC-6-IPACCESSLOG
```

---

## 4. 🚨 대표 장애 시나리오

### 시나리오 1. Deny 한 줄 추가 후 전체 통신이 끊김

#### 증상

```cisco
ip access-list standard ACL_BLOCK_HOST
 deny 192.168.1.0 0.0.0.255
```

#### 원인

마지막 암묵적 `deny any` 때문에 지정한 대역뿐 아니라 나머지 출발지도 모두 차단됩니다.

#### 조치

```cisco
ip access-list standard ACL_BLOCK_HOST
 10 deny 192.168.1.0 0.0.0.255
 20 permit any
```

Extended ACL이라면:

```cisco
ip access-list extended ACL_BLOCK_SERVICE
 10 deny tcp any host 192.0.2.10 eq 23
 20 permit ip any any
```

---

### 시나리오 2. ACL이 설정돼 있는데 차단되지 않음

#### 원인 후보

1. ACL이 인터페이스에 적용되지 않음
2. `in/out` 방향이 반대
3. 더 위의 넓은 permit에 먼저 매칭
4. 패킷의 실제 주소가 NAT 등으로 예상과 다름
5. ACL이 해당 트래픽 경로에 존재하지 않음

#### 확인

```cisco
show ip interface Serial1/0
show ip access-lists ACL_WAN_IN
show ip route <DESTINATION>
traceroute <DESTINATION>
```

#### 잘못된 순서

```cisco
ip access-list extended ACL_WEB
 10 permit ip any any
 20 deny tcp any host 192.0.2.10 eq 80
```

#### 수정

```cisco
ip access-list extended ACL_WEB
 10 deny tcp any host 192.0.2.10 eq 80 log
 20 permit ip any any
```

---

### 시나리오 3. Standard ACL 적용 후 다른 목적지까지 차단됨

#### 원인

Standard ACL은 목적지를 확인하지 않으므로 출발지 가까이 적용하면 해당 출발지의 모든 트래픽에 영향을 줄 수 있습니다.

#### 조치

- ACL을 목적지 가까운 인터페이스로 이동
- 특정 목적지/서비스만 제어해야 한다면 Extended ACL로 변경

```cisco
ip access-list extended ACL_PRECISE_BLOCK
 10 deny ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255
 20 permit ip any any
```

---

### 시나리오 4. ACL 적용 후 OSPF Neighbor가 Down

#### 확인

```cisco
show ip ospf neighbor
show ip access-lists ACL_WAN_IN
show logging
```

#### 원인

OSPF IP Protocol 89가 암묵적 deny 또는 명시적 deny에 의해 차단됐을 가능성이 있습니다.

#### 조치

```cisco
ip access-list extended ACL_WAN_IN
 5 permit ospf host 13.13.12.2 any
```

> 실제 Neighbor 주소와 ACL 방향에 맞춰 source/destination을 제한합니다.

Neighbor 회복 확인:

```cisco
show ip ospf neighbor
show ip route ospf
```

> ACL 수정 후 일반적으로 Neighbor가 자동 재수립되므로, 운영망에서 곧바로 `clear ip ospf process`를 실행하지 않습니다. 먼저 자연 회복 여부를 확인합니다.

---

### 시나리오 5. EIGRP Hello는 보이지만 Neighbor가 불안정

#### 원인

멀티캐스트 Hello만 허용하고 유니캐스트 Update/Query/Reply를 차단했을 수 있습니다.

#### 조치

```cisco
ip access-list extended ACL_WITH_EIGRP
 permit eigrp host 13.13.12.2 host 224.0.0.10
 permit eigrp host 13.13.12.2 host 13.13.12.1
```

더 단순하게 Neighbor의 EIGRP 패킷 전체 허용:

```cisco
ip access-list extended ACL_WITH_EIGRP
 permit eigrp host 13.13.12.2 any
```

검증:

```cisco
show ip eigrp neighbors
show ip route eigrp
```

---

### 시나리오 6. Numbered ACL 수정 중 전체 ACL 삭제

#### 위험 명령

```cisco
no access-list 1
no access-list 101
```

#### 긴급 조치

1. 인터페이스에 잘못된 ACL이 남아 있는지 확인
2. 필요하면 ACL 바인딩을 임시 해제
3. 백업된 설정으로 ACL 재생성
4. 검증 후 다시 적용

```cisco
interface FastEthernet0/0
 no ip access-group 1 out
```

최신 IOS에서 지원되는 경우:

```cisco
ip access-list standard 1
 no 20
 25 permit 10.10.30.0 0.0.0.255
```

> 지원 여부는 플랫폼과 IOS 버전에 따라 확인합니다. 운영망에서는 Named ACL 사용을 권장합니다.

---

### 시나리오 7. VTY ACL 적용 후 SSH 접속 불가

#### 확인

```cisco
show running-config | section line vty
show ip access-lists ACL_VTY_ADMIN
```

#### 조치

콘솔 또는 별도 관리 경로로 접속한 후 관리자 대역을 추가합니다.

```cisco
ip access-list standard ACL_VTY_ADMIN
 15 permit 10.10.30.0 0.0.0.255
```

VTY 적용 확인:

```cisco
line vty 0 4
 access-class ACL_VTY_ADMIN in
 transport input ssh
```

> 원격 변경 시 현재 관리 세션의 Source IP를 먼저 permit해야 Lockout을 방지할 수 있습니다.

---

### 시나리오 8. Reflective ACL에서 응답이 차단됨

#### 확인

```cisco
show ip access-lists REFLECTIVE_SESSIONS
show ip access-lists OUT_TO_IN
```

#### 원인 후보

- `reflect`와 `evaluate` 방향 반대
- `evaluate`가 deny 아래에 배치
- Outbound 트래픽이 reflect ACE보다 먼저 다른 ACE에 매칭
- 세션 timeout 만료
- FTP 등 포트가 바뀌는 애플리케이션 사용

#### 정상 예시

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
```

Timeout 조정:

```cisco
ip reflexive-list timeout 300
```

---

### 시나리오 9. Dynamic ACL 인증 후에도 접근 불가

#### 확인

```cisco
show ip access-lists 105
show running-config | section line vty
show running-config | include access-enable
```

#### 점검 사항

- Dynamic ACE가 실제 생성됐는가?
- 모든 사용 VTY에 `autocommand access-enable`이 적용됐는가?
- `host` 키워드가 있는가?
- Idle/Absolute timeout이 설정됐는가?
- 인증용 Telnet 트래픽이 ACL에서 허용됐는가?

```cisco
line vty 0 4
 login local
 transport input telnet
 autocommand access-enable host timeout 5
```

임시 Entry 수동 삭제:

```cisco
clear access-template
```

> Dynamic ACL은 Telnet 의존성과 Spoofing 위험이 있으므로 신규 설계에서는 사용하지 않는 것이 좋습니다.

---

### 시나리오 10. Time-range가 반대로 동작

#### 확인

```cisco
show clock detail
show time-range
show ntp status
show ntp associations
```

#### 조치

```cisco
clock timezone KST 9
ntp server <ORGANIZATION_NTP_IP>
```

임시 시간 설정:

```cisco
clock set 14:30:00 27 July 2026
```

> 수동 설정은 임시 조치이며 장기적으로 NTP 동기화를 구성해야 합니다.

---

### 시나리오 11. Ping은 되지만 대용량 TCP만 실패

#### 원인 후보

- MTU 초과
- PMTUD ICMP 차단
- 비초기 Fragment 차단
- 터널 오버헤드 미반영
- TCP MSS 값 부적절

#### 확인

```cisco
ping <DESTINATION> size 1400 df-bit
show ip interface <INTERFACE>
show ip access-lists ACL_ANTI_FRAG
```

#### 조치 예시

```cisco
interface Tunnel0
 ip tcp adjust-mss 1360
```

> 무조건 MSS 1360을 적용하지 말고 실제 Path MTU와 터널 오버헤드에 맞춰 계산합니다.

---

## 5. 🔧 안전한 ACL 변경 절차

```properties
[1] 현재 ACL 및 인터페이스 바인딩 백업
[2] 현재 관리 Source IP가 permit되는지 확인
[3] 신규 ACL을 인터페이스에 적용하기 전에 완성
[4] 시퀀스 번호로 신규 ACE 삽입
[5] Hit Count 및 서비스 통신 검증
[6] 불필요한 기존 ACE 삭제
[7] Running-config 저장
```

```cisco
show running-config | section ip access-list
show running-config | include access-group
show ip access-lists
copy running-config startup-config
```

---

> 📌 **핵심 요약**
> 
> - 첫 확인은 **적용 인터페이스와 in/out 방향**
> - 다음 확인은 **ACE 순서와 Hit Count**
> - 전체 단절은 대부분 **암묵적 deny**
> - Neighbor Down은 **OSPF/EIGRP/RIP 제어 패킷 예외**
> - 요청 성공·응답 실패는 **Return ACL 또는 Stateful 정책**
> - ACL 변경 전 현재 관리 IP permit 여부를 반드시 확인
> - 관련: 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리 · 3-1-3. ⚙️  고급 ACL - Reflective & Dynamic & Time-range · 3-1-4. 🧩 ACL 통합 정리 · 3-1-6. ⚡ ACL 명령어 퀵 레퍼런스