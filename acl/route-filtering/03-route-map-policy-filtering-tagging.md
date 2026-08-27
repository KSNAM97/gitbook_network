# 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging

> **Tag:** #Network #Cisco #ACL #Security #AccessControl #RouteMap #PolicyRouting
> **핵심 요약:** ACL/Prefix-list 위에서 동작하는 정책 엔진 - 라우팅 세계의 if-then-else
> Match(조건) + Set(액션) 조합으로 필터링 + 속성 조작을 동시에 수행

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Distribute-list와 Route-map의 선택 기준

**단순 허용/차단**만 필요하면 Distribute-list, **속성 조작(tag, metric, next-hop) 또는 다단계 조건**이 필요하면 Route-map을 쓴다. Route-map은 if-then-else 구조라 "이 대역은 tag 100 붙이고, 저 대역은 tag 200 붙이고, 나머지는 차단" 같은 정책을 하나로 표현할 수 있다.

Distribute-list는 매치 조건만 있고 액션이 없어 통과/차단 이분법으로만 동작한다. 반면 Route-map은 `match`(조건) + `set`(액션) 조합으로 필터링, 마킹, 조작을 동시에 수행할 수 있다. BGP에서는 AS_PATH, LOCAL_PREF, MED 등 속성 제어가 필요하므로 Route-map이 사실상 필수다.

### Route-map Match 구문의 OR/AND 조건 차이

**같은 줄(space-separated)**은 OR 조건, **다른 줄(별도 match)**은 AND 조건이다. 이 차이를 모르면 정책이 정반대로 동작한다.

`match tag 100 200`은 tag가 100 **또는** 200이면 매치(OR)되고, `match tag 100` / `match ip address 1`처럼 두 줄로 쓰면 tag 100 **그리고** ACL 1을 동시에 만족해야 한다(AND). 실무에서 흔한 실수는 "여러 조건 중 하나만 맞아도 통과"하게 하려는데 별도 줄로 써서 AND로 동작해 아무것도 매치되지 않는 경우다.

### Route Tagging이 실무에서 필수인 이유

경로에 **메타데이터(tag)**를 붙여두면, 여러 라우터를 거치더라도 그 태그로 정책을 이어갈 수 있다. 특히 재분배(redistribution) 환경에서 **루프 방지**와 **경로 출처 추적**을 위해 반드시 필요하다.

재분배 시 tag를 붙여두면 다시 원래 프로토콜로 재유입되는 것을 감지·차단해 라우팅 루프를 방지할 수 있다. 다중 사이트/다중 벤더 환경에서 경로 출처를 추적하는 표준 방법이며, 송신 라우터에서 `set tag`로 부여하고 중간/최종 라우터에서 `match tag`로 활용하는 구조다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Route-map 기본 구조

```cisco
! [문법]
route-map <이름> {permit | deny} <시퀀스>
 match <조건1>                     ! AND 조건 (여러 줄 = 모두 만족해야 함)
 match <조건2> <조건3>             ! OR 조건 (같은 줄 = 하나만 만족하면 됨)
 set <액션>                         ! 매치 시 수행할 동작
!
! ★ 마지막에 암묵적 deny 존재 → 명시적 permit 라인이 반드시 필요
```
### 📌 Distribute-list와 Route-map 비교 (동일 정책 4가지 방식)
```cisco
! [시나리오] R3은 R5로부터 13.13.5.0/24, 13.13.15.0/24, 198.198.x.x/16 만 수신
!
! ===== 방식 1: ACL 기반 Distribute-list =====
access-list 1 permit 13.13.5.0 0.0.0.255
access-list 1 permit 13.13.15.0 0.0.0.255
access-list 1 permit 198.198.0.0 0.0.255.255
!
router eigrp 100
 distribute-list 1 in FastEthernet0/0
!
!
! ===== 방식 2: Prefix-list 기반 Distribute-list =====
ip prefix-list PFX_IN permit 13.13.5.0/24
ip prefix-list PFX_IN permit 13.13.15.0/24
ip prefix-list PFX_IN permit 198.198.0.0/16 le 32
!
router eigrp 100
 distribute-list prefix PFX_IN in FastEthernet0/0
!
!
! ===== 방식 3: Route-map + ACL =====
access-list 1 permit 13.13.5.0 0.0.0.255
access-list 1 permit 13.13.15.0 0.0.0.255
access-list 1 permit 198.198.0.0 0.0.255.255
!
route-map EIGRP_IN permit 10
 match ip address 1
!
router eigrp 100
 distribute-list route-map EIGRP_IN in FastEthernet0/0
!
!
! ===== 방식 4: Route-map + Prefix-list (실무 권장) =====
ip prefix-list PFX_IN permit 13.13.5.0/24
ip prefix-list PFX_IN permit 13.13.15.0/24
ip prefix-list PFX_IN permit 198.198.0.0/16 le 32
!
route-map EIGRP_IN permit 10
 match ip address prefix-list PFX_IN
!
router eigrp 100
 distribute-list route-map EIGRP_IN in FastEthernet0/0
!
```
### 📌 Tagging - 경로에 메타데이터 부여 (송신 측)
```cisco
! [시나리오] R4가 "199.171.8.0/24 ~ 199.171.15.0/24" 대역에 tag 100 부여 후 광고
!
access-list 41 permit 199.171.8.0 0.0.7.255           ! 8개 서브넷 (연속 대역)
!
route-map R4_TAG100 permit 10
 match ip address 41
 set tag 100                                            ! 매치되는 경로에 tag 100 부여
!
route-map R4_TAG100 permit 20                          ! ★ 나머지 경로도 통과시켜야 함 (tag 없이)
!
router eigrp 100
 distribute-list route-map R4_TAG100 out FastEthernet0/0
!
```
### 📌 Tag 기반 필터링 (수신 측)
```cisco
! [시나리오] R5는 tag 100이 붙은 경로를 EIGRP로 수신하지 않음 (ACL/Prefix-list 사용 금지)
!
route-map TAG100_DENY deny 10
 match tag 100                                          ! tag 100 매치 → 차단
!
route-map TAG100_DENY permit 20                        ! ★ 나머지는 전부 허용
!
router eigrp 100
 distribute-list route-map TAG100_DENY in FastEthernet0/0
!
```
### 📌 다단계 정책 - 사설 차단 + Tag 부여 + 나머지 처리
```cisco
! [시나리오]
!  1) R1은 R4로부터 수신하는 모든 사설 네트워크를 차단
!  2) 199.171.0.0/21 대역에는 tag 150 부여
!  3) 나머지 모든 네트워크에는 tag 100 부여
!
ip prefix-list PRIVATE permit 10.0.0.0/8 le 32
ip prefix-list PRIVATE permit 172.16.0.0/12 le 32
ip prefix-list PRIVATE permit 192.168.0.0/16 le 32
!
ip prefix-list NET199 permit 199.171.0.0/21 le 32
!
route-map EIGRP_FILTER deny 10                         ! 시퀀스 10: 사설 차단
 match ip address prefix-list PRIVATE
!
route-map EIGRP_FILTER permit 20                       ! 시퀀스 20: 199.171 대역 → tag 150
 match ip address prefix-list NET199
 set tag 150
!
route-map EIGRP_FILTER permit 30                       ! 시퀀스 30: 나머지 → tag 100
 set tag 100
!
router eigrp 100
 distribute-list route-map EIGRP_FILTER in FastEthernet0/0
!
```
### 📌 Match 조건의 OR / AND 예시
```cisco
! ===== OR 조건 (같은 줄) =====
route-map POLICY permit 10
 match tag 100 200 300                                  ! tag가 100 OR 200 OR 300 이면 매치
!
!
! ===== AND 조건 (다른 줄) =====
route-map POLICY permit 20
 match tag 100                                          ! tag 100 이면서
 match ip address prefix-list PFX_LIST                  ! prefix-list 도 매치해야 함
!
!
! ===== 실무 응용 - tag 여러 개 한 번에 차단 =====
route-map FILTER deny 10
 match tag 150 250                                      ! tag 150 또는 250 이면 차단
!
route-map FILTER permit 20                              ! 나머지 허용
!
```
### 📌 Match 조건 종류 (자주 쓰는 것)

|Match 구문|매치 대상|
|---|---|
|`match ip address <ACL>`|ACL로 지정한 네트워크|
|`match ip address prefix-list <PFX>`|Prefix-list로 지정한 네트워크|
|`match tag <값>`|경로에 부여된 tag 값|
|`match interface <intf>`|특정 인터페이스로 학습한 경로|
|`match ip next-hop prefix-list <PFX>`|next-hop IP 조건|
|`match ip route-source prefix-list <PFX>`|광고자(peer) IP 조건|
|`match metric <값>`|메트릭 값|
|`match route-type`|internal/external/level-1/level-2|

### 📌 Set 액션 종류 (자주 쓰는 것)

|Set 구문|액션|
|---|---|
|`set tag <값>`|경로에 tag 부여|
|`set metric <값>`|메트릭 재설정|
|`set metric-type <type>`|OSPF Type-1/Type-2 지정|
|`set local-preference <값>`|BGP Local Preference|
|`set as-path prepend <AS>`|BGP AS_PATH 조작|
|`set ip next-hop <IP>`|Next-hop 강제 지정 (PBR)|
|`set community <값>`|BGP Community 부여|

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어
```cisco
! Route-map 설정 확인
show route-map
show route-map EIGRP_FILTER
!
! 출력 예시:
! route-map EIGRP_FILTER, permit, sequence 20
!   Match clauses:
!     ip address prefix-lists: NET199
!   Set clauses:
!     tag 150
!   Policy routing matches: 0 packets, 0 bytes

! 경로별 tag 값 확인 (라우팅 테이블)
show ip route 199.171.8.0
! Routing entry for 199.171.8.0/24
!   Known via "eigrp 100", distance 90, metric 409600
!   Tag 100, type internal                              ← tag 확인

! EIGRP Topology에서 tag별 경로 조회
show ip eigrp topology | include tag
show ip eigrp topology | include tag is 100
show ip eigrp topology | include tag is 150

! Route-map 매치 통계 (hit count)
show route-map EIGRP_FILTER | include matches
```
### 🚨 트러블슈팅 시나리오 1: Route-map 적용 후 모든 경로가 사라졌다

- **증상:** Route-map으로 특정 대역만 차단하려 했는데 라우팅 테이블이 통째로 비어버림.
- **원인 파악:**
    1. `show route-map` → 시퀀스 순서 확인.
    2. Route-map도 **마지막에 암묵적 deny 존재** → 명시적 `permit` 라인이 없으면 전부 차단.
- **해결법:**
    
    ```cisco
    route-map FILTER deny 10
     match tag 100
    !
    route-map FILTER permit 20               ! ★ 반드시 마지막에 빈 permit 추가
    ! (match 없는 permit = 나머지 전부 허용)
    ```
    
- **재발 방지:** Route-map 작성 시 항상 "deny 조건들 → 마지막에 빈 permit" 패턴 유지.

### 🚨 트러블슈팅 시나리오 2: Match 조건이 OR인 줄 알았는데 AND로 동작

- **증상:** "tag 100 또는 200이면 차단" 하려고 두 줄로 썼는데 아무것도 차단 안 됨.
- **원인 파악:**
    - 두 줄로 쓰면 AND 조건 → tag가 동시에 100이면서 200일 수는 없으므로 매치 실패.
- **해결법:**
    
    ```cisco
    ! ❌ 잘못된 방식 (AND = 매치 안 됨)
    route-map WRONG deny 10
     match tag 100
     match tag 200
    !
    ! ✅ 올바른 방식 (OR = 같은 줄)
    route-map RIGHT deny 10
     match tag 100 200
    !
    ! ✅ 또는 시퀀스 분리
    route-map RIGHT_V2 deny 10
     match tag 100
    route-map RIGHT_V2 deny 20
     match tag 200
    route-map RIGHT_V2 permit 30
    ```
    

### 🚨 트러블슈팅 시나리오 3: Tag 설정했는데 다음 라우터에서 안 보임

- **증상:** R4에서 `set tag 100` 했는데 R1의 라우팅 테이블에는 tag가 안 보임.
- **원인 파악:**
    1. `show ip eigrp topology <네트워크>` → Topology에는 tag가 있는지 확인.
    2. Route-map을 **out 방향에 걸었는지** 확인. in에 걸면 자기 테이블에만 tag 부여됨.
    3. 재분배 환경이면 원본 프로토콜에서 tag가 유지되는지 확인.
- **해결법:**
    
    ```cisco
    ! 광고자(R4)에서 out 방향으로 설정해야 다음 라우터가 tag 인식
    router eigrp 100
     distribute-list route-map R4_TAG100 out FastEthernet0/0
    ```
    
- **주의:** RIP는 tag 필드 지원 (v2), EIGRP/OSPF/BGP 모두 지원. RIPv1은 지원 안 함.

### 🚨 트러블슈팅 시나리오 4: 재분배 후 Tag가 사라진다

- **증상:** EIGRP → OSPF로 재분배할 때 EIGRP에서 붙인 tag가 OSPF에서 안 보임.
- **원인 파악:**
    - 재분배 시 route-map을 명시적으로 걸지 않으면 tag가 전달되지 않을 수 있음.
- **해결법:**
    
    ```cisco
    ! 재분배 시 tag 유지 정책
    route-map REDIST_KEEP_TAG permit 10
     ! match 조건 없이 통과 → 원본 tag 유지
    !
    router ospf 1
     redistribute eigrp 100 subnets route-map REDIST_KEEP_TAG
    ```
    
- **재발 방지:** 재분배 구간에서는 반드시 tag 유지 여부를 검증하는 절차 필수.

### ⚠️ Route-map 실무 체크리스트

- [ ]  Route-map 마지막에 **명시적 permit** 라인이 있는가? (없으면 암묵적 deny)
- [ ]  Match 조건의 **OR(같은 줄) / AND(다른 줄)** 을 정확히 이해했는가?
- [ ]  Tag는 광고자(**out 방향**)에 걸어야 다음 라우터에서 활용 가능한 것을 알고 있는가?
- [ ]  시퀀스 번호를 **10, 20, 30** 처럼 여유 있게 배치해 추후 삽입 가능하도록 했는가?
- [ ]  재분배 환경에서 tag 유지를 검증했는가?
- [ ]  정책 변경 시 `clear ip <protocol> neighbors` 로 재수립하여 반영을 확인했는가?
- [ ]  BGP 환경에서는 Distribute-list 대신 **Route-map**을 표준으로 사용하는가?

---

> 관련: 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter · 3-2-2. ⛳  Distribute-list를 활용한 라우팅 필터링 · 3-2-4. ⚖️ Offset-list를 활용한 경로 조작 · 3-2-5. 🧩 라우팅 필터링 & 정책 제어 통합 정리 · 3-2-6. 🚑 라우팅 필터링 & 정책 제어 트러블슈팅 치트시트 · 3-2-7. ⚡ 라우팅 필터링 & 정책 제어 퀵 레퍼런스