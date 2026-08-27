# 라우팅(Routing) 기초 개념

> **Tag:** #Network #Cisco #Routing #StaticRoute #CEF
> **핵심 요약:** **문서 목적:** IP 네트워크의 핵심 기능인 라우팅의 3대 기능(Discovery/Selection/Management)과 정적/동적 라우팅의 차이를 실무 아키텍처 관점에서 정리한다.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. 실무에서 정적 라우팅(Static)과 동적 라우팅(Dynamic)을 나누는 기준은 무엇인가요?

- 👉 **핵심 답변:** **네트워크 규모와 변경 빈도**가 기준입니다. 경로가 거의 변하지 않는 스텁(Stub) 구간·보안 경계는 Static, 링크가 많고 장애 대응이 필요한 백본/DC는 Dynamic이 원칙입니다.

- **꼬리 질문 대비 설명:**
  - Static은 **결정론적(Deterministic)**이라 예측 가능하고 CPU 부담이 없지만, 장애 시 자동 우회가 불가능해 관리자가 직접 개입해야 합니다. (예: 인터넷 회선 Default Route, 방화벽 뒤 서버망)
  - Dynamic은 **자동 수렴**이 강점이지만 프로토콜 자체가 CPU/메모리를 소모하고, 잘못 설계하면 루프·경로 flapping 같은 사고를 일으킵니다. 그래서 실무에서는 **Static + Dynamic 하이브리드**로 설계하는 경우가 많습니다.

### Q2. 라우팅의 3대 기능(Path Discovery/Selection/Management) 중 실무에서 가장 사고가 많이 나는 단계는 어디인가요?

- 👉 **핵심 답변:** **Path Management(경로 관리)** 단계입니다. Discovery/Selection은 초기 설계 시점의 문제인 반면, Management는 운영 중 토폴로지 변화에 실시간으로 대응해야 하므로 수렴 지연·루프 문제가 여기서 발생합니다.

- **꼬리 질문 대비 설명:**
  - Path Selection의 기준값(Metric)은 Management 단계에서 링크 상태 변화에 따라 계속 재계산되므로, **타이머 튜닝(Hello/Dead, SPF Throttle)**이 안정성의 핵심입니다.
  - 라우팅 테이블은 **최종 결과물**일 뿐이고, 실제 트러블슈팅은 프로토콜별 Neighbor Table / Topology Table을 함께 봐야 정확한 원인을 알 수 있습니다.

### 라우팅의 3대 기능 요약

| 기능 | 역할 | 실무 관점 핵심 |
|---|---|---|
| **Path Discovery** | 도달 가능한 모든 경로 수집 | Unicast/Multicast/Broadcast로 정보 교환 |
| **Path Selection** | 수집된 경로 중 최적 경로 선출 | **Metric** 계산이 핵심 (Cost, Bandwidth, Delay 등) |
| **Path Management** | 경로 변화 감지 및 재수렴 | 타이머·SPF·DUAL 등 프로토콜 고유 로직 |

### Static vs Dynamic Routing 비교

| 구분 | Static Routing | Dynamic Routing |
|---|---|---|
| 설정 방식 | 관리자 수동 등록 | 프로토콜이 자동 학습 |
| CPU/메모리 부담 | 거의 없음 | 프로토콜 오버헤드 존재 |
| 장애 대응 | 수동 개입 필요 | 자동 재수렴 |
| 예측 가능성 | 매우 높음 (결정론적) | 상대적으로 낮음 |
| 적합 환경 | Stub 망, 방화벽 뒤, DMZ | 백본, DC, 캠퍼스, WAN |

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. Static Route 표준 설정

```cisco
! ============================================================
! [Static Route Baseline] - Stub 구간 / Default Route 표준
! ============================================================

! ▼ Default Route (인터넷 방향 - 가장 흔한 실무 케이스)
ip route 0.0.0.0 0.0.0.0 203.0.113.1              ! 모든 미학습 트래픽을 ISP GW로

! ▼ 특정 서브넷 Static (넥스트 홉 IP 지정 방식 - 권장)
ip route 10.20.30.0 255.255.255.0 10.1.1.2        ! 목적지/마스크/넥스트홉

! ▼ Exit Interface 지정 방식 (P2P 링크에서만 사용, Ethernet 금지)
ip route 10.20.30.0 255.255.255.0 Serial0/0       ! Ethernet에 쓰면 ARP 폭주 위험

! ▼ Floating Static Route (백업 경로 - AD 값을 높여서 대기)
ip route 10.20.30.0 255.255.255.0 10.2.2.2 200    ! AD 200 → 주 경로 죽어야 활성화
```
### 2-2. 라우팅 테이블 확인 후 트래픽 흐름 검증
```cisco
! 라우팅 테이블 전체
show ip route

! 특정 목적지에 대해 어떤 경로가 선택되는지 (Longest Match 검증)
show ip route 10.20.30.5

! Static Route만 필터링
show ip route static

```
## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어
```cisco
show ip route                       ! 라우팅 테이블 (RIB) 전체
show ip route <목적지 IP>           ! 특정 목적지에 대한 Longest Match 결과
show ip route summary               ! 프로토콜별 경로 개수 요약
show ip protocols                   ! 구동 중인 라우팅 프로토콜 상태
traceroute <목적지 IP>              ! 실제 패킷이 타는 경로 검증
ping <목적지 IP> source <IF>        ! 소스 지정 Ping (비대칭 라우팅 확인)
```
### 3-2. 대표 장애 시나리오 & 트러블슈팅

#### 시나리오 1: Static Route를 걸었는데 통신이 안 됨

**증상:** `show ip route`에 경로는 보이는데 Ping이 안 되는 상황.

**원인 분석 및 조치 단계:**

1. **넥스트 홉 도달성 확인**
    
    - `ping <넥스트홉 IP>`로 넥스트 홉 자체가 살아있는지 확인
    - 넥스트 홉이 죽어있으면 Static Route는 라우팅 테이블에서 자동 제거됨
2. **역방향 경로(Return Path) 확인**
    
    - 상대편 라우터에도 **응답용 Return Route**가 있는지 확인
    - 실무 사고의 80%가 "가는 길만 만들고 오는 길을 안 만들어서" 발생
3. **Ethernet에 Exit Interface만 지정한 경우**
    
    - `ip route x.x.x.x y.y.y.y GigabitEthernet0/0` 형태는 Ethernet에서 위험
    - 모든 목적지에 대해 ARP를 날려서 CPU/메모리 폭주 유발 → 반드시 **넥스트 홉 IP 지정**으로 변경

#### 시나리오 2: Floating Static Route가 활성화되지 않음

**증상:** 주 경로가 죽었는데 백업 경로로 전환되지 않음.

**원인 분석 및 조치 단계:**

1. **AD 값 비교**
    
    - `show ip route <목적지>`로 현재 활성 경로의 AD 확인
    - Floating Static의 AD가 주 경로보다 높게 설정되었는지 검증
2. **주 경로 실제 다운 여부**
    
    - 인터페이스는 살아있지만 상대편이 죽은 "Silent Failure" 상황일 수 있음
    - **IP SLA + Track** 기능으로 실제 End-to-End 도달성 기반 전환 구성 필요

---

> 관련: 10. ⚡ Frame-relay 명령어 퀵 레퍼런스 · 2. 📍 Static Route (정적 라우팅) · 3. ⚡  라우터의 스위칭 방식 (CEF) · 4. 🧭 라우팅 기초 개념 통합 정리 · 5. 🚑 라우팅 기초 트러블슈팅 치트시트 · 6. ⚡ 라우팅 기초 명령어 퀵 레퍼런스 · 7. 📡 Frame-relay (WAN L2 Protocol) · 8. 🧩 Frame-relay 통합 정리 · 9. 🚑 Frame-relay 트러블슈팅 치트시트