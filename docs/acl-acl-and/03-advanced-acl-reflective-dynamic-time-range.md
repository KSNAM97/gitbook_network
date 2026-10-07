# 고급 ACL - Reflective & Dynamic & Time-range

> **Tag:** #Network #Cisco #ACL #Security #AccessControl
> **핵심 요약:** 상태 기반(Stateful) 필터링과 시간·인증 기반 접근 제어
> ZBF/ASA 방화벽 도입 전 IOS 라우터에서 구현 가능한 최상위 ACL 기법

---

## 1. 핵심 기술 개념 (Concept)

### Reflective ACL과 `established`의 결정적 차이

`established`는 **TCP 플래그(ACK/RST)만** 검사하는 반쪽짜리 상태 검사인 반면, Reflective ACL은 **TCP/UDP/ICMP 모든 프로토콜**에 대해 내부→외부 세션을 감지해 임시 ACL을 자동 생성/삭제하는 **진짜 상태 기반(Stateful)** 필터링이다.

`established`는 UDP(DNS 응답)나 ICMP(Ping 응답)에 적용할 수 없지만, Reflective ACL은 이를 모두 커버한다. Reflective ACL은 세션 종료 후 자동으로 임시 ACL을 삭제(기본 300초 timeout)하므로 관리 부담도 없다. 다만 라우터 CPU/메모리를 소모하므로 대용량 트래픽 환경에서는 **ZBF(Zone-Based Firewall)**나 전용 방화벽 도입을 검토해야 한다.

### Dynamic ACL(Lock&Key)의 실무 활용 시나리오

평소에는 외부 접근을 전부 차단하다가, **인증된 사용자에게만 일시적으로 통로를 열어주는** 시나리오에 사용한다. 대표적으로 외부 협력사나 원격 관리자가 필요할 때만 Telnet/SSH로 라우터에 인증하면, 그 순간부터 지정된 내부망 접근이 허용된다.

VPN이 없던 시절의 원격 접근 솔루션으로 요즘은 대부분 IPsec/SSL VPN으로 대체되었다. 그러나 **네트워크 장비 원격 유지보수 벤더 접근 제어**나 임시 감사(auditor) 접근 등에는 여전히 유용하다. 인증 후 자동 삭제되는 임시 ACL이므로 방화벽 정책을 상시 개방하는 것보다 안전하다.

### Time-range 적용 시 반드시 확인해야 할 사항

라우터 시스템 시간이 **NTP로 동기화**되어 있어야 한다. Time-range는 라우터 로컬 시간을 기준으로 판단하기 때문에, NTP가 어긋나면 의도한 시간에 정책이 적용되지 않는다.

`absolute`는 특정 기간(일회성), `periodic`은 반복(요일/시간대)에 사용하며 목적에 맞게 선택해야 한다. `weekdays`(월~금), `weekend`(토~일), `daily`(매일) 예약어를 활용하면 간결한 정책 작성이 가능하다.

---

## 2. 표준 설정 템플릿 (Configuration)

### Reflective ACL - 진짜 Stateful 필터링

```cisco
! [시나리오] 내부→외부는 자유롭게 나가고, 외부→내부는 응답 트래픽만 허용
!
! (1) 내부→외부 방향 ACL: 나가는 트래픽을 감지해 임시 ACL 자동 생성
ip access-list extended IN_TO_OUT
 permit tcp any any reflect REFLECTIVE_SESSIONS
 permit udp any any reflect REFLECTIVE_SESSIONS
 permit icmp any any reflect REFLECTIVE_SESSIONS
!
! (2) 외부→내부 방향 ACL: 라우팅 프로토콜 + 임시 ACL 평가만 허용
ip access-list extended OUT_TO_IN
 permit ospf any any                         ! 라우팅 프로토콜 예외 필수
 evaluate REFLECTIVE_SESSIONS                ! ★ 임시 ACL 참조 → 응답만 통과
 deny ip any any log                         ! 나머지 외부발 신규 세션 차단
!
! (3) WAN 인터페이스에 in/out 각각 바인딩
interface Serial1/0
 ip access-group IN_TO_OUT out               ! 내부→외부 (out)
 ip access-group OUT_TO_IN in                ! 외부→내부 (in)
!
! ▶ 임시 ACL 타임아웃 조정 (default 300초)
ip reflexive-list timeout 120
```
### Dynamic ACL (Lock&Key) - 인증 기반 임시 개방

```cisco
! [시나리오] 외부에서 라우터에 Telnet 인증 성공 시 → 내부망 접근 5분간 허용
!
! (1) 인증용 사용자 계정 + autocommand 설정
username soladmin privilege 15 password solcisco
username soladmin autocommand access-enable host timeout 5
! access-enable = 인증 성공 시 dynamic ACL 활성화
! host = 인증한 사용자 IP만 개방
! timeout 5 = 5분 idle 시 자동 회수
!
! (2) VTY 라인에서 로컬 인증 사용
line vty 0 4
 login local
 transport input telnet ssh
!
! (3) Dynamic ACL 정의
access-list 105 permit tcp any host 13.13.3.3 eq 23      ! 인증용 Telnet 상시 개방
access-list 105 permit ospf any any                       ! 라우팅 프로토콜 예외
access-list 105 dynamic SOL_DYNAMIC timeout 5 permit ip any 13.13.15.0 0.0.0.255
! ★ dynamic 라인이 인증 성공 시 활성화 → 5분 후 자동 삭제
!
! (4) 외부향 인터페이스에 in 방향 적용
interface Serial1/0
 ip access-group 105 in
!
```
### Time-range - 시간 기반 접근 제어

```cisco
! [시나리오 1] 평일 업무시간에만 Telnet/HTTP 허용
!
time-range BIZ_HOURS
 periodic weekdays 08:00 to 18:00            ! 평일 8시~18시
 periodic weekend 10:00 to 16:00             ! 주말 10시~16시
!
! [시나리오 2] 특정 기간만 임시 개방 (프로젝트 오픈 기간)
time-range PROJECT_WINDOW
 absolute start 09:30 01 July 2026 end 18:30 29 August 2026
!
! (2) ACL에 time-range 바인딩
ip access-list extended ACL_TIME_BASED
 permit ospf any any                                                       ! 라우팅 프로토콜 상시 허용
 permit tcp any host 13.13.4.4 eq 23 time-range BIZ_HOURS                  ! Telnet은 업무시간만
 permit tcp any host 13.13.4.4 eq 80 time-range BIZ_HOURS                  ! HTTP는 업무시간만
 permit icmp any host 13.13.4.4 time-range PROJECT_WINDOW                  ! ICMP는 프로젝트 기간만
 deny ip any any log
!
interface FastEthernet0/0
 ip access-group ACL_TIME_BASED in
!
```

### Time-range 예약어 요약

|키워드|의미|
|---|---|
|`daily`|매일|
|`weekdays`|월~금|
|`weekend`|토~일|
|`monday`, `tuesday`, ...|특정 요일|
|`periodic`|반복 스케줄|
|`absolute start ... end ...`|일회성 기간 지정|

### NTP 동기화 (Time-range 사용 전 필수)
```cisco
! Time-range는 시스템 시간을 기준으로 동작 → NTP 동기화 필수
!
clock timezone KST 9                         ! 한국 표준시 (UTC+9)
!
ntp server 203.248.240.140                   ! KRISS 국가표준시
ntp server 210.98.16.100                     ! 백업 NTP 서버
!
! ▶ 검증
! show clock
! show ntp associations
! show ntp status
```
## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
! Reflective ACL - 임시 ACL 생성 여부 확인
show ip access-lists
show ip access-lists REFLECTIVE_SESSIONS
! Reflexive IP access list REFLECTIVE_SESSIONS
!   permit icmp host 13.13.5.5 host 13.13.4.4 (10 matches) (time left 267)

! Dynamic ACL - 인증 성공 시 동적으로 라인 생성 확인
show ip access-lists 105
! 30 Dynamic SOL_DYNAMIC permit ip any 13.13.15.0 0.0.0.255
!    permit ip host <인증한-사용자-IP> 13.13.15.0 0.0.0.255 (활성)

! Time-range - active/inactive 상태 확인
show time-range
show ip access-lists ACL_TIME_BASED
! 20 permit tcp any host 13.13.4.4 eq telnet time-range BIZ_HOURS (active)
!                                                                 ^^^^^^^^

! NTP 동기화 상태 확인 (Time-range 필수)
show clock detail
show ntp status
show ntp associations
```

### 트러블슈팅 시나리오 1: Reflective ACL 응답이 안 온다

- **증상:** 내부에서 Ping은 나가는데 응답이 안 돌아옴. Neighbor는 정상.
- **원인 파악:**
    1. `show ip access-lists REFLECTIVE_SESSIONS` → 임시 ACL이 생성됐는지 확인.
    2. 임시 ACL은 있는데 응답이 차단되면 → `OUT_TO_IN` ACL에서 `evaluate REFLECTIVE_SESSIONS` 문이 라우팅 프로토콜 permit **뒤에** 있는지 확인.
- **해결법:**
    - `evaluate` 문이 `deny ip any any` 앞에 위치해야 함.
    - 임시 ACL timeout이 너무 짧으면 응답 도착 전에 만료 → `ip reflexive-list timeout 300` 으로 조정.
- **재발 방지:** in/out 방향을 절대 반대로 걸지 말 것. `reflect` 는 out 방향에, `evaluate` 는 in 방향에.

### 트러블슈팅 시나리오 2: Dynamic ACL 인증 후에도 접근이 안 된다

- **증상:** Telnet 인증까지는 성공 → 하지만 내부망 접근 여전히 불가.
- **원인 파악:**
    1. `show ip access-lists 105` → dynamic 라인이 활성화됐는지 확인. 활성화 안 됐으면 → `autocommand access-enable` 설정 누락.
    2. `show running-config | include autocommand` → username 계정에 autocommand가 걸려있는지 확인.
- **해결법:**
    
    ```cisco
    username soladmin autocommand access-enable host timeout 5
    ! host 키워드가 없으면 모든 IP 대역이 열려버림 → 보안 사고 위험
    ```
    
- **주의:** `privilege 15` 계정으로 인증하면 autocommand가 실행되기 전에 exec 모드로 진입해서 dynamic ACL이 활성화 안 될 수 있음 → 별도 계정 사용 권장.

### 트러블슈팅 시나리오 3: Time-range가 있는데 정책이 안 걸린다

- **증상:** 업무시간인데 Telnet이 차단되거나, 업무시간 외인데 열려있음.
- **원인 파악:**
    1. `show clock` → 라우터 시간이 실제와 일치하는지 최우선 확인. 대부분 여기서 문제.
    2. `show ntp status` → `Clock is unsynchronized` 이면 NTP 미동기 상태.
    3. `show time-range` → active/inactive 상태와 현재 매칭 조건 확인.
- **해결법:**
    1. NTP 서버 설정 → `ntp server <IP>` 후 5~10분 대기.
    2. 임시 조치: `clock set 14:30:00 27 July 2026` 로 수동 세팅.
    3. Timezone 확인: `clock timezone KST 9` 없으면 UTC 기준으로 동작 → 9시간 어긋남.

### 고급 ACL 실무 체크리스트

- [ ]  Reflective ACL의 `reflect`(out) / `evaluate`(in) 방향을 정확히 바인딩했는가?
- [ ]  Reflective/Dynamic ACL 모두 **라우팅 프로토콜 permit을 최상단**에 두었는가?
- [ ]  Dynamic ACL의 username에 `autocommand access-enable host` 를 설정했는가?
- [ ]  Time-range 사용 전 **NTP 동기화 및 timezone** 설정을 완료했는가?
- [ ]  대규모 세션 환경에서는 **ZBF나 전용 방화벽 도입**을 검토했는가?
- [ ]  `absolute` 정책은 종료일 이후 자동 만료되므로 **정책 갱신 캘린더**를 관리하는가?

---

> 관련: 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리 · 3-1-4. 🧩 ACL 통합 정리 · 3-1-5. 🚑 ACL 트러블슈팅 치트시트 · 3-1-6. ⚡ ACL 명령어 퀵 레퍼런스 