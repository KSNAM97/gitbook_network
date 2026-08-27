# RIP (Routing Information Protocol) v1 & v2

> **Tag:** #Network #Cisco #RIP #Routing #IGP #DistanceVector
> **핵심 요약:** **Distance Vector 기반의 IGP(Interior Gateway Protocol)**
> AD 120 / Metric = Hop Count (Max 15) / UDP 520 사용

---

## 1. 핵심 기술 개념 (Concept)

### RIP가 사장된 이유와 그럼에도 학습해야 하는 이유

RIP는 **Hop Count 15의 확장성 한계**와 **30초 주기 Full Update로 인한 느린 Convergence Time** 때문에 실무 백본에서는 OSPF/EIGRP/BGP로 대체되었다. 그러나 **Distance Vector 알고리즘의 동작 원리와 라우팅 루프 방지 메커니즘(Split-horizon, Route Poison)의 원형(原型)**이기 때문에 반드시 이해해야 한다.

실무에서 RIP는 **소규모 지점망, 레거시 장비 연동, 특정 벤더 CPE**에서 여전히 발견되므로 마이그레이션 프로젝트 시 필수 지식이 된다. 또한 **AD 120**이라는 값은 다른 프로토콜(EIGRP 90, OSPF 110)과 함께 **Route Preference 설계 시 반드시 암기**해야 하는 기준값이다.

---

### RIPv1과 RIPv2의 결정적 차이와 v2 사용 당위성

RIPv1은 **Classful(서브넷 마스크 미포함) + Broadcast(255.255.255.255)** 방식이라 VLSM/CIDR 환경에서 사용이 불가한 반면, RIPv2는 **Classless(서브넷 마스크 포함) + Multicast(224.0.0.9) + MD5 인증**을 지원하므로 현대 네트워크에서는 v2가 필수다.

주의할 점은 RIPv2도 기본값으로 **`auto-summary`가 켜져 있어** Classful 요약이 발생한다는 것이다. 반드시 `no auto-summary` 명령으로 비활성화해야 VLSM이 정상 동작한다. RIPv2 Multicast(224.0.0.9)는 RIP를 사용하지 않는 호스트의 CPU 부하를 줄여주는 Broadcast 대비 명확한 이점도 갖는다.

---

## 2. 표준 설정 템플릿 (Configuration)

````
### [표준] RIPv2 Full 설정 (권장)

```cisco
! ========== R1 (Edge Router) ==========
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 network 172.16.0.0
 passive-interface loopback 172
 passive-interface fastethernet 0/0
!
````

```cisco
! ========== R2 (Transit Router) ==========
router rip
 version 2                                
 no auto-summary                          
 network 13.0.0.0                         
 passive-interface fastethernet 0/0       
!
```

```cisco
! ========== R3 (Edge Router) ==========
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 network 172.16.0.0
 passive-interface loopback 172
 passive-interface fastethernet 0/0
!
```
### MD5 인증 설정 (보안 강화 - 실무 필수)
```cisco
! ========== R1 & R2 공통 (양쪽 Key 반드시 일치) ==========
key chain RIP_AUTH                        
 key 20110117                             
  key-string cisco1234                    
!
interface serial 1/0                      
 ip rip authentication mode md5           
 ip rip authentication key-chain RIP_AUTH 
!
```

### 성능 최적화 옵션
```cisco
! ---------- Triggered RIP (WAN P2P 전용) ----------
interface serial 1/0
 ip rip triggered                         
!

! ---------- Unicast Update (NBMA / 특수 환경) ----------
router rip
 passive-interface serial 1/0             
 neighbor 13.13.12.2                      
!
```

### ⏱ RIP Timer 요약 (Convergence 이해 필수)
|Timer|Default|역할|
|---|---|---|
|**Update**|30초|주기적 Full Update 송신 (±15% Jitter)|
|**Invalid**|180초|갱신 없으면 경로 무효화 시작|
|**Holddown**|180초|잘못된 정보 수신 방지 (Loop 방지)|
|**Flushed**|240초|Routing Table에서 완전 제거|

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어
```cisco
! 라우팅 테이블 전체 확인
show ip route
show ip route rip                          

! RIP 프로세스 상태, 버전, Passive 인터페이스, 인증 확인
show ip protocol                           

! 실시간 RIP 패킷 송수신 확인 (가장 중요!)
debug ip rip                               

! 인증 Key 확인
show key chain                             

! End-to-End 통신 검증 (Source 지정 필수)
ping 172.16.3.3 source 13.13.10.254
```

### 정상 동작 시 `debug ip rip` 출력 예시
```text
RIP: sending v2 update to 224.0.0.9 via Serial1/0 (13.13.12.1)
RIP: build update entries
      13.13.10.0/24 via 0.0.0.0, metric 1, tag 0    ← Subnet Mask 포함 (Classless OK)
      172.16.1.0/24 via 0.0.0.0, metric 1, tag 0    ← auto-summary 해제 확인
```

---

### 장애 시나리오 #1 : "이웃 라우터의 경로가 학습되지 않는다"

**증상:** `show ip route rip` 결과 비어 있음. Ping 실패.

**단계별 트러블슈팅:**

1. **버전 불일치 점검**
    - `show ip protocol`로 R1/R2 양쪽 `send/receive version` 확인
    - **RIPv1 ↔ RIPv2는 기본적으로 통신 불가** → 양단 버전 통일
2. **`auto-summary` 확인**
    - VLSM 환경에서 `auto-summary`가 켜져 있으면 **불연속 서브넷(Discontiguous Network)** 문제 발생
    - `no auto-summary` 반드시 적용
3. **`passive-interface` 실수 여부**
    - 트래픽이 나가야 할 인터페이스가 passive로 잡혀 있는지 `show ip protocol`로 확인
4. **인증 불일치**
    - `debug ip rip`에서 `ignored v2 packet (invalid authentication)` 로그 발생 시 → Key ID / Key-string 양단 동일 여부 재확인
5. **Network 명령어 범위**
    - `network` 명령은 **Classful 주소로 입력**해야 함 (예: `network 172.16.1.0` ❌ → `network 172.16.0.0` ✅)

---

### 장애 시나리오 #2 : "특정 네트워크 다운 후 다른 라우터의 Convergence가 너무 느리다"

**증상:** 인터페이스 Shutdown 후 원격지에서 최대 **240초(Flush Timer)**까지 홀딩됨.

**단계별 트러블슈팅:**

1. **Route Poison 동작 확인**
    - `debug ip rip`에서 `metric 16, tag 0` 로그 확인 → 정상적으로 장애 전파 중
2. **Split-horizon 확인**
    - `show ip interface serial 1/0 | include split`
    - Frame-relay Multipoint 환경에서는 Split-horizon이 **오히려 문제**를 일으킬 수 있음 → 필요 시 `no ip split-horizon`
3. **Triggered RIP 도입 검토**
    - WAN P2P 구간이라면 `ip rip triggered` 적용 시 **네트워크 변경 시에만 즉시 Update 전송** → Convergence 대폭 개선
4. **근본적 대안 제시 (아키텍트 관점)**
    - RIP의 Timer 기반 Convergence는 구조적 한계 → **OSPF(SPF) 또는 EIGRP(DUAL) 마이그레이션이 정답**

---

### 실무 Tip: Route Poison vs Poison Reverse

|구분|방향|목적|
|---|---|---|
|**Route Poison**|장애 발생 라우터 → 이웃으로 전파|"이 경로는 죽었다"고 Metric 16으로 광고|
|**Poison Reverse**|이웃 → 다시 원래 라우터로 응답|"확실히 죽은 것으로 인지했다"고 Metric 16 회신 (Split-horizon 예외)|

> **핵심:** 두 메커니즘 모두 Count-to-Infinity 문제를 방지하고 Convergence를 가속화하는 안전장치.

---

> 관련: 1-2. ♟️ RIP 통합 정리 · 1-3. 🚑 RIP 트러블슈팅 치트시트 · 1-4. ⚡ RIP 명령어 퀵 레퍼런스