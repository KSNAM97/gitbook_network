# ⛳ Distribute-list를 활용한 라우팅 필터링

> **Tag:** #Network #Cisco #ACL #Security #AccessControl #DistributeList #RouteFilter
> **핵심 요약:** 라우팅 프로토콜 업데이트를 송/수신할 때 특정 네트워크만 걸러내는 표준 도구
> ACL 기반과 Prefix-list 기반의 차이, in/out 방향 결정 원칙

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Distribute-list in/out 방향 선택 기준

**자기 라우터를 기준**으로 판단한다. `in`은 이웃으로부터 **받은 업데이트를 자신의 라우팅 테이블에 넣기 전에** 필터링하고, `out`은 이웃에게 **보내기 직전에** 필터링한다. 원칙적으로는 **광고자 쪽에서 out으로 거르는 것**이 네트워크 자원을 절약하지만, 정책 소유권이 수신자에게 있으면 `in`을 쓴다.

`out`은 이웃에게 아예 광고 자체를 보내지 않아 네트워크 자원을 절약하고, `in`은 광고는 모두 받되 자기 테이블에만 넣지 않아 다른 이웃에게도 광고되지 않는다. EIGRP/OSPF는 DB 동기화 특성 때문에 `in` 필터링이 예상과 다르게 동작할 수 있으며, OSPF Type-1/2 LSA는 in으로 막지 못하고 Type-3 이후만 가능하다.

### 라우팅 필터링에 ACL 대신 Prefix-list를 쓰는 이유

**서브넷마스크 길이 조건**을 걸 수 있기 때문이다. ACL은 "출발지 IP 대역"만 매칭할 수 있어서 `/24`와 `/25`를 구분하지 못하지만, Prefix-list는 `172.16.0.0/16 ge 24 le 24`처럼 정확히 특정 마스크만 골라낼 수 있다.

실무에서는 **라우팅 필터링 = Prefix-list**, **트래픽 필터링 = ACL**로 역할을 분리하는 것이 표준이다. 다만 서브넷 길이 조건이 필요 없는 단순 필터링에서는 ACL이 오히려 가독성이 좋으며, Route-map과 결합할 때는 두 방식 모두 지원되므로 상황에 맞게 선택한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Distribute-list 기본 문법

```cisco
! [ACL 기반]
router <protocol> <as>
 distribute-list <ACL-번호|이름> {in | out} [interface]
!
! [Prefix-list 기반]
router <protocol> <as>
 distribute-list prefix <Prefix-list-이름> {in | out} [interface]
!
! ★ 하나의 라우팅 프로세스에서 인터페이스당 in/out 각 1개씩만 적용 가능
```
### 📌 ACL 기반 Distribute-list - 특정 대역만 수신
```cisco
! [시나리오] R3은 R5로부터 13.13.5.0/24, 13.13.15.0/24, 198.198.x.x/16만 수신
!
access-list 1 permit 13.13.5.0 0.0.0.255
access-list 1 permit 13.13.15.0 0.0.0.255
access-list 1 permit 198.198.0.0 0.0.255.255
! 나머지는 암묵적 deny → 수신 차단
!
router eigrp 100
 distribute-list 1 in FastEthernet0/0
!
```
### 📌 Prefix-list 기반 Distribute-list - 서브넷 길이 조건 활용
```cisco
! [시나리오] 172.16.0.0/16 범위 중 /25 ~ /27 서브넷만 수신
!
ip prefix-list NET172 permit 172.16.0.0/16 ge 25 le 27
!
router eigrp 100
 distribute-list prefix NET172 in FastEthernet0/0
!
```
### 📌 특정 대역 제외 + 나머지 허용 (실무 표준 패턴)
```cisco
! [시나리오] 172.16.0.0/16 대역만 차단, 나머지는 전부 수신
!
ip prefix-list FILTER deny  172.16.0.0/16 le 32       ! 172.16.x.x 는 무조건 차단
ip prefix-list FILTER permit 0.0.0.0/0 le 32          ! 나머지는 전부 허용 (필수)
!
router eigrp 100
 distribute-list prefix FILTER in FastEthernet0/0
!
```
### 📌 사설 IP 차단 - 인터넷 경계 라우터 표준 정책
```cisco
! [시나리오] 사설 대역이 EIGRP로 유입되는 것을 전면 차단
!
ip prefix-list PRIVATE deny 10.0.0.0/8 le 32
ip prefix-list PRIVATE deny 172.16.0.0/12 le 32
ip prefix-list PRIVATE deny 192.168.0.0/16 le 32
ip prefix-list PRIVATE permit 0.0.0.0/0 le 32
!
router eigrp 100
 distribute-list prefix PRIVATE in Serial1/0
 distribute-list prefix PRIVATE in Serial1/1
 distribute-list prefix PRIVATE in FastEthernet0/0
!
! ▶ 대칭 정책 - 자기 사설 IP를 외부로 광고하지 않도록 out에도 적용
router eigrp 100
 distribute-list prefix PRIVATE out Serial1/0
!
```
### 📌 광고자 vs 수신자 - 어디에 걸어야 하는가?
```cisco
! ===== 방식 A: 광고자(R5)에서 out 필터링 (권장) =====
! → 광고 자체를 안 보내므로 네트워크/CPU 자원 절약
!
! R5:
access-list 1 permit 13.13.5.0 0.0.0.255
access-list 1 permit 13.13.15.0 0.0.0.255
!
router eigrp 100
 distribute-list 1 out FastEthernet0/0
!
!
! ===== 방식 B: 수신자(R3)에서 in 필터링 =====
! → 광고는 다 받되 자기 테이블에만 반영 안 함
! → 정책 소유권이 수신 측에 있을 때 사용
!
! R3:
access-list 1 permit 13.13.5.0 0.0.0.255
access-list 1 permit 13.13.15.0 0.0.0.255
!
router eigrp 100
 distribute-list 1 in FastEthernet0/0
!
```
### 📌 프로토콜별 적용 특이사항
```cisco
! ===== RIP - 가장 단순, in/out 모두 자유롭게 =====
router rip
 distribute-list prefix FILTER in
 distribute-list prefix FILTER out Serial0/0
!
!
! ===== EIGRP - in/out 모두 지원, 인터페이스별 세밀 제어 가능 =====
router eigrp 100
 distribute-list prefix FILTER in FastEthernet0/0
 distribute-list prefix FILTER out Serial1/0
!
!
! ===== OSPF - Type-3 이후 LSA만 필터 가능 (in 방향 특히 주의) =====
router ospf 1
 distribute-list prefix FILTER in                   ! 자신의 라우팅 테이블 반영만 차단
 distribute-list prefix FILTER out <프로세스>       ! 재분배(redistribute)된 경로만 필터
!
! ★ OSPF는 LSA 플러딩을 막지 못하므로 진짜 필터링은 ABR/ASBR에서 area filter로 처리
```
## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어
```cisco
! Distribute-list 적용 상태 확인
show running-config | section router eigrp
show ip protocols                                    ! Distribute-list가 어디에 걸렸는지

! 필터링 결과 라우팅 테이블 확인
show ip route eigrp
show ip route eigrp | include FastEthernet0/0        ! 특정 인터페이스로 수신한 경로만
show ip route eigrp | include 172.16                 ! 특정 대역만 조회

! 매칭 도구(Prefix-list/ACL) 자체 확인
show ip prefix-list
show ip prefix-list detail FILTER                    ! hit count 포함
show access-lists 1

! EIGRP Topology에서 필터링 전 원본 상태 확인
show ip eigrp topology all-links                     ! 필터 적용 전 원본 정보
```
### 🚨 트러블슈팅 시나리오 1: Distribute-list 적용 후 모든 경로가 사라졌다

- **증상:** 특정 대역만 차단하려 했는데 라우팅 테이블에서 EIGRP 경로가 통째로 사라짐.
- **원인 파악:**
    1. `show ip prefix-list` 또는 `show access-lists` → **마지막에 `permit any` 또는 `permit 0.0.0.0/0 le 32` 라인이 있는지** 확인.
    2. 없으면 → 암묵적 deny에 걸려 전부 차단됨.
    - **해결법:**
    
    ```cisco
    ip prefix-list FILTER deny 172.16.0.0/16 le 32
    ip prefix-list FILTER permit 0.0.0.0/0 le 32       ! ★ 이 라인 누락이 원인
    ```

- **재발 방지:** 필터링 정책 작성 시 항상 "차단 → 명시적 허용" 두 단계로 구성.

### 🚨 트러블슈팅 시나리오 2: Prefix-list 수정했는데 라우팅 테이블이 안 바뀐다

- **증상:** Prefix-list 라인을 추가/수정했는데 `show ip route`에는 변화가 없음.
- **원인 파악:**
    - Distribute-list는 **새로운 업데이트가 올 때** 필터링 적용 → 기존 학습된 경로는 즉시 반영 안 됨.
    - EIGRP는 특히 Topology에 이미 있는 정보를 재검증하지 않음.
- **해결법:**
    
    ```cisco
    ! EIGRP 이웃 재수립 → 전체 경로 재학습 강제
    clear ip eigrp neighbors
    !
    ! 또는 특정 이웃만
    clear ip eigrp neighbors 150.3.13.5
    !
    ! RIP은 자동 갱신 대기 (30초) 또는
    clear ip route *
    ```
    

### 🚨 트러블슈팅 시나리오 3: OSPF Distribute-list가 예상대로 동작 안 함

- **증상:** OSPF에서 `distribute-list in` 걸었는데 이웃 라우터의 테이블에는 여전히 해당 경로가 있음.
- **원인 파악:**
    - OSPF의 `distribute-list in` 은 **LSA 플러딩을 막지 못한다** → LSDB에는 정보가 그대로 있고, 자기 라우팅 테이블에만 반영 안 하는 것.
    - 옆 라우터로는 LSA가 그대로 전달되므로 근본적 차단이 아님.
- **해결법:**
    1. **ABR에서 area 필터링** 사용:
        
        ```cisco
        router ospf 1
         area 1 filter-list prefix FILTER in
        ```
        
    2. **재분배 필터링**은 `distribute-list out <프로세스>` 로 가능.
    3. 근본적으로 OSPF는 링크스테이트 특성상 LSA 필터가 제한적 → **경계는 ABR/ASBR** 로 설계.

### 🚨 트러블슈팅 시나리오 4: EIGRP에서 out 필터가 원거리 이웃까지 영향 미침

- **증상:** R1에서 R2 방향으로 out 필터를 걸었는데 R3의 라우팅 테이블에서도 해당 경로가 사라짐.
- **원인 파악:**
    - EIGRP는 **광고자에서 필터링하면 하위 이웃 모두에게 영향** → R1이 R2에게 광고 안 하면 R2도 R3에게 광고 못 함.
    - 이는 정상 동작이지만 의도치 않은 결과를 낳을 수 있음.
- **해결법:**
    - 특정 이웃에게만 차단하려면 광고 방향(interface)을 정확히 지정.
    - 세밀한 정책이 필요하면 **Route-map 기반 Distribute-list** 로 전환 (Chapter 4-3에서 다룸).

### ⚠️ Distribute-list 실무 체크리스트

- [ ]  Prefix-list/ACL 마지막에 **명시적 permit** 을 넣어 정상 경로가 살아있는가?
- [ ]  `in` / `out` 방향을 **자기 라우터 기준**으로 정확히 판단했는가?
- [ ]  필터링 후 반드시 `clear ip <protocol> neighbors` 로 재수립하여 반영 확인했는가?
- [ ]  OSPF는 `distribute-list in` 의 한계를 이해하고 **ABR filter-list** 로 대체했는가?
- [ ]  광고자 out 필터가 **하위 라우터 전체에 영향**을 미치는 것을 예상했는가?
- [ ]  서브넷 길이 조건이 필요하면 ACL이 아닌 **Prefix-list** 를 사용했는가?

---

> 관련: 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter · 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging · 3-2-4. ⚖️ Offset-list를 활용한 경로 조작 · 3-2-5. 🧩 라우팅 필터링 & 정책 제어 통합 정리 · 3-2-6. 🚑 라우팅 필터링 & 정책 제어 트러블슈팅 치트시트 · 3-2-7. ⚡ 라우팅 필터링 & 정책 제어 퀵 레퍼런스