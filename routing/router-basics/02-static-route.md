# 2. 📍 Static Route (정적 라우팅)

> **Tag:** #Network #Cisco
> **핵심 요약:** **한 줄 요약:** 관리자가 목적지 네트워크로 가는 경로를 라우팅 테이블에 수동으로 직접 박아넣는 방식. 소규모/변화 없는 망에서 CPU·대역폭 부담 없이 정확한 경로 제어가 필요할 때 사용한다.

---

## 1. 🎯 핵심 기술 개념 (Concept)

### 대규모 망 시대에도 Static Route가 필요한 이유

동적 라우팅이 만능이 아니기 때문에, **경로가 단 하나뿐인 Stub 구간, 보안상 라우팅 정보를 노출하면 안 되는 구간, 그리고 Default Route(0.0.0.0/0)** 는 실무에서 무조건 Static이 정답이다.

Static Route의 Administrative Distance는 `1`로, Connected(0) 다음으로 신뢰도가 높다. OSPF(110), EIGRP(90)보다 우선하므로 동적 라우팅 환경에서도 **특정 트래픽을 강제로 우회시키는 PBR 대용**으로 쓰인다. Floating Static은 AD를 200 등으로 높여 백업 경로로 활용할 수 있다. 또한 Hello/Update 패킷을 주고받지 않으므로 저사양 CPE 장비나 저대역폭 WAN 회선(예: 64K HDLC)에서 리소스 부담 없이 운용할 수 있다.

### Next-hop IP 방식과 Exit Interface 방식의 선택 기준

**Point-to-Point(HDLC, PPP) 구간은 Exit Interface, Multi-Access(Ethernet, Frame-Relay) 구간은 반드시 Next-hop IP**를 사용해야 한다. 잘못 쓰면 ARP 폭주나 라우팅 실패가 발생한다.

Ethernet 구간에서 Exit Interface만 지정하면 라우터가 목적지 대역의 **모든 IP에 대해 ARP Request를 브로드캐스트**하게 된다(Proxy ARP 의존). 브로드캐스트 도메인이 커질수록 ARP 테이블 폭증과 CPU 부하를 유발하므로 반드시 Next-hop IP를 함께 명시해야 한다. `show ip route` 출력에서 Next-hop 방식은 `[1/0] via x.x.x.x`로, Interface 방식은 `is directly connected`로 표시된다. Interface 방식은 재귀 조회(Recursive Lookup)를 생략해 조회 속도가 빠르다는 장점이 있어 P2P 구간에서 선호된다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 🖧 실습 Topology
```text
    192.168.12.0/24              192.168.23.0/24
```

R1 ─────────────────── R2 ─────────────────── R3  192.168.1.0/24                                192.168.2.0/24                                    192.168.3.0/24


### 📌 R1 설정 (Stub Router - Default Route 스타일 권장)

```cisco
! ===== 인터페이스 기본 설정 =====
interface FastEthernet0/0
 no shutdown
 ip address 192.168.1.254 255.255.255.0                 ! LAN GW (마지막 주소 사용 컨벤션)
!
interface Serial1/0
 no shutdown
 encapsulation hdlc                                     ! WAN 캡슐화 : HDLC
 bandwidth 64                                           ! 64K 회선 - 라우팅 metric 계산용
 ip address 192.168.12.1 255.255.255.0
!
! ===== Static Route (Next-hop 방식) =====
! R1은 Stub 구간 → 모든 원격 대역을 R2로만 보내면 되므로 Default Route 권장
ip route 0.0.0.0 0.0.0.0 192.168.12.2                   ! 실무 Best Practice : 3줄 → 1줄로 축약
!
! (참고) 개별 지정 방식 - 학습/시험용
! ip route 192.168.2.0  255.255.255.0 192.168.12.2
! ip route 192.168.3.0  255.255.255.0 192.168.12.2
! ip route 192.168.23.0 255.255.255.0 192.168.12.2
```
### 📌 R2 설정 (Transit Router - 양방향 명시 필수)
```cisco
interface FastEthernet0/0
 no shutdown
 ip address 192.168.2.254 255.255.255.0
!
interface Serial1/1
 no shutdown
 encapsulation hdlc
 bandwidth 64
 clock rate 64000                                       ! DCE 측에서 클럭 제공
 ip address 192.168.12.2 255.255.255.0
!
interface Serial1/0
 no shutdown
 encapsulation hdlc
 bandwidth 64
 clock rate 64000
 ip address 192.168.23.2 255.255.255.0
!
! ===== Static Route =====
ip route 192.168.1.0 255.255.255.0 192.168.12.1         ! R1 LAN 방향
ip route 192.168.3.0 255.255.255.0 192.168.23.3         ! R3 LAN 방향
```
### 📌 R3 설정 (Stub Router)
```cisco
interface FastEthernet0/0
 no shutdown
 ip address 192.168.3.254 255.255.255.0
!
interface Serial1/1
 no shutdown
 encapsulation hdlc
 bandwidth 64
 ip address 192.168.23.3 255.255.255.0
!
! ===== Static Route =====
ip route 0.0.0.0 0.0.0.0 192.168.23.2                   ! Stub 라우터 : Default Route로 축약
```
## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)
### ✅ 필수 검증 명령어
```cisco
show ip route                    ! 라우팅 테이블 전체 확인 (S = Static, C = Connected)
show ip route static             ! Static 경로만 필터링
show running-config | include ip route
show ip interface brief          ! 인터페이스 UP/UP 상태 확인 (Layer1/2)
show cdp neighbors               ! 물리 연결 상대방 라우터 식별
ping 192.168.3.254 source 192.168.1.254   ! 종단 간 통신 검증 (반드시 source 지정)
traceroute 192.168.3.254         ! Hop-by-hop 경로 추적
```
### 🚨 대표 장애 시나리오

#### 시나리오 ①. "Ping이 편도로만 되고 응답이 안 옵니다" (가장 흔한 신입 실수)

- **증상**: R1에서 R3 LAN(192.168.3.0/24)으로 ping을 쏘면 요청은 가는데 `Request timeout` 발생.
- **원인**: **Return Path(복귀 경로) 누락**. 대부분 목적지 라우터에 출발지 대역으로 돌아오는 Static Route를 안 넣어서 발생.
- **단계별 조치**:
    1. R3에서 `show ip route`로 `192.168.1.0/24` 경로 존재 여부 확인.
    2. 없다면 → `ip route 192.168.1.0 255.255.255.0 192.168.23.2` 추가.
    3. 중간 경유 라우터(R2)에도 양쪽 대역 경로가 모두 있는지 재확인.
    4. **교훈**: Static Route는 **양방향 설정이 원칙**이다. 한쪽만 넣으면 ICMP Echo Reply가 길을 잃는다.

#### 시나리오 ②. "Serial 인터페이스는 UP인데 Ping이 안 됩니다"

- **증상**: `show ip interface brief`에서 Serial1/0이 `up/up`이지만 상대 라우터로 통신 불가.
- **원인 후보 & 조치**:
    1. **Clock rate 누락** → DCE 측에서 `show controllers serial 1/0`으로 DCE/DTE 확인 후 DCE에 `clock rate 64000` 설정.
    2. **캡슐화 불일치** → 양쪽 다 `encapsulation hdlc` (또는 ppp)로 일치시키기. 한쪽만 PPP면 즉시 down.
    3. **Subnet Mask 오타** → `/24`끼리는 되지만 한쪽만 `/30`이면 서로 다른 서브넷이 되어 통신 불가.
    4. **ACL 차단** → `show access-lists` 및 인터페이스에 적용된 필터 확인.

#### 💡 실무 Tip - Floating Static Route
```cisco
ip route 192.168.3.0 255.255.255.0 192.168.12.2         ! Primary (AD=1, 기본값)
ip route 192.168.3.0 255.255.255.0 10.0.0.1 200         ! Backup (AD=200, 평소엔 비활성)
```
Primary 경로 다운 시 자동으로 Backup 경로가 라우팅 테이블에 삽입된다. WAN 이중화 설계의 기본.

---

> 관련: 1.  📖   라우팅(Routing) 기초 개념 · 10. ⚡ Frame-relay 명령어 퀵 레퍼런스 · 3. ⚡  라우터의 스위칭 방식 (CEF) · 4. 🧭 라우팅 기초 개념 통합 정리 · 5. 🚑 라우팅 기초 트러블슈팅 치트시트 · 6. ⚡ 라우팅 기초 명령어 퀵 레퍼런스 · 7. 📡 Frame-relay (WAN L2 Protocol) · 8. 🧩 Frame-relay 통합 정리 · 9. 🚑 Frame-relay 트러블슈팅 치트시트