# 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter

> **Tag:** #Network #Cisco #Routing #StaticRoute #CEF #ACL #Security #AccessControl #PrefixList #RouteFilter
> **핵심 요약:** ACL을 넘어선 네트워크 범위 매칭 도구 - Prefix-list의 문법과 실무 활용
> 인터넷 경계 라우터의 필수 정책, Bogon Filter로 스푸핑/오탐 차단

---

## 1. 🎯 핵심 기술 개념 (Concept)

### ACL 대신 Prefix-list를 쓰는 이유

ACL은 **서브넷마스크 길이(prefix length)를 조건으로 지정할 수 없다**. 예를 들어 "172.16.0.0/16 대역 중 /24 서브넷만 허용" 같은 정책은 ACL로 표현 불가능하지만, Prefix-list는 `ge`(greater equal), `le`(less equal) 옵션으로 정확히 매칭할 수 있다.

라우팅 필터링(Distribute-list, Route-map)에서는 **Prefix-list가 사실상 표준**이며, ACL은 트래픽 필터링에, Prefix-list는 경로 필터링에 사용한다. Prefix-list는 방화벽으로는 사용 불가하며 라우팅 업데이트 필터링 전용 도구다. Named 방식이 기본이라 시퀀스 번호로 라인 단위 편집이 가능해 유지보수성이 우수하다.

### Bogon Filter가 인터넷 경계 라우터에 필수인 이유

Bogon(사설 IP, 링크로컬, 문서용, 예약 대역 등)은 **공인 인터넷에서 라우팅되어서는 안 되는 대역**이다. 이런 주소가 출발지로 온다면 100% 스푸핑 공격이거나 잘못된 광고이므로, 경계에서 차단해야 DDoS/스푸핑 공격을 1차 방어할 수 있다.

실무에서는 **BGP 인바운드 정책**에 Bogon Prefix-list를 걸어 잘못된 광고를 원천 차단한다. Bogon 목록은 시간에 따라 변하며(예: 100.64.0.0/10 CGN 대역은 RFC 6598 이후 추가), **Team Cymru Bogon List** 등 외부 피드로 자동 업데이트하는 것이 실무 표준이다. 자기 AS에서 사설 IP를 인터넷으로 광고하지 않도록 **아웃바운드에도 동일 정책**을 적용해야 한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Prefix-list 문법 3가지 형태

```cisco
! [형태 1] 정확한 prefix 매칭 (한 개의 네트워크만)
ip prefix-list NET172 permit 172.16.0.0/16
! → 172.16.0.0/16 하나만 매칭. /17, /24 등은 매칭 안 됨.
!
! [형태 2] le (less equal) - 지정 prefix부터 특정 길이까지
ip prefix-list NET172 permit 172.16.0.0/16 le 32
! → 172.16.0.0/16 범위 안에서 서브넷마스크가 /16 ~ /32 인 모든 네트워크
!
! [형태 3] ge + le - 범위 내에서 특정 마스크 길이만
ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
! → 172.16.0.0/16 범위 안에서 서브넷마스크가 정확히 /24 인 네트워크만
```

### 📌 ge/le 조합 예시 (실무 필수 암기)

```cisco
! 예시 1: 10.0.0.0/8 대역의 모든 서브넷 (/8 ~ /32)
ip prefix-list NET10 permit 10.0.0.0/8 le 32
!
! 예시 2: 100.0.0.0/8 대역 중 /25 ~ /27 서브넷만
ip prefix-list NET100 permit 100.0.0.0/8 ge 25 le 27
!
! 예시 3: 182.160.0.0/16 대역 중 정확히 /24 서브넷만
ip prefix-list NET182 permit 182.160.0.0/16 ge 24 le 24
!
! 예시 4: 172.16.0.0 ~ 172.16.7.255 (연속 8개 /24 대역)
ip prefix-list NET172 permit 172.16.0.0/21 le 32
! /21 = 하위 3비트 = 8개 서브넷 커버
```

### 📌 Bogon Filter - 인터넷 경계 라우터 표준 정책

```cisco
! [Bogon Filter] 인터넷에 나타나면 안 되는 대역 전체 차단
!
ip prefix-list BOGON deny 0.0.0.0/8 le 32                 ! "This network" (RFC 1122)
ip prefix-list BOGON deny 10.0.0.0/8 le 32                ! 사설 (RFC 1918) - 사내망
ip prefix-list BOGON deny 100.64.0.0/10 le 32             ! CGN 가입자 NAT (RFC 6598)
ip prefix-list BOGON deny 127.0.0.0/8 le 32               ! Loopback (RFC 1122)
ip prefix-list BOGON deny 169.254.0.0/16 le 32            ! Link-local (RFC 3927)
ip prefix-list BOGON deny 172.16.0.0/12 le 32             ! 사설 (RFC 1918)
ip prefix-list BOGON deny 192.0.2.0/24 le 32              ! TEST-NET-1 (문서 예제용)
ip prefix-list BOGON deny 192.168.0.0/16 le 32            ! 사설 (RFC 1918)
ip prefix-list BOGON deny 198.18.0.0/15 le 32             ! Benchmark (RFC 2544)
ip prefix-list BOGON deny 198.51.100.0/24 le 32           ! TEST-NET-2 (문서 예제용)
ip prefix-list BOGON deny 203.0.113.0/24 le 32            ! TEST-NET-3 (문서 예제용)
ip prefix-list BOGON deny 224.0.0.0/4 le 32               ! Multicast (Class D)
ip prefix-list BOGON deny 240.0.0.0/4 le 32               ! Reserved (Class E)
ip prefix-list BOGON permit 0.0.0.0/0 le 32               ! 나머지 모든 대역 허용
!
! ▶ BGP 인바운드에 적용 (ISP 피어링 시나리오)
router bgp 65000
 neighbor <ISP-IP> prefix-list BOGON in
 neighbor <ISP-IP> prefix-list BOGON out                  ! 아웃바운드도 동일 적용 권장
!
```

### 📌 Bogon 대역 요약 (반드시 암기)

|대역|분류|실무 의미|
|---|---|---|
|`0.0.0.0/8`|This network|"내 네트워크" placeholder|
|`10.0.0.0/8`|사설 (RFC 1918)|대규모 사내망|
|`100.64.0.0/10`|CGN (RFC 6598)|ISP 가입자 NAT용|
|`127.0.0.0/8`|Loopback|자기 자신|
|`169.254.0.0/16`|Link-local|DHCP 실패 시 자동 할당|
|`172.16.0.0/12`|사설 (RFC 1918)|중규모 사내망|
|`192.0.2.0/24`|TEST-NET-1|문서 예제용|
|`192.168.0.0/16`|사설 (RFC 1918)|가정/SOHO|
|`198.18.0.0/15`|Benchmark|장비 성능 테스트용|
|`198.51.100.0/24`|TEST-NET-2|문서 예제용|
|`203.0.113.0/24`|TEST-NET-3|문서 예제용|
|`224.0.0.0/4`|Multicast|그룹 통신|
|`240.0.0.0/4`|Reserved|미래 예약 (Class E)|

### 📌 사설 IP 전용 필터 (내부용 축약 버전)
```cisco
! [사설 IP만 차단] 대외 인터페이스에서 사설 대역 광고/수신 차단
!
ip prefix-list PRIVATE deny 10.0.0.0/8 le 32
ip prefix-list PRIVATE deny 172.16.0.0/12 le 32
ip prefix-list PRIVATE deny 192.168.0.0/16 le 32
ip prefix-list PRIVATE permit 0.0.0.0/0 le 32
!
router eigrp 100
 distribute-list prefix PRIVATE in FastEthernet0/0        ! 수신 차단
 distribute-list prefix PRIVATE out FastEthernet0/0       ! 광고 차단
!
```

### 📌 Prefix-list 편집 (시퀀스 번호 활용)

```cisco
! 초기 생성 시 자동으로 seq 5, 10, 15... 부여
ip prefix-list NET172 permit 172.16.1.0/27
ip prefix-list NET172 permit 172.16.4.0/26
!
! ▶ 특정 시퀀스에 라인 삽입
ip prefix-list NET172 seq 7 permit 172.16.2.0/27
!
! ▶ 특정 라인 삭제
no ip prefix-list NET172 seq 10
!
! ▶ 전체 삭제
no ip prefix-list NET172
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어
```cisco
! 설정된 Prefix-list 전체 확인
show ip prefix-list
show ip prefix-list BOGON
!
! 출력 예시:
! ip prefix-list NET172: 4 entries
!    seq 5 permit 172.16.1.0/27
!    seq 10 permit 172.16.4.0/26
!    seq 15 permit 172.16.7.0/25
!    seq 20 permit 172.16.10.0/24

! 특정 네트워크가 어느 시퀀스에 매칭되는지 테스트
show ip prefix-list BOGON 10.0.0.0/8

! Prefix-list의 매칭 통계 (hit count)
show ip prefix-list detail BOGON

! 라우팅 테이블에 필터링 결과 반영 여부
show ip route eigrp | include <필터-대상-대역>
show ip route bgp | include <필터-대상-대역>
```

### 🚨 트러블슈팅 시나리오 1: Prefix-list 적용 후 정상 경로까지 사라졌다

- **증상:** Distribute-list에 Prefix-list 걸었더니 원하는 대역만 남아야 하는데 **전부 사라짐**.
- **원인 파악:**
    1. `show ip prefix-list` → 마지막 라인에 `permit 0.0.0.0/0 le 32` (또는 명시적 permit)이 있는지 확인.
    2. Prefix-list는 ACL과 동일하게 **암묵적 deny any** 존재 → 명시 permit 없으면 전부 차단.
- **해결법:**
    
    ```cisco
    ip prefix-list MYLIST deny 10.0.0.0/8 le 32
    ip prefix-list MYLIST permit 0.0.0.0/0 le 32     ! ★ 이 라인 반드시 추가
    ```
    

### 🚨 트러블슈팅 시나리오 2: `/24` 서브넷만 걸러내려는데 다른 마스크까지 잡힌다

- **증상:** `172.16.0.0/16 le 24` 로 걸었더니 `/16, /17, /18 ... /24` 까지 전부 매칭됨.
- **원인 파악:**
    - `le 24` = **서브넷마스크가 24 이하** 라는 뜻이 아니라, **prefix length ≤ 24** 라는 조건.
    - `/16` 자체도 조건에 포함됨 → 의도와 다름.
- **해결법:**
    
    ```cisco
    ! 정확히 /24 서브넷만 매칭하려면 ge와 le를 같은 값으로
    ip prefix-list NET172 permit 172.16.0.0/16 ge 24 le 24
    ```
    
- **암기 팁:**
    - `ge X le Y` = "서브넷마스크가 X ≤ mask ≤ Y 범위"
    - 정확히 하나의 마스크 길이만 원하면 `ge Y le Y` 로 명시.

### 🚨 트러블슈팅 시나리오 3: Bogon Filter 적용 후 CGN 대역 통신이 안 된다

- **증상:** 특정 모바일 사용자(CGN 이용자)가 접속이 안 됨.
- **원인 파악:**
    - `100.64.0.0/10` 대역이 Bogon 목록에 포함됨 → RFC 6598 CGN 대역.
    - ISP가 이 대역을 실제 가입자 NAT에 사용하는 경우 목적지 응답이 이 대역으로 감.
- **해결법:**
    - CGN 대역은 **입장에 따라 Bogon 여부가 달라짐** → ISP 환경/서비스 제공 대상에 따라 예외 처리.
    - 컨텐츠 서비스 사업자는 100.64.0.0/10을 Bogon에서 **제외**해야 CGN 이용자 서비스 가능.
- **재발 방지:**
    - Bogon 목록은 시간에 따라 갱신됨 → **Team Cymru Bogon Feed** 등으로 자동 갱신 검토.
    - 자체 서비스 특성(모바일/컨텐츠/기업망)에 따라 Bogon 리스트를 커스터마이즈.

### ⚠️ Prefix-list 실무 체크리스트

- [ ]  마지막에 `permit 0.0.0.0/0 le 32` 를 넣어 나머지를 명시적으로 허용했는가? (deny 정책일 경우)
- [ ]  `ge`/`le` 의미를 정확히 이해했는가? (prefix length 조건)
- [ ]  Bogon Filter는 **인바운드/아웃바운드 양방향**에 적용했는가?
- [ ]  CGN 대역(`100.64.0.0/10`)은 서비스 특성에 맞게 처리했는가?
- [ ]  Bogon 목록은 **정기적으로 갱신**하는 프로세스가 있는가? (Team Cymru 등)
- [ ]  BGP 피어링 시 Prefix-list 없이 Full-mesh 광고를 받고 있지는 않은가?

---

> 관련: 3-2-2. ⛳  Distribute-list를 활용한 라우팅 필터링 · 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging · 3-2-4. ⚖️ Offset-list를 활용한 경로 조작 · 3-2-5. 🧩 라우팅 필터링 & 정책 제어 통합 정리 · 3-2-6. 🚑 라우팅 필터링 & 정책 제어 트러블슈팅 치트시트 · 3-2-7. ⚡ 라우팅 필터링 & 정책 제어 퀵 레퍼런스