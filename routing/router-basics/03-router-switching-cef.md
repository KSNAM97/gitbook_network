# 라우터의 스위칭 방식 (CEF)

> **Tag:** #Network #Cisco
> **핵심 요약:** **문서 목적:** Cisco 라우터의 3대 스위칭 방식(Process/Fast/CEF)과 FIB/Adjacency Table의 동작 원리를 데이터 플레인 관점에서 정리한다. 라우팅 프로토콜(컨트롤 플레인)과는 별개의 레이어임에 주의.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. 컨트롤 플레인(Routing Protocol)과 데이터 플레인(Switching Method)의 차이를 설명해 주세요.

- 👉 **핵심 답변:** **컨트롤 플레인은 "경로를 결정"**하고(OSPF/EIGRP 등), **데이터 플레인은 "실제 패킷을 전달"**합니다(CEF). 컨트롤 플레인의 결과물인 라우팅 테이블(RIB)을 데이터 플레인이 FIB로 변환해서 사용합니다.

- **꼬리 질문 대비 설명:**
  - **RIB(Routing Information Base)** = 컨트롤 플레인의 결과물(= 라우팅 테이블, `show ip route`)
  - **FIB(Forwarding Information Base)** = 데이터 플레인이 실제 조회하는 고속 검색 테이블(`show ip cef`)
  - 이 분리 덕분에 라우팅 프로토콜이 재계산 중에도 기존 트래픽 포워딩은 중단 없이 유지됩니다. (**NSF - Non-Stop Forwarding**의 근간)

### Q2. 라우터 CPU가 갑자기 폭주할 때 가장 먼저 의심해야 할 것은?

- 👉 **핵심 답변:** **CEF 미스(Miss)로 인한 Process Switching Punt** 현상을 최우선으로 의심해야 합니다. 정상 상황에서는 데이터 플레인이 CPU를 거의 쓰지 않기 때문입니다.

- **꼬리 질문 대비 설명:**
  - CEF가 처리하지 못하는 트래픽(IP Options, TTL=1, Fragmentation 필요, ACL 로깅 등)은 CPU로 올라와 Process Switching으로 처리됨 → 이를 **Punt**라고 함
  - `show ip cef switching statistics feature` 로 Punt 원인별 카운터 확인이 가능하며, 이게 실무 CPU 트러블슈팅의 시작점

### 3대 스위칭 방식 비교

| 항목 | Process Switching | Fast Switching | **CEF** ✅ |
|---|---|---|---|
| CPU 개입 | 모든 패킷 | 첫 패킷만 | 거의 없음 |
| 경로 조회 | 매번 RIB 조회 | 캐시 기반(첫 패킷 예외) | FIB 사전 계산 |
| 캐시 생성 | 없음 | On-demand (요청 시) | Proactive (미리 생성) |
| 속도 | 매우 느림 | 중간 | **라인 레이트** |
| 확장성 | 낮음 | 중간 | 매우 높음 |
| 현재 실무 | 디버깅용만 | Deprecated | **표준 (기본값)** |


### CEF 아키텍처: FIB & Adjacency Table

[Routing Table (RIB)] ← 컨트롤 플레인 (OSPF, BGP 등) ↓ (복사/변환) [FIB] ← "어디로 보낼지" (목적지 → 넥스트홉/IF) ↓ (참조) [Adjacency Table] ← "어떻게 보낼지" (L2 헤더, MAC 등) ↓ [실제 패킷 포워딩]


- **FIB:** 라우팅 테이블에서 목적지 관련 정보만 추출한 **고속 검색 테이블**. RIB이 변경되면 자동 동기화됨.
- **Adjacency Table:** 각 넥스트 홉의 **L2 정보(MAC, 프레임 헤더)를 미리 계산**해두어 ARP 지연 없이 즉시 전송 가능.

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. CEF 표준 설정 (실무 기본값 - 유지 권장)

```cisco
! ============================================================
! [CEF Baseline] - 모든 Cisco 라우터의 실무 표준
!   ※ 기본적으로 활성화되어 있으며, 절대 끄지 말 것
! ============================================================

configure terminal
 ip cef                                     ! 전역 CEF 활성화 (기본 ON)
!
interface GigabitEthernet0/0
 ip route-cache cef                         ! IF 레벨 CEF 활성화 (기본 ON)


---

```
### 2-2. Process Switching 강제 ( 트러블슈팅 전용)
```cisco
! ============================================================
! [Process Switching 강제] - debug ip packet 사용 시에만
!   ※ 운영망 절대 금지, CPU 폭주 위험
! ============================================================

configure terminal
 no ip cef                                  ! CEF 전역 비활성화
!
interface Serial1/0
 no ip route-cache                          ! Fast Switching까지 해제 → Process Switching 강제

! ▼ 디버깅 완료 후 반드시 복구할 것
interface Serial1/0
 ip route-cache cef
exit
ip cef
```
### 2-3. Fast Switching ( Deprecated - 참고용)
```cisco
! ============================================================
! [Fast Switching] - 레거시 장비 호환 시에만 사용
!   ※ 현대 IOS에서는 CEF로 완전 대체됨
! ============================================================

configure terminal
 no ip cef                                  ! CEF OFF
!
interface Serial1/0
 ip route-cache                             ! Fast Switching 활성화 (CEF 없을 때)
```
## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어
```cisco
! ---------- CEF 상태 확인 ----------
show ip cef summary                         ! CEF 전역 활성 여부 + FIB 엔트리 수
show ip cef                                 ! FIB 전체 조회
show ip cef <목적지 IP>                     ! 특정 목적지의 FIB 엔트리 (넥스트홉/IF)
show ip cef <IP> detail                     ! 상세 정보 (Recursive Lookup 포함)
show cef interface <IF>                     ! IF별 CEF 상태

! ---------- Adjacency Table 확인 ----------
show adjacency                              ! Adjacency Table 전체
show adjacency <IF>                         ! IF별 Adjacency
show adjacency detail                       ! L2 헤더 재작성 정보 (프레임 바이트 포함)

! ---------- CPU/Punt 진단 (⭐ 실무 핵심) ----------
show processes cpu sorted | exclude 0.0     ! CPU 사용률 프로세스별 정렬
show ip cef switching statistics feature    ! Punt 원인별 카운터
show ip traffic                             ! IP Options, TTL Expired 등 특수 패킷 통계
```
### 명령어 출력 해석 예시

**`show ip cef` 출력 해석:**

```cisco
Router# show ip cef
FastEthernet0/1, version 4, epoch 1
10.10.10.0/24, epoch 2, recursive
  via 192.168.1.2, FastEthernet0/1, adjacency 123456
       ↑ 넥스트홉        ↑ 출력 IF        ↑ Adjacency 참조 ID
```
**`show adjacency` 출력 해석:**

```cisco
Router# show adjacency FastEthernet0/1
FastEthernet0/1
  192.168.1.2, 001a.2b3c.4d5f, valid
   ↑ 넥스트홉    ↑ 넥스트홉 MAC        ↑ 상태 (valid/incomplete)
```
- **valid:** L2 정보가 완전히 채워져 즉시 전송 가능
- **incomplete:** ARP 미해결 상태 → 패킷은 CPU로 Punt되어 ARP 재시도
- ### 🚨 3-2. 대표 장애 시나리오 & 트러블슈팅

#### 시나리오 1: 라우터 CPU 90% 폭주 (IP Input 프로세스 점유)

**증상:** 트래픽은 흐르는데 CPU가 지속적으로 높음. `show processes cpu`에서 **`IP Input`**이 상위.

**원인 분석 및 조치 단계:**

1. 1. **CEF 상태 최우선 확인**
    
    ```cisco
    show ip cef summary
    show cef interface <문제 IF>
    ```
    
    - `CEF is disabled`가 보이면 즉시 `ip cef` 재활성화
    - 누군가 실수로 `no ip cef` 걸었을 가능성 → configuration 이력 확인
2. **Punt 원인 식별**
    
    ```cisco
    show ip cef switching statistics feature
    ```
    
    - 어떤 Feature가 CEF를 우회시키는지 확인 (ACL 로깅, NAT, IP Options 등)
    - 특히 **`ACL with log` 옵션**이 흔한 원인 → `log` 키워드 제거 검토
3. **불필요한 CPU 부하 제거**
    
    ```cisco
    interface <문제 IF>
     no ip unreachables                      ! ICMP Unreachable 생성 중단
     no ip redirects                         ! ICMP Redirect 생성 중단
     no ip proxy-arp                         ! Proxy ARP 비활성화
    ```
#### 시나리오 2: 특정 목적지만 통신이 안 됨 (RIB에는 있는데 FIB에 없음)

**증상:** `show ip route`에는 경로가 있는데 실제 통신은 실패.

**원인 분석 및 조치 단계:**

1. **RIB vs FIB 일치성 검증**
    
    ```cisco
    show ip route <목적지>
    show ip cef <목적지>
    ```
    
    - 두 결과가 다르면 CEF 동기화 문제
2. **Adjacency 상태 확인**
    
    ```cisco
    show adjacency <넥스트홉> detail
    ```
    
    - `incomplete` 상태면 ARP 미해결 → `show arp` 로 ARP 테이블 확인
    - ARP가 없으면 L2 문제(케이블, VLAN, Trunk 설정) 확인
3. **CEF 테이블 강제 재구성** (⚠️ 순단 발생 가능, 유지보수 시간에만)
    
    ```cisco
    clear ip cef *
    clear adjacency
    ```
    

### 3-3. 실무 Quick-Check 체크리스트

|점검 항목|명령어|정상 기준|
|---|---|---|
|CEF 전역 활성|`show ip cef summary`|`CEF is enabled`|
|IF별 CEF 활성|`show cef interface <IF>`|`IP CEF switching enabled`|
|RIB ↔ FIB 일치|`show ip route` vs `show ip cef`|동일 넥스트홉|
|Adjacency 유효성|`show adjacency detail`|모든 엔트리 `valid`|
|CPU 상태|`show processes cpu`|IP Input < 5%|
|Punt 카운터|`show ip cef switching statistics`|증가 추세 없음|

---

## 실무 Tip

- **CEF는 절대 끄지 않는다.** 정말 필요한 순간은 `debug ip packet` 명령을 쓸 때뿐이며, 그마저도 특정 IF에 한정해서 잠깐 끄고 즉시 복구해야 한다.
- **CPU 튐 = CEF 문제** 공식을 기억하자. 라우팅 프로토콜(OSPF/EIGRP) 자체는 안정 상태에서 CPU를 거의 안 쓴다.
- **FIB와 RIB이 다르면 무조건 비정상.** 이 경우 `clear ip cef *` 로 강제 재동기화하되, 순단 리스크가 있으니 반드시 유지보수 시간에 수행한다.

---

> 관련: 1.  📖   라우팅(Routing) 기초 개념 · 10. ⚡ Frame-relay 명령어 퀵 레퍼런스 · 2. 📍 Static Route (정적 라우팅) · 4. 🧭 라우팅 기초 개념 통합 정리 · 5. 🚑 라우팅 기초 트러블슈팅 치트시트 · 6. ⚡ 라우팅 기초 명령어 퀵 레퍼런스 · 7. 📡 Frame-relay (WAN L2 Protocol) · 8. 🧩 Frame-relay 통합 정리 · 9. 🚑 Frame-relay 트러블슈팅 치트시트