# 📝 Network 종합 연습문제

> **Tag:** #Network #Cisco #연습문제 #IPv4 #Subnetting #StaticRoute #DynamicRouting #ACL #VLAN #Switch #DHCP #NAT #심화
>
> **목적:** 강의 내용을 스스로 떠올려 풀면서 손에 익히는 것. 답을 보기 전에 반드시 직접 실행해볼 것.
>
> **난이도:** 🟢 기초(Level 1) → 🔵 응용(Level 2) → 🔴 심화(Level 3, 실무형 종합)

---

## 📚 문제 구성 & 관련 문서

| PART | 주제 | 문항 | 관련 문서 |
| --- | --- | --- | --- |
| 1 | IPv4 & Subnetting & Wildcard | Q1~Q6 | 1. 🧮 IPv4 Addressing & Subnetting & VLSM & Summary & Wildcard Mask |
| 2 | Cisco 라우터 기초 (Mode · Interface · Password) | Q7~Q11 | 2. ⚙️ Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리 |
| 3 | Static Route & Floating Static | Q12~Q16 | 2. 📍 Static Route (정적 라우팅) |
| 4 | Dynamic Routing (RIP · EIGRP · OSPF) | Q17~Q24 | 1. ♟️ RIP · 2. 🐎 EIGRP · 3. 🌲 OSPF |
| 5 | Standard ACL & Extended ACL | Q25~Q31 | 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리 |
| 6 | Switch · VLAN · Trunk · Inter-VLAN | Q32~Q37 | 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN |
| 7 | DHCP & NAT/PAT | Q38~Q43 | 5-1. 💾 DHCP (동적 IP 할당) · 5-2. 🔀 NAT & PAT (주소 변환) |
| 8 | 🔴 실무 종합 심화 | Q44~Q48 | 전 범위 |

> 명령어 문법이 기억나지 않을 때는 각 챕터의 ⚡ 퀵 레퍼런스를, 오류가 났을 때는 🚑 트러블슈팅 치트시트를 먼저 확인한다.

---

## PART 1. IPv4 & Subnetting & Wildcard

> 📖 관련 문서: 1. 🧮 IPv4 Addressing & Subnetting & VLSM & Summary & Wildcard Mask

### 📌 핵심 개념 복습

```text
서브넷 계산 공식
  호스트 수(가용)    : 2^(32-prefix) - 2
  서브넷 블록 크기   : 256 - 서브넷 마스크의 해당 옥텟 값
  네트워크 주소      : 호스트 비트가 모두 0
  브로드캐스트 주소  : 호스트 비트가 모두 1
  첫 번째 가용 IP    : 네트워크 주소 + 1
  마지막 가용 IP     : 브로드캐스트 주소 - 1

Wildcard Mask 계산 : 255.255.255.255 - Subnet Mask
  예) /27 → Subnet Mask 255.255.255.224 → Wildcard 0.0.0.31

VLSM 설계 원칙
  큰 서브넷부터 할당 → 작은 서브넷 순서로 설계
  P2P 링크는 /30(가용 2개) 또는 /31(RFC 3021) 사용
```

---

### 🟢 기초 (Level 1)

**Q1.** `192.168.10.0/27` 네트워크에 대해 다음을 구하시오.

1. 서브넷 마스크 (점십진수 표기)
2. 가용 호스트 수
3. 브로드캐스트 주소
4. Wildcard Mask

> 💡 **힌트:** /27 = 255.255.255.224, 블록 크기 = 256 - 224 = 32. 브로드캐스트 = 다음 네트워크 주소 - 1.

> [!success]- ✅ 정답 보기
>
> ```text
> 1. 서브넷 마스크     : 255.255.255.224
> 2. 가용 호스트 수    : 2^5 - 2 = 30개
> 3. 브로드캐스트 주소 : 192.168.10.31
> 4. Wildcard Mask     : 0.0.0.31
>
> 계산 과정:
>   블록 크기 = 256 - 224 = 32
>   네트워크  : 192.168.10.0
>   브로드캐스트: 192.168.10.0 + 32 - 1 = 192.168.10.31
>   가용 범위  : 192.168.10.1 ~ 192.168.10.30
> ```

---

**Q2.** `172.16.0.0/24`를 4개의 같은 크기 서브넷으로 나누시오. 각 서브넷의 네트워크 주소, 브로드캐스트 주소, 가용 호스트 수를 적으시오.

> 💡 **힌트:** 4개 서브넷 = 2비트 추가 → /26. 블록 크기 = 64.

> [!success]- ✅ 정답 보기
>
> ```text
> Prefix : /26   Subnet Mask : 255.255.255.192   가용 호스트 : 62개
>
> 서브넷 1 : 172.16.0.0/26    (브로드캐스트 172.16.0.63)
> 서브넷 2 : 172.16.0.64/26   (브로드캐스트 172.16.0.127)
> 서브넷 3 : 172.16.0.128/26  (브로드캐스트 172.16.0.191)
> 서브넷 4 : 172.16.0.192/26  (브로드캐스트 172.16.0.255)
> ```

---

**Q3.** IP `10.10.5.200`이 속한 네트워크가 `/29`라면 해당 서브넷의 네트워크 주소와 브로드캐스트 주소를 구하시오.

> 💡 **힌트:** 블록 크기 = 256 - 248 = 8. 200 ÷ 8 = 25...0 → 네트워크 = 10.10.5.200 (200은 8의 배수).

> [!success]- ✅ 정답 보기
>
> ```text
> 블록 크기  : 8 (256 - 248)
> 네트워크   : 10.10.5.200  (200 = 25 × 8, 8의 배수)
> 브로드캐스트: 10.10.5.207
> 가용 범위  : 10.10.5.201 ~ 10.10.5.206  (6개)
> ```

---

### 🔵 응용 (Level 2)

**Q4.** 다음 VLSM 요구사항에 맞게 `192.168.1.0/24`를 분배하시오.

| 세그먼트 | 필요 호스트 수 |
| --- | --- |
| LAN-A | 60개 |
| LAN-B | 28개 |
| LAN-C | 12개 |
| P2P 링크 | 2개 |

> 💡 **힌트:** VLSM은 큰 것부터 할당. 60개 → /26(62개), 28개 → /27(30개), 12개 → /28(14개), 2개 → /30(2개).

> [!success]- ✅ 정답 보기
>
> ```text
> LAN-A (60개) → /26  : 192.168.1.0/26    (가용 .1 ~ .62,   BC .63)
> LAN-B (28개) → /27  : 192.168.1.64/27   (가용 .65 ~ .94,  BC .95)
> LAN-C (12개) → /28  : 192.168.1.96/28   (가용 .97 ~ .110, BC .111)
> P2P   (2개)  → /30  : 192.168.1.112/30  (가용 .113 ~ .114, BC .115)
>
> 남은 주소 공간 : 192.168.1.116 ~ 192.168.1.255 (향후 확장용)
> ```

---

**Q5.** `172.16.0.0/24` ~ `172.16.7.0/24` 8개 서브넷을 하나의 Summary 경로로 요약하시오. Summary 주소와 Wildcard Mask를 적으시오.

> 💡 **힌트:** 8개 = 2³ → 3비트 추가 → /24에서 3비트 올림 = /21. 세 번째 옥텟 0~7 범위 = 0.0.7.255 (Wildcard).

> [!success]- ✅ 정답 보기
>
> ```text
> Summary 주소     : 172.16.0.0/21
> 서브넷 마스크    : 255.255.248.0
> Wildcard Mask    : 0.0.7.255
>
> OSPF 광고 예시:
>   area 0 range 172.16.0.0 255.255.248.0
>
> EIGRP 광고 예시:
>   summary-address 172.16.0.0 255.255.248.0
> ```

---

**Q6.** Wildcard Mask `0.0.0.254`를 ACL에 사용할 때 어떤 IP 주소와 매칭되는지 설명하고, `192.168.1.0 0.0.0.254`가 매칭하는 IP 예시 3개를 적으시오.

> 💡 **힌트:** 0=비교(고정), 1=무시(Don't Care). `0.0.0.254`의 2진수 = 11111110 → 마지막 비트만 무시 = 짝수 IP만 매칭.

> [!success]- ✅ 정답 보기
>
> ```text
> 0.0.0.254 = 1111 1110 → 마지막 1비트 무시
> → 마지막 옥텟의 짝수(0, 2, 4, ... 254)만 매칭
>
> 192.168.1.0  0.0.0.254 가 매칭하는 IP 예시:
>   - 192.168.1.0   (마지막 비트 0 → 무시)
>   - 192.168.1.2
>   - 192.168.1.100
>
> → 홀수 IP (192.168.1.1, 192.168.1.3 등) 는 매칭하지 않는다.
> ```

---

## PART 2. Cisco 라우터 기초 (Mode · Interface · Password)

> 📖 관련 문서: 2. ⚙️ Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리

### 📌 핵심 개념 복습

```text
모드 계층
  Router>          : User EXEC (show 일부만 가능)
  Router#          : Privileged EXEC (enable 로 진입, 모든 show 가능)
  Router(config)#  : Global Configuration (conf t)
  Router(config-if)#  : Interface 설정
  Router(config-line)# : Line(VTY/Console) 설정

Password 종류
  enable secret 암호   : Privileged EXEC 진입 암호 (MD5 암호화)
  line console 0 / password : 콘솔 접속 암호
  line vty 0 4 / password  : 텔넷/SSH 원격 접속 암호
  service password-encryption : 평문 암호를 Type 7 로 인코딩

기본 보안 설정 명령어
  no ip domain-lookup           : 오탈자 시 DNS 조회 방지
  exec-timeout 분 초            : 세션 자동 종료 시간
  logging synchronous           : 로그 메시지가 입력 중인 줄을 가리지 않도록
```

---

### 🟢 기초 (Level 1)

**Q7.** 라우터에서 다음 작업을 수행하는 명령어를 순서대로 작성하시오.

1. Privileged EXEC 모드로 진입
2. Global Configuration 모드로 진입
3. 호스트명을 `R1`으로 변경
4. `enable secret cisco123` 설정
5. Privileged EXEC 모드로 돌아가기

> 💡 **힌트:** `enable` → `configure terminal` → `hostname` → `enable secret` → `end` 또는 `Ctrl+Z`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> Router> enable
> Router# configure terminal
> Router(config)# hostname R1
> R1(config)# enable secret cisco123
> R1(config)# end
> R1#
> ```

---

**Q8.** `GigabitEthernet0/0` 인터페이스에 IP 주소 `192.168.1.1/24`를 설정하고 활성화하시오. 설정 후 확인 명령어도 작성하시오.

> 💡 **힌트:** `interface` → `ip address` → `no shutdown`. 확인은 `show ip interface brief`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# ip address 192.168.1.1 255.255.255.0
> R1(config-if)# no shutdown
> R1(config-if)# end
>
> ! 확인
> R1# show ip interface brief
> R1# show interface GigabitEthernet0/0
> ```

---

**Q9.** VTY 0~4 라인에 텔넷 접속 암호 `admin1234`를 설정하고, 로그인을 요구하도록 하시오.

> 💡 **힌트:** `line vty 0 4` → `password` → `login`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# line vty 0 4
> R1(config-line)# password admin1234
> R1(config-line)# login
> R1(config-line)# exec-timeout 5 0
> R1(config-line)# exit
> ```

---

### 🔵 응용 (Level 2)

**Q10.** 다음 보안 기본 설정을 완성하시오.

- 콘솔 암호 `console123` 설정 및 로그인 요구
- 모든 평문 암호를 암호화
- 오탈자 DNS 조회 방지
- 콘솔 세션 비활동 10분 후 자동 종료
- `logging synchronous` 설정

> 💡 **힌트:** `service password-encryption` 은 Global config, `no ip domain-lookup` 도 Global config.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# no ip domain-lookup
> R1(config)# service password-encryption
>
> R1(config)# line console 0
> R1(config-line)# password console123
> R1(config-line)# login
> R1(config-line)# exec-timeout 10 0
> R1(config-line)# logging synchronous
> R1(config-line)# exit
> ```
>
> > ⚠️ `service password-encryption` 은 Type 7 인코딩(약한 암호화)이므로 반드시 `enable secret` 과 병행해야 한다.

---

**Q11.** `show` 명령어를 사용하여 다음 정보를 확인하는 명령어를 각각 작성하시오.

1. 인터페이스 상태와 IP 주소 요약
2. 라우팅 테이블 전체
3. CDP 인접 장비 목록 (상세)
4. 현재 실행 중인 설정 (RAM)
5. NVRAM에 저장된 설정

> 💡 **힌트:** `show ip interface brief`, `show ip route`, `show cdp neighbors detail`, `show running-config`, `show startup-config`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1# show ip interface brief
> R1# show ip route
> R1# show cdp neighbors detail
> R1# show running-config
> R1# show startup-config
>
> ! 설정을 NVRAM에 저장하는 명령어 (참고)
> R1# copy running-config startup-config
> ! 또는 축약형
> R1# wr
> ```

---

## PART 3. Static Route & Floating Static

> 📖 관련 문서: 2. 📍 Static Route (정적 라우팅)

### 📌 핵심 개념 복습

```text
Static Route 기본 문법
  ip route [목적지 네트워크] [서브넷 마스크] [Next-hop IP | Exit Interface] [AD값]

Default Route (모든 목적지)
  ip route 0.0.0.0 0.0.0.0 [Next-hop IP]

Floating Static (백업 경로)
  ip route ... [높은 AD값]   ← 기본 경로보다 AD가 높아야 라우팅 테이블에서 숨겨짐

AD 비교
  Connected : 0    Static : 1    EIGRP : 90
  OSPF      : 110  RIP    : 120

Next-hop vs Exit Interface
  P2P 링크(HDLC/PPP) → Exit Interface 사용 가능
  Multi-Access(Ethernet) → 반드시 Next-hop IP 사용 (ARP 폭주 방지)
```

---

### 🟢 기초 (Level 1)

**Q12.** R1에서 `192.168.3.0/24` 네트워크로 가는 정적 경로를 설정하시오. Next-hop은 `10.0.12.2`이다.

> 💡 **힌트:** `ip route 목적지 마스크 next-hop`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# ip route 192.168.3.0 255.255.255.0 10.0.12.2
>
> ! 확인
> R1# show ip route static
> R1# show ip route 192.168.3.0
> ```

---

**Q13.** R1이 Stub 라우터(단일 출구)일 때 Default Route를 설정하고, 설정이 라우팅 테이블에 반영되었는지 확인하는 명령어를 작성하시오. Next-hop은 `10.0.12.2`.

> 💡 **힌트:** `0.0.0.0 0.0.0.0` = 모든 목적지. `show ip route` 에서 `S*` 로 표시된다.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2
>
> ! 확인
> R1# show ip route
> ! 출력에서 S* 0.0.0.0/0 [1/0] via 10.0.12.2 확인
> ```

---

### 🔵 응용 (Level 2)

**Q14.** 다음 토폴로지에서 R1, R2, R3 각각에 Static Route를 설정해 전체 통신이 가능하게 하시오.

```text
R1 ── (S1/0: 10.0.12.0/30) ── R2 ── (S1/1: 10.0.23.0/30) ── R3
│                                                               │
Fa0/0: 192.168.1.0/24                              Fa0/0: 192.168.3.0/24
```

> 💡 **힌트:** R1은 Stub이므로 Default Route 권장. R2는 Transit이므로 양방향 경로가 필요.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! ===== R1 (Stub) =====
> R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2            ! Default Route
>
> ! ===== R2 (Transit) =====
> R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.12.1   ! R1 LAN 방향
> R2(config)# ip route 192.168.3.0 255.255.255.0 10.0.23.2   ! R3 LAN 방향
>
> ! ===== R3 (Stub) =====
> R3(config)# ip route 0.0.0.0 0.0.0.0 10.0.23.1            ! Default Route
>
> ! 확인
> R1# ping 192.168.3.1 source 192.168.1.1
> R1# traceroute 192.168.3.1
> ```

---

**Q15.** R1에서 `192.168.3.0/24`로의 기본 경로는 `10.0.12.2`(OSPF), 백업 경로는 `10.0.13.2`(Serial)로 Floating Static을 설정하시오.

> 💡 **힌트:** Floating Static의 AD는 기본 경로 프로토콜보다 **높아야** 라우팅 테이블에서 숨겨진다. OSPF AD = 110 이므로 150 이상 설정.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! OSPF가 주 경로를 광고하는 상태라면 Static이 테이블에서 숨겨져야 함
> ! Floating Static AD = 150 (OSPF 110보다 높게)
> R1(config)# ip route 192.168.3.0 255.255.255.0 10.0.13.2 150
>
> ! 확인 (OSPF 경로가 있을 때는 Floating Static이 보이지 않음)
> R1# show ip route 192.168.3.0
> ! OSPF 경로 삭제 시 Floating Static이 자동으로 활성화됨
> ```

---

**Q16.** Static Route 설정에서 Next-hop IP 방식과 Exit Interface 방식의 차이를 설명하고, Ethernet 인터페이스에서 Exit Interface만 사용하면 안 되는 이유를 서술하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> Next-hop IP 방식
>   ip route 192.168.3.0 255.255.255.0 10.0.12.2
>   → show ip route 에서 [1/0] via 10.0.12.2 표시
>   → 재귀 조회(Recursive Lookup) 발생
>
> Exit Interface 방식
>   ip route 192.168.3.0 255.255.255.0 Serial1/0
>   → show ip route 에서 directly connected 표시
>   → 재귀 조회 없음 (P2P에서 빠름)
>
> Ethernet에서 Exit Interface만 사용 시 문제점
>   → 라우터가 목적지 대역 모든 IP에 대해 ARP Request 브로드캐스트 발생
>   → Proxy ARP 의존 → ARP 테이블 폭증 + CPU 과부하
>   → 따라서 Ethernet 구간은 반드시 Next-hop IP 방식 사용
> ```

---

## PART 4. Dynamic Routing (RIP · EIGRP · OSPF)

> 📖 관련 문서: 1. ♟️ RIP · 2. 🐎 EIGRP · 3. 🌲 OSPF

### 📌 핵심 개념 복습

```text
프로토콜 비교
  구분       RIP v2         EIGRP              OSPF
  타입       Distance Vector Hybrid (DUAL)     Link-State (Dijkstra)
  AD         120            90 (내부) / 170    110
  Metric     Hop Count(최대15) BW·Delay 등 복합  Cost (= 10^8 / BW)
  Hello      30초           5초(LAN) / 60초    10초(LAN) / 30초(WAN)
  Dead       180초          3×Hello (15s/180s) 4×Hello (40s/120s)
  Update     30초 주기      트리거 방식         트리거 방식
  Auto-summary  기본 ON     기본 ON (no auto 권장) 없음

OSPF 핵심
  Router-ID  : Loopback > Active Physical IF (높은 IP 우선) > 수동 지정 권장
  DR/BDR     : Broadcast 네트워크에서 선출 (Priority 높을수록 유리, 0=선출 제외)
  LSA 범위   : Area 내부만 공유 (ABR이 Area 간 요약 가능)
```

---

### 🟢 기초 (Level 1)

**Q17.** R1에서 RIPv2를 설정하시오. 광고할 네트워크는 `192.168.1.0`과 `10.0.12.0`이며, Auto-summary를 비활성화하시오.

> 💡 **힌트:** `router rip` → `version 2` → `no auto-summary` → `network` (Classful 주소로 입력).

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# router rip
> R1(config-router)# version 2
> R1(config-router)# no auto-summary
> R1(config-router)# network 192.168.1.0
> R1(config-router)# network 10.0.0.0         ! Classful 주소 입력 (10.0.12.0 → 10.0.0.0)
> R1(config-router)# end
>
> ! 확인
> R1# show ip protocols
> R1# show ip route rip
> ```

---

**Q18.** R1에서 EIGRP AS 100을 설정하시오. 광고할 네트워크는 `192.168.1.0/24`와 `10.0.12.0/30`이다.

> 💡 **힌트:** `router eigrp 100` → `no auto-summary` → `network 주소 wildcard`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# router eigrp 100
> R1(config-router)# no auto-summary
> R1(config-router)# network 192.168.1.0 0.0.0.255
> R1(config-router)# network 10.0.12.0 0.0.0.3
> R1(config-router)# end
>
> ! 확인
> R1# show ip eigrp neighbors
> R1# show ip route eigrp
> R1# show ip eigrp topology
> ```

---

**Q19.** R1에서 OSPF Process 1을 설정하시오. Router-ID는 `1.1.1.1`로 수동 지정하고, `192.168.1.0/24`는 Area 1에, `10.0.12.0/30`은 Area 0에 광고하시오.

> 💡 **힌트:** `router ospf 1` → `router-id` → `network 주소 wildcard area 번호`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# router ospf 1
> R1(config-router)# router-id 1.1.1.1
> R1(config-router)# network 192.168.1.0 0.0.0.255 area 1
> R1(config-router)# network 10.0.12.0 0.0.0.3 area 0
> R1(config-router)# end
>
> ! 확인
> R1# show ip ospf neighbor
> R1# show ip route ospf
> R1# show ip ospf database
> R1# show ip ospf interface brief
> ```

---

### 🔵 응용 (Level 2)

**Q20.** OSPF에서 DR/BDR 선출 기준을 설명하고, R1이 무조건 DR이 되도록 Priority를 설정하시오. 인터페이스는 `GigabitEthernet0/0`.

> 💡 **힌트:** Priority 기본값 = 1. 높을수록 유리. 0으로 설정하면 선출 제외.

> [!success]- ✅ 정답 보기
>
> ```text
> DR/BDR 선출 기준 (순서대로 적용):
>   1. OSPF Priority 높은 쪽 (기본값 1, 범위 0~255)
>   2. Priority 동일 시 → Router-ID 높은 쪽
>   3. Priority = 0 → DR/BDR 선출에서 제외 (DROther 고정)
>
> 주의: DR/BDR은 재선출되지 않는다 (Non-preemptive).
>       Priority 변경 후에는 ospf process clear 가 필요.
> ```
>
> ```cisco
> ! R1을 DR로 고정
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# ip ospf priority 255
>
> ! 경쟁 라우터들의 Priority를 0으로 설정하면 더 확실
> R2(config-if)# ip ospf priority 0
> R3(config-if)# ip ospf priority 0
>
> ! OSPF 재시작 (현재 설정된 프로세스 초기화)
> R1# clear ip ospf process
> ```

---

**Q21.** 다음 OSPF 상태(State)의 의미와 정상 수렴 시 거쳐야 하는 순서를 설명하시오.

`Down → Init → 2-Way → ExStart → Exchange → Loading → Full`

> [!success]- ✅ 정답 보기
>
> ```text
> Down      : Hello 수신 없음 (인접 관계 없음)
> Init      : Hello 수신했지만 내 Router-ID가 상대 Hello에 없음
> 2-Way     : 서로의 Hello에 상대 RID 포함 → 인접 관계 형성
>             (DR/BDR가 아닌 DROther 끼리는 여기서 멈춤)
> ExStart   : 마스터/슬레이브 결정 (DBD 교환 준비)
> Exchange  : DBD(Database Description) 교환 → LSDB 목록 비교
> Loading   : LSR/LSU/LSAck 교환으로 빠진 LSA 채움
> Full      : LSDB 동기화 완료 → 정상 수렴 상태
>
> 정상 인접 라우터는 Full 상태여야 하며,
> show ip ospf neighbor 에서 Full/DR, Full/BDR, Full/DROTHER 로 확인.
> ```

---

**Q22.** EIGRP에서 Passive Interface를 설정하는 이유와 방법을 설명하시오. LAN 인터페이스 `GigabitEthernet0/0`에 Passive를 적용하시오.

> 💡 **힌트:** Passive Interface = Hello를 보내지 않음 = 네이버를 맺지 않음. 단 network 명령으로 해당 네트워크는 라우팅 테이블에 광고된다.

> [!success]- ✅ 정답 보기
>
> ```text
> Passive Interface 설정 이유:
>   LAN 쪽(사용자 PC가 있는 세그먼트)으로는 라우팅 프로토콜 Hello를
>   보낼 필요가 없고, 불필요한 트래픽과 보안 위협(가짜 라우터 접속 시도)을
>   방지하기 위해 설정한다.
>   네트워크는 광고하되 Hello는 차단하는 것이 목적이다.
> ```
>
> ```cisco
> ! EIGRP
> R1(config)# router eigrp 100
> R1(config-router)# passive-interface GigabitEthernet0/0
> ! 또는 기본값을 passive로 하고 WAN 인터페이스만 활성화
> R1(config-router)# passive-interface default
> R1(config-router)# no passive-interface Serial1/0
>
> ! OSPF 동일 방법
> R1(config)# router ospf 1
> R1(config-router)# passive-interface GigabitEthernet0/0
> ```

---

**Q23.** `show ip protocols` 출력에서 확인할 수 있는 정보를 4가지 이상 나열하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> show ip protocols 에서 확인 가능한 정보:
>   1. 실행 중인 라우팅 프로토콜 종류 (RIP, EIGRP, OSPF 등)
>   2. 광고 중인 네트워크 목록
>   3. 라우팅 업데이트 타이머 (Update/Invalid/Holddown/Flush)
>   4. 이웃 라우터(Neighbor) 정보
>   5. AD(Administrative Distance) 값
>   6. EIGRP: AS 번호, K 값 (메트릭 계산 가중치)
>   7. OSPF: Process ID, Router-ID, 참여 Area
>   8. Passive Interface 목록
> ```

---

**Q24.** 🔴 심화 — R1(OSPF)과 R2(EIGRP) 경계에서 Redistribution(재분배)을 설정할 때 반드시 지정해야 하는 파라미터와 그 이유를 OSPF → EIGRP 방향을 기준으로 설명하시오.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! OSPF → EIGRP 재분배 (R2에서 설정)
> R2(config)# router eigrp 100
> R2(config-router)# redistribute ospf 1 metric 10000 100 255 1 1500
> !                                        BW   Delay Rel Load MTU
>
> ! EIGRP → OSPF 재분배 (R2에서 설정)
> R2(config)# router ospf 1
> R2(config-router)# redistribute eigrp 100 subnets
> !                                          ↑ 반드시 필요 (없으면 /24 단위로만 광고)
> ```
>
> ```text
> OSPF → EIGRP 필수 파라미터 (metric):
>   EIGRP는 메트릭 계산을 위해 5개 값이 필요하다:
>   BW(대역폭 Kbps), Delay(10μs 단위), Reliability(255=100%),
>   Load(1=최소), MTU(바이트)
>   → 지정 안 하면 메트릭 = infinity → 라우팅 테이블에 설치 안 됨
>
> EIGRP → OSPF 필수 파라미터 (subnets):
>   subnets 키워드 없으면 Classful 경계(주요 네트워크 주소)만 재분배 → VLSM 경로 유실
>   실무에서 subnets 생략은 가장 흔한 실수 중 하나
>
> 재분배 시 주의: Routing Loop 방지를 위해 반드시 Route-map 또는 distribute-list로
>   재분배 범위를 제한해야 한다 (양방향 재분배 환경에서 특히 중요).
> ```

---

## PART 5. Standard ACL & Extended ACL

> 📖 관련 문서: 3-1-1. 🔰  ACL 기초 - Standard ACL (Numbered & Named) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리

### 📌 핵심 개념 복습

```text
ACL 3대 원칙
  1. 위에서 아래로 순차 매칭 → 구체적 조건을 먼저, 넓은 조건을 나중에
  2. 마지막에 암묵적 "deny any" 자동 추가 → permit 없으면 전부 차단
  3. Numbered ACL은 라인 단위 수정 불가 → 운영망에서는 Named ACL 필수

적용 위치 원칙
  Standard ACL  → 목적지 인터페이스에 가깝게 (out 방향 권장)
  Extended ACL  → 출발지 인터페이스에 가깝게 (in 방향 권장)

Standard ACL 번호  : 1~99, 1300~1999
Extended ACL 번호  : 100~199, 2000~2699

Extended ACL 문법
  access-list [번호] [permit|deny] [프로토콜] [출발지 IP WC] [목적지 IP WC] [포트조건]
  포트 조건: eq(=), neq(≠), gt(>), lt(<), range(범위)
```

---

### 🟢 기초 (Level 1)

**Q25.** Standard ACL을 사용해 `192.168.2.0/24` 네트워크의 접근은 허용하고 나머지는 차단하는 ACL을 작성하시오. (ACL 번호: 10, 인터페이스 Fa0/0 out 방향 적용)

> 💡 **힌트:** Standard ACL은 출발지 IP만 필터링. 암묵적 deny any가 있으므로 permit 후 끝내면 된다.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# access-list 10 permit 192.168.2.0 0.0.0.255
> ! 나머지는 암묵적 deny any 로 자동 차단
>
> R1(config)# interface FastEthernet0/0
> R1(config-if)# ip access-group 10 out
>
> ! 확인
> R1# show access-lists
> R1# show ip interface FastEthernet0/0 | include access list
> ```

---

**Q26.** ACL에 `permit any`를 마지막에 명시하는 경우와 생략하는 경우의 차이를 설명하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> permit any 생략 시:
>   암묵적 "deny any" 가 ACL 마지막에 자동 추가된다.
>   → 명시적으로 permit한 트래픽 외 모든 것이 차단된다.
>   → 일부만 허용(Whitelist)하는 정책에 적합.
>
> permit any 명시 시:
>   마지막 줄에 permit any 를 추가하면 특정 트래픽만 차단하고 나머지는 허용된다.
>   → 일부만 차단(Blacklist)하는 정책에 적합.
>
> 예시:
>   Blacklist 방식 (192.168.1.0/24 만 차단, 나머지 허용):
>     access-list 2 deny 192.168.1.0 0.0.0.255
>     access-list 2 permit any       ← 반드시 필요
> ```

---

**Q27.** Named Standard ACL `BLOCK_PC`를 작성하시오. `10.0.0.100` IP만 차단하고 나머지는 허용한다. 인터페이스 `GigabitEthernet0/1` out 방향에 적용하시오.

> 💡 **힌트:** Named ACL은 `ip access-list standard 이름` 으로 생성. 특정 Host는 `host` 키워드 또는 Wildcard `0.0.0.0` 사용.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# ip access-list standard BLOCK_PC
> R1(config-std-nacl)# deny host 10.0.0.100
> R1(config-std-nacl)# permit any
> R1(config-std-nacl)# exit
>
> R1(config)# interface GigabitEthernet0/1
> R1(config-if)# ip access-group BLOCK_PC out
>
> ! 확인
> R1# show ip access-lists BLOCK_PC
> ```

---

### 🔵 응용 (Level 2)

**Q28.** Extended ACL을 사용해 다음 두 조건을 하나의 ACL(번호: 110)에 작성하시오.

1. `192.168.1.0/24` → `192.168.3.0/24` 방향 HTTP(80)만 허용
2. `192.168.2.0/24` → `192.168.3.0/24` 방향 HTTPS(443) 허용
3. 나머지 전부 차단

인터페이스 `Serial1/0` in 방향에 적용하시오.

> 💡 **힌트:** Extended ACL = `access-list 번호 permit tcp 출발지 WC 목적지 WC eq 포트`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# access-list 110 permit tcp 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255 eq 80
> R1(config)# access-list 110 permit tcp 192.168.2.0 0.0.0.255 192.168.3.0 0.0.0.255 eq 443
> ! 암묵적 deny any any 로 나머지 전부 차단
>
> R1(config)# interface Serial1/0
> R1(config-if)# ip access-group 110 in
>
> ! 확인
> R1# show access-lists 110
> ```

---

**Q29.** Named Extended ACL `ACL_VTY_SSH`를 작성해 관리 PC `172.16.0.10`에서만 SSH 접속(TCP 22)을 허용하고, 나머지 VTY 접속은 차단하시오. VTY 라인에 적용하시오.

> 💡 **힌트:** VTY 라인에 ACL 적용은 `access-class` 명령을 사용한다 (인터페이스의 `ip access-group` 과 다름).

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# ip access-list standard ACL_VTY_SSH
> R1(config-std-nacl)# permit host 172.16.0.10
> R1(config-std-nacl)# exit
>
> R1(config)# line vty 0 4
> R1(config-line)# access-class ACL_VTY_SSH in
> R1(config-line)# transport input ssh
> R1(config-line)# login local
>
> ! 확인
> R1# show line vty 0 4
> R1# show ip access-lists ACL_VTY_SSH
> ```

---

**Q30.** ACL 적용 후 트래픽이 전혀 허용되지 않는 문제가 발생했다. 점검해야 할 사항 4가지를 서술하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> ACL 트러블슈팅 체크리스트:
>
> 1. ACL 적용 방향 확인 (in/out)
>    → show ip interface 인터페이스 | include access list
>    → 방향이 반대로 적용되어 있으면 반대로 재적용
>
> 2. permit any 누락 여부 확인
>    → Blacklist 방식이라면 마지막 줄에 permit any 가 있어야 함
>    → show access-lists 로 각 라인 히트 카운트 확인
>
> 3. ACL 조건 순서 확인
>    → 더 구체적인 조건이 위에 있어야 함
>    → 넓은 permit이 먼저 있으면 구체적 deny 가 무시됨
>
> 4. 올바른 인터페이스에 적용되었는지 확인
>    → Standard ACL은 목적지 근처, Extended ACL은 출발지 근처
>    → 반대로 걸면 의도치 않은 트래픽이 차단됨
>
> 5. 추가: permit 라인의 Wildcard Mask 오류 확인
>    → 예) /24 네트워크에 0.0.0.0 (host 조건) 을 쓰는 실수
> ```

---

**Q31.** 🔴 심화 — Reflective ACL과 일반 Extended ACL의 차이를 설명하고, Reflective ACL의 동작 원리를 서술하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> 일반 Extended ACL
>   → 정적 규칙: 관리자가 허용/차단 조건을 수동으로 명시
>   → 외부 → 내부 방향 허용 규칙은 항상 열려 있어 세션 없어도 접속 가능
>   → 스테이트리스(Stateless) 방식
>
> Reflective ACL (반사 ACL)
>   → 스테이트풀(Stateful) 방식 = 세션 추적
>   → 내부 → 외부 트래픽이 나갈 때 임시 permit 규칙을 자동 생성
>   → 해당 세션의 응답 트래픽(외부 → 내부)만 통과 허용
>   → 세션 종료 시 임시 규칙 자동 삭제
>
> 설정 핵심:
>   ip access-list extended OUT_ACL
>     permit tcp any any reflect MY_SESSIONS
>   !
>   ip access-list extended IN_ACL
>     evaluate MY_SESSIONS      ← 반사된 임시 규칙 적용
>     deny ip any any
>
> 장점: 시작되지 않은 외부 접속 차단 → 방화벽과 유사한 보안 효과
> 단점: UDP/ICMP는 세션 개념이 없어 타임아웃 기반으로 관리
> ```

---

## PART 6. Switch · VLAN · Trunk · Inter-VLAN

> 📖 관련 문서: 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN

### 📌 핵심 개념 복습

```text
VLAN 기초
  VLAN     : 논리적 브로드캐스트 도메인 분리 (1~4094, 1=기본, 1002~1005=예약)
  Access   : PC 연결 포트 (단일 VLAN 소속)
  Trunk    : 스위치-스위치, 스위치-라우터 연결 (여러 VLAN 태그 전달, 802.1Q)
  Native VLAN : Trunk에서 태그 없이 전달되는 VLAN (기본 VLAN 1)

Inter-VLAN 라우팅 방식
  Router-on-a-Stick : 라우터 서브인터페이스 (Sub-interface) 방식
  L3 Switch         : SVI(Switched Virtual Interface) + ip routing 활성화

핵심 명령어
  vlan 10                           : VLAN 생성
  name SALES                        : VLAN 이름 지정
  switchport mode access            : Access 포트 설정
  switchport access vlan 10         : VLAN 할당
  switchport mode trunk             : Trunk 포트 설정
  switchport trunk allowed vlan 10,20,30  : 허용 VLAN 지정
  show vlan brief                   : VLAN 목록 확인
  show interfaces trunk             : Trunk 상태 확인
```

---

### 🟢 기초 (Level 1)

**Q32.** 스위치에서 VLAN 10(SALES)과 VLAN 20(HR)을 생성하고, `Fa0/1`은 VLAN 10, `Fa0/2`는 VLAN 20 Access 포트로 설정하시오.

> 💡 **힌트:** `vlan database` 또는 `conf t` 에서 `vlan 번호` 로 생성. VLAN DB에서 생성한 게 더 권장됨.

> [!success]- ✅ 정답 보기
>
> ```cisco
> SW1(config)# vlan 10
> SW1(config-vlan)# name SALES
> SW1(config-vlan)# exit
> SW1(config)# vlan 20
> SW1(config-vlan)# name HR
> SW1(config-vlan)# exit
>
> SW1(config)# interface FastEthernet0/1
> SW1(config-if)# switchport mode access
> SW1(config-if)# switchport access vlan 10
> SW1(config-if)# exit
>
> SW1(config)# interface FastEthernet0/2
> SW1(config-if)# switchport mode access
> SW1(config-if)# switchport access vlan 20
> SW1(config-if)# exit
>
> ! 확인
> SW1# show vlan brief
> ```

---

**Q33.** SW1과 SW2 사이 링크 `Fa0/24`를 Trunk로 설정하고, VLAN 10, 20, 30만 통과하도록 허용하시오. Native VLAN을 99로 변경하시오.

> 💡 **힌트:** `switchport mode trunk` → `switchport trunk allowed vlan 10,20,30` → `switchport trunk native vlan 99`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> SW1(config)# interface FastEthernet0/24
> SW1(config-if)# switchport mode trunk
> SW1(config-if)# switchport trunk allowed vlan 10,20,30
> SW1(config-if)# switchport trunk native vlan 99
> SW1(config-if)# exit
>
> ! SW2에도 동일하게 설정 필요
>
> ! 확인
> SW1# show interfaces trunk
> SW1# show interfaces Fa0/24 trunk
> ```

---

### 🔵 응용 (Level 2)

**Q34.** Router-on-a-Stick 방식으로 Inter-VLAN 라우팅을 설정하시오.

- 라우터 인터페이스 `Fa0/0` 에 서브인터페이스 생성
- VLAN 10: `192.168.10.1/24`, VLAN 20: `192.168.20.1/24`
- 스위치 연결 포트는 Trunk, Native VLAN 1 유지

> 💡 **힌트:** 서브인터페이스 `Fa0/0.10` → `encapsulation dot1Q 10` → `ip address`. 물리 인터페이스에는 IP 없이 `no shutdown` 만.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! ===== 라우터 설정 =====
> R1(config)# interface FastEthernet0/0
> R1(config-if)# no shutdown                 ! 물리 IF 활성화 (IP 없음)
> R1(config-if)# exit
>
> R1(config)# interface FastEthernet0/0.10
> R1(config-subif)# encapsulation dot1Q 10
> R1(config-subif)# ip address 192.168.10.1 255.255.255.0
> R1(config-subif)# exit
>
> R1(config)# interface FastEthernet0/0.20
> R1(config-subif)# encapsulation dot1Q 20
> R1(config-subif)# ip address 192.168.20.1 255.255.255.0
> R1(config-subif)# exit
>
> ! ===== 스위치 설정 =====
> SW1(config)# interface FastEthernet0/24     ! 라우터 연결 포트
> SW1(config-if)# switchport mode trunk
>
> ! 확인
> R1# show ip interface brief
> R1# show ip route
> ```

---

**Q35.** STP(Spanning Tree Protocol)에서 Root Bridge 선출 기준과, SW1이 무조건 Root Bridge가 되도록 하는 두 가지 방법을 설명하시오.

> 💡 **힌트:** BID = Priority(16bit) + MAC 주소. 낮은 BID가 Root Bridge.

> [!success]- ✅ 정답 보기
>
> ```text
> Root Bridge 선출 기준:
>   1. Bridge Priority 가 낮은 스위치 (기본값 32768)
>   2. Priority 동일 시 MAC 주소가 낮은 스위치
>
> SW1을 Root Bridge로 강제하는 방법:
>
> 방법 1: Priority 수동 설정 (4096의 배수만 허용)
>   SW1(config)# spanning-tree vlan 1 priority 0
>   ! 0이 최솟값 → Root Bridge 보장
>
> 방법 2: 매크로 명령어 사용
>   SW1(config)# spanning-tree vlan 1 root primary
>   ! 현재 Root Bridge Priority - 4096 으로 자동 설정 (최소 24576)
>
>   SW1(config)# spanning-tree vlan 1 root secondary
>   ! Backup Root (Priority 28672) 설정
> ```

---

**Q36.** VLAN 설계에서 VLAN 1을 관리 VLAN으로 사용하지 말아야 하는 이유를 서술하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> VLAN 1 관리 VLAN 사용 금지 이유:
>
> 1. 보안 취약점
>    - VLAN 1은 모든 스위치에서 기본 Native VLAN
>    - VLAN Hopping 공격에 노출 (이중 태그 공격 등)
>    - 공격자가 VLAN 1 트래픽을 쉽게 스니핑 가능
>
> 2. 광범위한 브로드캐스트 도메인
>    - VLAN 1은 모든 포트가 기본적으로 소속
>    - 관리 트래픽이 전체 브로드캐스트 도메인에 퍼짐
>
> 3. 네트워크 장애 전파
>    - STP, CDP, VTP 등 제어 트래픽도 VLAN 1 사용
>    - 관리 트래픽 혼재로 네트워크 가시성 저하
>
> Best Practice:
>   - 관리 VLAN은 별도 번호(예: VLAN 99) 지정
>   - VLAN 1은 사용하지 않거나 Trunk에서 제외
>   - Native VLAN을 사용하지 않는 번호로 변경
> ```

---

**Q37.** 🔴 심화 — EtherChannel(LACP)로 SW1과 SW2 사이 `Fa0/23`, `Fa0/24` 두 포트를 묶어 Port-Channel 1을 구성하시오. Port-Channel은 Trunk로 설정하시오.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! ===== SW1 설정 =====
> SW1(config)# interface range FastEthernet0/23-24
> SW1(config-if-range)# channel-group 1 mode active    ! LACP 활성 모드
> SW1(config-if-range)# exit
>
> SW1(config)# interface Port-channel 1
> SW1(config-if)# switchport mode trunk
> SW1(config-if)# switchport trunk allowed vlan 10,20,30
>
> ! ===== SW2 설정 =====
> SW2(config)# interface range FastEthernet0/23-24
> SW2(config-if-range)# channel-group 1 mode active
> SW2(config-if-range)# exit
>
> SW2(config)# interface Port-channel 1
> SW2(config-if)# switchport mode trunk
>
> ! 확인
> SW1# show etherchannel summary
> SW1# show interfaces port-channel 1
> ```
>
> ```text
> LACP 모드 조합 (두 스위치 간):
>   active - active   : 정상 동작 ✅
>   active - passive  : 정상 동작 ✅
>   passive - passive : 협상 안 됨 ❌
>
> PAgP 모드 (Cisco 전용):
>   desirable - desirable : 정상 ✅
>   desirable - auto      : 정상 ✅
>   auto - auto           : 협상 안 됨 ❌
> ```

---

## PART 7. DHCP & NAT/PAT

> 📖 관련 문서: 5-1. 💾 DHCP (동적 IP 할당) · 5-2. 🔀 NAT & PAT (주소 변환)

### 📌 핵심 개념 복습

```text
DHCP 핵심 과정 (DORA)
  Discover → Offer → Request → Acknowledge

DHCP 서버 설정 핵심
  ip dhcp pool 이름                  : 풀 생성
  network 주소 마스크                 : 할당 대역
  default-router IP                  : 게이트웨이
  dns-server IP                      : DNS
  lease 일 시간 분                   : 임대 시간 (기본 1일)
  ip dhcp excluded-address 시작 끝   : 제외 주소 (GW/서버 등)

NAT/PAT 핵심
  Static NAT  : 1:1 고정 변환 (서버 공개용)
  Dynamic NAT : 공인 IP Pool → 다수 내부 IP (1:1, Pool 소진 시 차단)
  PAT(Overload): 공인 IP 1개 + 포트 번호로 다수 변환 (가장 많이 사용)

ip nat inside  : 내부 인터페이스 지정
ip nat outside : 외부 인터페이스 지정
```

---

### 🟢 기초 (Level 1)

**Q38.** R1에서 DHCP 서버를 설정하시오.

- Pool 이름: OFFICE
- 할당 대역: `192.168.1.0/24`
- 게이트웨이: `192.168.1.1`
- DNS: `8.8.8.8`
- 제외 주소: `192.168.1.1` ~ `192.168.1.10` (서버/GW용)

> 💡 **힌트:** `ip dhcp excluded-address` 는 Pool 설정 전에 먼저 선언하는 것이 일반적.

> [!success]- ✅ 정답 보기
>
> ```cisco
> R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
>
> R1(config)# ip dhcp pool OFFICE
> R1(dhcp-config)# network 192.168.1.0 255.255.255.0
> R1(dhcp-config)# default-router 192.168.1.1
> R1(dhcp-config)# dns-server 8.8.8.8
> R1(dhcp-config)# lease 1
> R1(dhcp-config)# exit
>
> ! 확인
> R1# show ip dhcp pool
> R1# show ip dhcp binding
> R1# show ip dhcp conflict
> ```

---

**Q39.** PAT(Overload)를 설정하시오.

- 내부 네트워크: `192.168.1.0/24`
- 내부 인터페이스: `GigabitEthernet0/0`
- 외부 인터페이스: `Serial1/0` (공인 IP 할당됨)
- ACL 번호: 1

> 💡 **힌트:** ACL로 NAT 대상 내부 주소 정의 → `ip nat inside source list ACL int 외부IF overload`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! 1. NAT 대상 정의
> R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
>
> ! 2. PAT(Overload) 설정
> R1(config)# ip nat inside source list 1 interface Serial1/0 overload
>
> ! 3. 인터페이스 방향 지정
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# ip nat inside
> R1(config-if)# exit
>
> R1(config)# interface Serial1/0
> R1(config-if)# ip nat outside
> R1(config-if)# exit
>
> ! 확인
> R1# show ip nat translations
> R1# show ip nat statistics
> ```

---

### 🔵 응용 (Level 2)

**Q40.** Static NAT를 사용해 내부 서버 `192.168.1.100`을 공인 IP `203.0.113.10`으로 1:1 매핑하시오.

> 💡 **힌트:** `ip nat inside source static 내부IP 공인IP`.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! Static NAT 매핑
> R1(config)# ip nat inside source static 192.168.1.100 203.0.113.10
>
> ! 인터페이스 방향 지정 (이전에 설정 안 된 경우)
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# ip nat inside
> R1(config-if)# exit
>
> R1(config)# interface Serial1/0
> R1(config-if)# ip nat outside
>
> ! 확인
> R1# show ip nat translations
> ! Pro  Inside global      Inside local       Outside local    Outside global
> ! ---  203.0.113.10       192.168.1.100      ---              ---
> ```

---

**Q41.** DHCP Relay Agent(ip helper-address)를 설정해야 하는 상황과 방법을 설명하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> 필요한 상황:
>   DHCP 서버와 클라이언트가 서로 다른 서브넷(VLAN)에 있을 때
>   DHCP Discover는 브로드캐스트이므로 라우터를 넘어가지 못한다
>   → 중간 라우터에서 브로드캐스트를 유니캐스트로 변환해 DHCP 서버로 전달
>
> 설정 위치:
>   클라이언트가 속한 네트워크의 게이트웨이 인터페이스(라우터/L3 스위치)
> ```
>
> ```cisco
> ! DHCP 서버: 10.0.0.1 / 클라이언트: 192.168.1.0/24 세그먼트
> ! 게이트웨이 인터페이스 GigabitEthernet0/0 에 설정
>
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# ip helper-address 10.0.0.1
>
> ! 확인
> R1# show ip interface GigabitEthernet0/0 | include Helper
> ```

---

**Q42.** NAT 변환 테이블이 비어있어 내부 → 외부 통신이 안 되는 문제를 트러블슈팅하는 순서를 서술하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> NAT 트러블슈팅 순서:
>
> 1. 인터페이스 방향 확인
>    R1# show ip interface GigabitEthernet0/0 | include NAT
>    R1# show ip interface Serial1/0 | include NAT
>    → inside/outside 가 올바른 인터페이스에 적용되었는지 확인
>
> 2. ACL 매칭 확인 (PAT의 경우)
>    R1# show access-lists 1
>    → permit 조건이 내부 주소와 일치하는지, 히트 카운트 확인
>
> 3. NAT 통계 확인
>    R1# show ip nat statistics
>    → Hits 카운트가 증가하는지 확인
>
> 4. 패킷 추적 (Debug)
>    R1# debug ip nat
>    → 실시간 NAT 변환 로그 확인 후 반드시 undebug all 로 종료
>
> 5. 라우팅 확인
>    R1# show ip route
>    → 외부 인터페이스로 나가는 Default Route 가 있는지 확인
> ```

---

**Q43.** 🔴 심화 — 내부 사용자들이 PAT로 인터넷을 사용하면서, 동시에 DMZ 서버(`192.168.100.10`)를 `203.0.113.10:80`으로 외부에 공개하는 설정을 함께 작성하시오.

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! === PAT 설정 (내부 사용자) ===
> R1(config)# access-list 10 permit 192.168.1.0 0.0.0.255
> R1(config)# ip nat inside source list 10 interface Serial1/0 overload
>
> ! === Static NAT (DMZ 서버 공개) ===
> ! TCP 포트 80으로 들어오는 요청을 DMZ 서버로 전달
> R1(config)# ip nat inside source static tcp 192.168.100.10 80 203.0.113.10 80
>
> ! === 인터페이스 방향 ===
> R1(config)# interface GigabitEthernet0/0        ! LAN
> R1(config-if)# ip nat inside
> R1(config)# interface GigabitEthernet0/1        ! DMZ
> R1(config-if)# ip nat inside
> R1(config)# interface Serial1/0                 ! WAN (공인 IP 203.0.113.10)
> R1(config-if)# ip nat outside
>
> ! 확인
> R1# show ip nat translations
> ! TCP 203.0.113.10:80  192.168.100.10:80  (Static 항목)
> ! TCP 203.0.113.10:xxxx 192.168.1.x:xxxx  (PAT 항목)
> ```

---

## PART 8. 🔴 실무 종합 심화

> 📖 관련 문서: 전 범위

> 이 파트는 여러 개념을 결합하는 종합 시나리오 문제이다. 실제 네트워크 설계처럼 접근해볼 것.

---

**Q44. 소규모 기업 네트워크 설계 문제**

다음 요구사항을 모두 만족하는 IP 주소 설계와 라우터 설정을 작성하시오.

**요구사항:**
- 주어진 주소 공간: `10.10.0.0/22`
- 영업팀(SALES): 호스트 120개 필요 → VLAN 10
- 개발팀(DEV): 호스트 60개 필요 → VLAN 20
- 관리팀(MGMT): 호스트 28개 필요 → VLAN 30
- R1 ↔ R2 WAN 링크: 호스트 2개 → /30

**설계 내용:**
1. VLSM으로 각 세그먼트에 IP 대역 할당 (서브넷 낭비 최소화)
2. 각 VLAN의 게이트웨이 IP (Router-on-a-Stick 방식)
3. R1의 Inter-VLAN 서브인터페이스 설정 명령어

> [!success]- ✅ 정답 보기
>
> ```text
> === IP 설계 (VLSM, 큰 것부터 할당) ===
>
> 10.10.0.0/22 = 10.10.0.0 ~ 10.10.3.255 (1022개 가용)
>
> SALES(120개) → /25 (126개 가용):
>   네트워크: 10.10.0.0/25
>   GW      : 10.10.0.1
>   범위    : 10.10.0.2 ~ 10.10.0.126
>   BC      : 10.10.0.127
>
> DEV(60개) → /26 (62개 가용):
>   네트워크: 10.10.0.128/26
>   GW      : 10.10.0.129
>   범위    : 10.10.0.130 ~ 10.10.0.190
>   BC      : 10.10.0.191
>
> MGMT(28개) → /27 (30개 가용):
>   네트워크: 10.10.0.192/27
>   GW      : 10.10.0.193
>   범위    : 10.10.0.194 ~ 10.10.0.222
>   BC      : 10.10.0.223
>
> WAN P2P → /30 (2개 가용):
>   네트워크: 10.10.0.224/30
>   R1      : 10.10.0.225
>   R2      : 10.10.0.226
>   BC      : 10.10.0.227
> ```
>
> ```cisco
> === R1 Router-on-a-Stick 설정 ===
>
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# no shutdown
>
> R1(config)# interface GigabitEthernet0/0.10
> R1(config-subif)# encapsulation dot1Q 10
> R1(config-subif)# ip address 10.10.0.1 255.255.255.128
>
> R1(config)# interface GigabitEthernet0/0.20
> R1(config-subif)# encapsulation dot1Q 20
> R1(config-subif)# ip address 10.10.0.129 255.255.255.192
>
> R1(config)# interface GigabitEthernet0/0.30
> R1(config-subif)# encapsulation dot1Q 30
> R1(config-subif)# ip address 10.10.0.193 255.255.255.224
> ```

---

**Q45. ACL + NAT 복합 설계 문제**

다음 보안 정책을 ACL과 NAT로 구현하시오.

**환경:**
- 내부 LAN: `192.168.10.0/24` (내부 인터페이스: Gi0/0)
- DMZ: `192.168.100.0/24` (DMZ 인터페이스: Gi0/1)
- 외부: `Serial0/0` (공인 IP: `203.0.113.1`)

**보안 정책:**
1. 내부 사용자는 인터넷 접속 가능 (PAT)
2. 외부에서 DMZ 웹서버 `192.168.100.10`의 HTTP(80), HTTPS(443) 접속만 허용
3. 외부에서 내부 LAN 직접 접속 전면 차단
4. 내부 → DMZ는 모든 트래픽 허용
5. DMZ → 내부 LAN 접속은 전면 차단

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! === 1. PAT (내부 → 인터넷) ===
> R1(config)# access-list 10 permit 192.168.10.0 0.0.0.255
> R1(config)# ip nat inside source list 10 interface Serial0/0 overload
>
> ! === 2. Static NAT (DMZ 서버 공개) ===
> R1(config)# ip nat inside source static tcp 192.168.100.10 80 203.0.113.1 80
> R1(config)# ip nat inside source static tcp 192.168.100.10 443 203.0.113.1 443
>
> ! === 3~5. ACL 정책 ===
> ! 외부 → 내부/DMZ 트래픽 필터링 (Serial0/0 in 방향)
> R1(config)# ip access-list extended ACL_WAN_IN
> R1(config-ext-nacl)# permit tcp any host 203.0.113.1 eq 80
> R1(config-ext-nacl)# permit tcp any host 203.0.113.1 eq 443
> R1(config-ext-nacl)# deny ip any any                ! 내부 직접 접속 차단
>
> ! DMZ → LAN 트래픽 차단 (Gi0/1 out 방향으로 LAN 인터페이스 적용)
> R1(config)# ip access-list extended ACL_DMZ_TO_LAN
> R1(config-ext-nacl)# deny ip 192.168.100.0 0.0.0.255 192.168.10.0 0.0.0.255
> R1(config-ext-nacl)# permit ip any any
>
> ! === 인터페이스 적용 ===
> R1(config)# interface GigabitEthernet0/0
> R1(config-if)# ip nat inside
>
> R1(config)# interface GigabitEthernet0/1
> R1(config-if)# ip nat inside
> R1(config-if)# ip access-group ACL_DMZ_TO_LAN out   ! DMZ→LAN 차단
>
> R1(config)# interface Serial0/0
> R1(config-if)# ip nat outside
> R1(config-if)# ip access-group ACL_WAN_IN in         ! 외부 접속 제한
> ```

---

**Q46. OSPF Multi-Area 설계 문제**

3개 Area를 가진 OSPF 네트워크에서 ABR(Area Border Router) 역할의 R2를 설정하시오.

**토폴로지:**
```cisco
Area 1            Area 0 (Backbone)       Area 2
R1 (1.1.1.1) ── R2 (2.2.2.2) ABR ── R3 (3.3.3.3)

R1 LAN: 192.168.1.0/24   Backbone: 10.0.0.0/30   R3 LAN: 192.168.3.0/24
```

**요구사항:**
1. R2 OSPF 설정 (Process 1)
2. Area 1 → Area 0 방향으로 Summary Route `192.168.1.0/24`를 ABR에서 요약해 광고

> [!success]- ✅ 정답 보기
>
> ```cisco
> ! ===== R2 (ABR) OSPF 설정 =====
> R2(config)# router ospf 1
> R2(config-router)# router-id 2.2.2.2
>
> ! Backbone(Area 0) 인터페이스
> R2(config-router)# network 10.0.0.0 0.0.0.3 area 0
>
> ! Area 1 방향 인터페이스 (R1 연결)
> R2(config-router)# network 172.16.12.0 0.0.0.3 area 1
>
> ! Area 1 → Area 0 방향으로 Summary 광고 (ABR 기능)
> R2(config-router)# area 1 range 192.168.1.0 255.255.255.0
>
> ! 확인
> R2# show ip ospf
> R2# show ip ospf border-routers
> R2# show ip route ospf
> ```
>
> ```text
> area range 명령어 동작:
>   ABR에서 Area 내부 특정 서브넷들을 묶어 하나의 요약 경로로 광고
>   → Backbone으로 나가는 LSA 수 감소 → 라우팅 테이블 경량화
>   → 개별 링크 Flapping 이 Summary 범위 밖으로 전파되지 않아 안정성 향상
>
> ABR vs ASBR:
>   ABR  : OSPF Area 경계 라우터 (Inter-Area 경로 요약)
>   ASBR : 외부 라우팅 프로토콜과 OSPF 간 재분배 담당
> ```

---

**Q47. 네트워크 장애 시나리오 분석**

다음 `show ip route` 출력을 보고 문제를 분석하고 해결책을 제시하시오.

```text
R1# show ip route
...
O    192.168.2.0/24 [110/2] via 10.0.12.2, 00:05:23, Serial1/0
O    192.168.3.0/24 [110/3] via 10.0.12.2, 00:05:23, Serial1/0
S    192.168.3.0/24 [1/0] via 10.0.13.2
```

**질문:** `192.168.3.0/24`로 가는 트래픽은 어느 경로를 사용하는가? 그 이유와 Static Route를 Floating Static으로 만들어야 하는 이유를 설명하시오.

> [!success]- ✅ 정답 보기
>
> ```text
> 현재 경로 선택:
>   S 192.168.3.0/24 [1/0] via 10.0.13.2 → Static Route가 사용됨
>
> 이유:
>   AD(Administrative Distance) 비교
>   - Static Route : AD = 1
>   - OSPF         : AD = 110
>   라우터는 AD가 낮은(신뢰도 높은) 경로를 선택 → Static이 우선
>
> 문제점:
>   OSPF가 더 나은 경로(토폴로지 기반)를 계산하고 있어도
>   Static이 항상 이를 덮어버린다.
>   Static 경로가 끊겨도 OSPF 경로로 자동 전환 안 됨 (Static이 항상 이기므로)
>
> 해결책 (Floating Static):
>   ip route 192.168.3.0 255.255.255.0 10.0.13.2 150
>   → AD를 150으로 올려 OSPF(110)보다 높게 설정
>   → 평상시에는 OSPF 경로 사용, OSPF 경로 장애 시에만 Static 활성화
>   → 자동 Failover 가능한 백업 경로 역할
> ```

---

**Q48. 🔴 최종 종합 — 네트워크 설계 체크리스트**

실무 소규모 기업 네트워크 구축 시 반드시 확인해야 할 항목을 카테고리별로 정리하시오.

> [!success]- ✅ 정답 보기
>
> **[ IP 설계 ]**
>
> - □ VLSM으로 IP 낭비 최소화 설계 완료
> - □ P2P 링크에 /30 (또는 /31) 적용
> - □ 연속 대역 확보로 Route Summary 가능한 구조
> - □ Loopback 인터페이스로 Router-ID 고정
>
> **[ 라우팅 ]**
>
> - □ 토폴로지에 맞는 라우팅 프로토콜 선택 (소규모: Static/OSPF, 대규모: EIGRP/OSPF)
> - □ LAN 인터페이스 Passive-interface 적용
> - □ Floating Static으로 백업 경로 구성
> - □ Default Route 전파 확인
>
> **[ 보안 ]**
>
> - □ enable secret 설정 (enable password 지양)
> - □ service password-encryption 적용
> - □ VTY 접속 제한 (ACL_VTY + transport input ssh)
> - □ no ip domain-lookup 적용
> - □ Banner 경고 메시지 설정
>
> **[ 스위치 ]**
>
> - □ VLAN 1 관리 VLAN 사용 금지, 전용 관리 VLAN 지정
> - □ 사용하지 않는 포트 shutdown + VLAN 999 등으로 격리
> - □ Trunk Native VLAN 변경 (VLAN Hopping 방지)
> - □ Root Bridge 수동 고정 (STP Priority)
> - □ Port Security 적용 (MAC 제한)
>
> **[ NAT/DHCP ]**
>
> - □ PAT(Overload) 설정 및 인터페이스 inside/outside 방향 확인
> - □ DHCP excluded-address 로 GW/서버 IP 제외
> - □ ip helper-address 로 DHCP Relay 설정 (필요시)
>
> **[ 최종 확인 명령어 ]**
>
> ```cisco
> show ip interface brief          ! 인터페이스 상태
> show ip route                    ! 라우팅 테이블
> show ip protocols                ! 라우팅 프로토콜 상태
> show vlan brief                  ! VLAN 목록
> show interfaces trunk            ! Trunk 상태
> show ip nat translations         ! NAT 변환 테이블
> show access-lists                ! ACL 히트 카운트
> copy running-config startup-config
> ```

---
