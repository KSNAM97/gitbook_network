# Offset-list를 활용한 경로 조작

> **Tag:** #Network #Cisco #ACL #Security #AccessControl #OffsetList #MetricControl
> **핵심 요약:** 필터링이 아닌 "경로 우선순위 변경" - Metric을 조작해 최적 경로를 원하는 방향으로 유도
> ACL 매칭 + Metric 가산의 조합으로 이중화 환경의 트래픽 엔지니어링 구현

---

## 1. 핵심 기술 개념 (Concept)

### Offset-list가 ACL 폴더에서 다뤄지는 이유

Offset-list는 **ACL을 매칭 도구로 사용**해 특정 네트워크의 Metric에 값을 더하는 기능이다. 트래픽을 차단하는 것이 아니라 **경로의 우선순위를 낮춰서 다른 경로가 선택되게 만드는** 도구이며, ACL 기반 정책의 응용 사례로 이해하면 된다.

Distribute-list(허용/차단), Route-map(속성 조작)과 함께 ACL 3대 활용의 하나로, 경로 자체는 유지하되 우선순위만 조정해 **백업 경로로 강등**하는 효과를 낸다. EIGRP/RIP에서 주로 사용하며, OSPF는 링크스테이트 특성상 Offset-list를 지원하지 않아 대신 interface cost 조작을 활용한다.

### Offset-list가 Metric 증가만 허용하는 설계 이유

Metric을 임의로 **감소**시키면 라우팅 루프가 발생할 위험이 크기 때문이다. 이웃보다 Metric을 낮추면 트래픽이 자기 쪽으로 끌려오고, 상대 쪽에서도 같은 조작을 하면 무한 순환이 발생한다. 그래서 안전 방향인 증가 방향으로만 조작이 가능하게 제한된 것이다.

특정 경로를 더 좋게 만들고 싶으면, 반대편 경로를 offset으로 더 나쁘게 만들어 상대적 우위를 확보하는 방식으로 해결한다. 이중화된 WAN에서 "평소엔 A회선, 장애 시 B회선" 같은 정책을 구현할 때 자주 사용된다.

### Offset-list와 Route-map의 `set metric` 차이

Offset-list는 **기존 값에 더하는(add) 방식**이고, `set metric`은 **덮어쓰는(overwrite) 방식**이다. 원본 경로 특성을 살리면서 조정하려면 Offset-list를, 완전히 새로운 값을 강제하려면 `set metric`을 쓴다.

Offset-list는 원본 Metric에 상대적 가중치를 적용해 홉 수 변화에 자연스럽게 반영되는 반면, `set metric`은 절대값을 강제해 재분배 시나리오에서 초기값 설정에 유용하다. 실무에서는 **동일 프로토콜 내 경로 유도는 Offset-list**, **재분배 시 초기값 설정은 Route-map set metric**으로 역할을 나눈다.

---

## 2. 표준 설정 템플릿 (Configuration)

### Offset-list 기본 문법

```cisco
! [문법]
router <protocol>
 offset-list <ACL-번호|이름> {in | out} <가산값> [interface]
!
! - ACL: 조작할 네트워크 범위 매칭
! - in/out: 라우팅 업데이트 수신/송신 시점에 적용
! - 가산값: 원본 Metric에 더할 값
! - interface: 특정 인터페이스에만 적용 (생략 시 전체)
!
! ★ 하나의 라우팅 프로세스에서 인터페이스당 in/out 각 1회씩만 적용 가능
```
### 시나리오 1: 특정 목적지 경로를 우회로로 강등

```cisco
! [토폴로지]
!   R4 -- R1 --(Serial1/1 직결)-- R3 -- R5      ← 기본 경로 (짧음)
!   R4 -- R1 -- R2 -- R3 -- R5                   ← 우회 경로 (김)
!
! [요구사항] R4→R5의 13.13.15.0/24 통신이 R2 경유(우회)로 이뤄지도록 유도
!
! ===== 방식 A: 수신자(R1)에서 in 방향 =====
access-list 1 permit 13.13.15.0 0.0.0.255
!
router eigrp 100
 offset-list 1 in 700000 Serial1/1              ! Serial1/1로 들어오는 해당 경로에 700000 가산
!
!
! ===== 방식 B: 광고자(R3)에서 out 방향 =====
access-list 1 permit 13.13.15.0 0.0.0.255
!
router eigrp 100
 offset-list 1 out 700000 Serial1/1             ! Serial1/1로 나가는 광고에 700000 가산
!
```

### 시나리오 2: WAN 이중화 - Primary/Backup 회선 지정

```cisco
! [토폴로지] 본사↔지사가 두 개의 WAN 회선으로 이중화
!   MPLS 회선 (Serial0/0)  ← 평시 사용
!   Internet VPN (Serial0/1) ← 백업
!
! [정책] Internet VPN 경로의 Metric을 크게 올려 항상 후순위로
!
access-list 10 permit 10.0.0.0 0.255.255.255     ! 지사 전체 대역
!
router eigrp 100
 offset-list 10 in 10000000 Serial0/1            ! VPN 회선 경로만 강등
!
! ▶ 결과: MPLS 정상 시 MPLS 사용, MPLS 장애 시 자동으로 VPN 사용
```

### 시나리오 3: RIP 환경에서 홉 수 조작

```cisco
! [RIP 특성] Metric = 홉 수 (최대 15)
! [정책] 특정 대역의 홉 수에 2를 더해 우선순위 낮춤
!
access-list 20 permit 192.168.100.0 0.0.0.255
!
router rip
 offset-list 20 in 2 Serial0/0                  ! 홉 수 +2
!
! ▶ 주의: RIP는 최대 홉 15 → 가산 후 16 이상이면 unreachable 처리
!    실무에서는 이를 역이용해 특정 경로를 강제 차단 효과로도 사용
```

### 시나리오 4: 양방향 트래픽 대칭 유지

```cisco
! [문제] Outbound는 A회선, Inbound는 B회선으로 흐르는 비대칭 라우팅 발생
!        → 방화벽/QoS 정책 오작동 원인
!
! [해결] 양쪽 라우터 모두에서 Offset-list로 경로 대칭화
!
! ===== 본사 라우터 =====
access-list 30 permit 172.16.0.0 0.0.255.255    ! 지사 대역
router eigrp 100
 offset-list 30 in 5000000 Serial0/1            ! 지사→본사 방향 B회선 강등
!
! ===== 지사 라우터 =====
access-list 30 permit 10.0.0.0 0.255.255.255    ! 본사 대역
router eigrp 100
 offset-list 30 in 5000000 Serial0/1            ! 본사→지사 방향 B회선 강등
!
! ▶ 양쪽 다 A회선 우선 → 트래픽 대칭 확보
```
### 프로토콜별 Metric 단위 이해

|프로토콜|Metric 구조|Offset-list 가산 시 감각|
|---|---|---|
|**RIP**|홉 수 (1~15)|1~2 정도로 미세 조정|
|**EIGRP**|복합 (BW+Delay 기반, 32bit)|수십만~수백만 단위로 조정|
|**OSPF**|Cost (32bit)|Offset-list **미지원** → interface cost 조작|
|**BGP**|AS_PATH, LOCAL_PREF, MED|Offset-list 미지원 → Route-map 사용|

### EIGRP Metric 계산 감각 (실무 참고)
```cisco
! EIGRP Metric = (K1×BW + K3×Delay) × 256   (기본 공식)
!
! [샘플] Serial 회선(1544Kbps) 기본 Metric
!   BW = 10^7 / 1544 = 6476
!   Delay = 20000 μs / 10 = 2000
!   Metric = (6476 + 2000) × 256 ≈ 2,169,856
!
! [Offset 감각]
!   +100,000    → 미세 조정 (동급 회선 간 우선순위)
!   +500,000    → 명확한 우선순위 차이
!   +5,000,000  → 사실상 백업 경로로 강등
!   +10,000,000 → 최후의 수단 (거의 안 씀)
```
## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
! Offset-list 적용 확인
show ip protocols                                        ! offset-list 설정 위치 표시
show running-config | section router eigrp

! EIGRP Topology - 모든 경로의 Metric 비교
show ip eigrp topology all-links
show ip eigrp topology all-links | begin 13.13.15.0
!
! 출력 예시:
! P 13.13.15.0/24, 1 successors, FD is 2221056
!         via 13.13.10.3 (2221056/307200), Serial1/1          ← 최적 경로
!         via 13.13.9.2  (2921056/2221056), Serial1/0.123     ← 백업 (offset 적용 후)

! 라우팅 테이블에서 최종 선택된 경로 확인
show ip route eigrp | include 13.13.15.0
!
! 적용 전: D  13.13.15.0 [90/2221056] via 13.13.10.3, Serial1/1
! 적용 후: D  13.13.15.0 [90/2921056] via 13.13.9.2, Serial1/0.123

! 실제 트래픽 경로 검증 (가장 확실한 방법)
traceroute 13.13.15.5 source 13.13.14.4
```

### 트러블슈팅 시나리오 1: Offset-list 적용했는데 경로가 안 바뀐다

- **증상:** Offset 값을 넣었는데 여전히 같은 경로 사용.
- **원인 파악:**
    1. `show ip eigrp topology <네트워크>` → Metric이 실제로 증가했는지 확인.
    2. Offset이 너무 작아 여전히 다른 경로보다 우위일 가능성.
    3. ACL이 잘못 매칭됐거나, in/out 방향을 반대로 걸었을 가능성.
- **해결법:**
    
    ```cisco
    ! 1단계: 현재 두 경로의 Metric 차이 확인
    show ip eigrp topology all-links | include <네트워크>
    !
    ! 2단계: 차이보다 큰 값으로 offset 재설정
    ! 예: 두 경로 차이가 500,000이면 offset은 700,000 이상
    router eigrp 100
     no offset-list 1 in 100000 Serial1/1
     offset-list 1 in 700000 Serial1/1        ! 더 큰 값으로
    ```
    
- **재발 방지:** Offset 값은 반드시 **경로 간 Metric 차이보다 크게** 설정.

### 트러블슈팅 시나리오 2: Offset 후 경로가 아예 사라졌다

- **증상:** Offset 적용했더니 라우팅 테이블에서 해당 경로가 통째로 사라짐.
- **원인 파악:**
    - RIP에서 offset 가산 후 홉 수가 16 이상이 되면 unreachable로 처리.
    - EIGRP에서도 극단적으로 큰 값(수억)을 넣으면 Metric overflow 가능.
- **해결법:**
    
    ```cisco
    ! RIP의 경우
    ! 원본 홉 수 확인 후 최대 15 이하 유지되도록 조정
    show ip rip database
    !
    ! EIGRP의 경우 상식적 범위 내로 조정
    router eigrp 100
     no offset-list 1 in 999999999 Serial1/1
     offset-list 1 in 5000000 Serial1/1
    ```
    
- **재발 방지:** 프로토콜별 Metric 최대값 감각을 익힐 것.

### 트러블슈팅 시나리오 3: Primary 회선 장애 시 Backup으로 전환이 느리다

- **증상:** Offset으로 백업 경로 지정했는데, 주 회선 장애 시 30초~1분 이상 통신 지연.
- **원인 파악:**
    - EIGRP는 Hello 실패로 이웃 Down 감지까지 기본 15초(Hold-time) 소요.
    - Backup 경로가 Feasible Successor로 미리 계산돼 있지 않으면 재계산 시간 추가.
- **해결법:**
    
    ```cisco
    ! Hello/Hold-time 단축으로 감지 시간 축소
    interface Serial0/0
     ip hello-interval eigrp 100 5
     ip hold-time eigrp 100 15
    !
    ! BFD(Bidirectional Forwarding Detection) 도입 시 100ms 이내 감지 가능
    interface Serial0/0
     bfd interval 100 min_rx 100 multiplier 3
    !
    router eigrp 100
     bfd all-interfaces
    ```
    
- **재발 방지:** 이중화 설계 시 **감지 시간(convergence time)** 을 SLA에 포함시켜 검증.

### 트러블슈팅 시나리오 4: 비대칭 라우팅 발생

- **증상:** Outbound는 A회선, Return은 B회선으로 흐르며 방화벽에서 세션 끊김.
- **원인 파악:**
    - 한쪽 방향에만 Offset을 적용해서 반대 방향의 경로 선택은 그대로.
- **해결법:**
    - 양쪽 엔드포인트 라우터 모두에서 대칭적으로 Offset 정책 적용.
    - 특히 방화벽/IPS를 통과하는 트래픽은 대칭 라우팅이 절대 조건.

### Offset-list 실무 체크리스트

- [ ]  Offset 값이 경로 간 Metric 차이보다 **충분히 큰가**?
- [ ]  `in` / `out` 방향을 자기 라우터 기준으로 정확히 설정했는가?
- [ ]  양방향 트래픽 대칭이 필요한 경우 **양쪽 라우터 모두**에 Offset을 적용했는가?
- [ ]  RIP는 홉 수 15 초과 시 unreachable 되는 것을 인지하고 값을 설정했는가?
- [ ]  이중화 환경에서 **감지 시간 단축** (Hello timer, BFD)을 함께 고려했는가?
- [ ]  OSPF는 Offset-list가 아닌 **`ip ospf cost`** 로 조정해야 함을 알고 있는가?
- [ ]  Offset 적용 후 `traceroute`로 실제 트래픽 경로가 의도대로 흐르는지 검증했는가?

---

> 관련: 3-2-1. 🧩 Prefix-list 기초 & Bogon Filter · 3-2-2. ⛳  Distribute-list를 활용한 라우팅 필터링 · 3-2-3. 🗺️  Route-map을 활용한 정책 기반 필터링 & Tagging · 3-2-5. 🧩 라우팅 필터링 & 정책 제어 통합 정리 · 3-2-6. 🚑 라우팅 필터링 & 정책 제어 트러블슈팅 치트시트 · 3-2-7. ⚡ 라우팅 필터링 & 정책 제어 퀵 레퍼런스