# 1. 🧮 IPv4 Addressing & Subnetting & VLSM & Summary & Wildcard Mask

> **Tag:** #Network #Cisco #IPv4 #Subnetting #VLSM #Addressing
> **핵심 요약:** **Layer 3 (Network Layer) Protocol** — 서로 다른 브로드캐스트 도메인(외부 네트워크) 간 통신 시 사용되는 32bit 논리 주소 체계.
> Class 기반 → CIDR / VLSM 기반으로 진화하며 **주소 고갈 방지 + 라우팅 테이블 최적화**가 핵심 목적.

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Q1. VLSM(가변 서브넷)과 Route Summary(경로 요약)를 실무에서 반드시 함께 설계해야 하는 이유는?

- 👉 **핵심 답변:** VLSM은 **주소 낭비를 최소화**하고, Summary는 **라우팅 테이블 크기와 Flapping 전파 범위를 최소화**하기 때문에, 둘은 "IP 설계 → 라우팅 안정성"으로 이어지는 한 쌍의 설계 원칙이다.
- **꼬리 질문 대비 설명:**
  - VLSM 없이 Class 단위(Classful)로 설계하면 P2P 링크(Host 2개)에 /24를 통째로 낭비 → **IP 고갈**과 **연속 대역 확보 실패**로 이어져 나중에 Summary 자체가 불가능해진다.
  - Summary가 되지 않는 설계는 특정 서브넷의 인터페이스가 Up/Down을 반복(Flapping)할 때 **LSA/Update가 전체 도메인으로 전파** → CPU/메모리 스파이크, SPF 재계산 폭주로 이어진다. 그래서 IP 할당 단계부터 **"연속된 대역을 큰 조직/사이트 단위로 묶어서" 분배**하는 것이 원칙.

### Q2. Subnet Mask와 Wildcard Mask는 왜 굳이 표기 방식이 반대(0/1)인가?

- 👉 **핵심 답변:** Subnet Mask는 "네트워크 경계 식별(고정)"용이라 `1=Network`, Wildcard Mask는 "**Don't Care 비트를 지정**(범위 매칭)"용이라 `1=무시`로, 목적 자체가 다르기 때문이다.
- **꼬리 질문 대비 설명:**
  - Wildcard는 **ACL / OSPF network 구문 / EIGRP** 등 "**어떤 IP 범위를 매칭할지**" 판단하는 곳에만 쓴다. 연속되지 않은 비트도 매칭 가능(예: `0.0.0.254` = 짝수 IP만).
  - 실무 꿀팁: `Wildcard = 255.255.255.255 - Subnet Mask` 로 빠르게 계산 (예: /24 → `0.0.0.255`, /27 → `0.0.0.31`).

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 📋 주소 설계 참고표 (실무 암기 필수)

| Prefix | Subnet Mask | Wildcard | Host 수(가용) | 용도 |
|---|---|---|---|---|
| /24 | 255.255.255.0 | 0.0.0.255 | 254 | 사무실 LAN 표준 |
| /25 | 255.255.255.128 | 0.0.0.127 | 126 | 중형 세그먼트 |
| /26 | 255.255.255.192 | 0.0.0.63 | 62 | 부서 단위 |
| /27 | 255.255.255.224 | 0.0.0.31 | 30 | 소규모 세그먼트 |
| /28 | 255.255.255.240 | 0.0.0.15 | 14 | 서버팜 소규모 |
| /29 | 255.255.255.248 | 0.0.0.7 | 6 | 관리용 |
| /30 | 255.255.255.252 | 0.0.0.3 | 2 | **P2P 링크 표준** |
| /31 | 255.255.255.254 | 0.0.0.1 | 2(RFC 3021) | P2P (최신) |

### 2-2. 🧩 VLSM 기반 인터페이스 설계 (Cisco)

> **시나리오:** `192.168.1.0/24` 를 R1—R2—R3 구조에 VLSM으로 분배  
> LAN 세그먼트(Host 30) 3개 + P2P 링크 2개

```cisco
! ============================================
! R1 - LAN(Host30) + R1<->R2 P2P Link
! ============================================
hostname R1
!
interface GigabitEthernet0/0
 description ### R1 LAN Segment (Host 30) ###
 ip address 192.168.1.1 255.255.255.224                  ! /27 : 192.168.1.0/27 대역, 30개 호스트 수용
 no shutdown
!
interface Serial1/0
 description ### R1 <-> R2 P2P Link ###
 ip address 192.168.1.97 255.255.255.252                 ! /30 : 192.168.1.96/30, P2P 최적화
 no shutdown
!
! Static Summary(선택) - 상위 라우터에 광고할 때 요약 경로
ip route 192.168.1.0 255.255.255.0 Null0                 ! 블랙홀 방지용 요약 경로 (재귀 루프 차단)

! ============================================
! R2 - LAN(Host30) + P2P Links (양쪽)
! ============================================
hostname R2
!
interface GigabitEthernet0/0
 description ### R2 LAN Segment (Host 30) ###
 ip address 192.168.1.33 255.255.255.224                 ! /27 : 192.168.1.32/27
!
interface Serial1/0
 description ### R2 <-> R1 ###
 ip address 192.168.1.98 255.255.255.252                 ! 동일 /30 세그먼트
!
interface Serial1/1
 description ### R2 <-> R3 ###
 ip address 192.168.1.101 255.255.255.252                ! /30 : 192.168.1.100/30

! ============================================
! R3 - LAN(Host30) + R2<->R3 P2P Link
! ============================================
hostname R3
!
interface GigabitEthernet0/0
 description ### R3 LAN Segment (Host 30) ###
 ip address 192.168.1.65 255.255.255.224                 ! /27 : 192.168.1.64/27
!
interface Serial1/0
 ip address 192.168.1.102 255.255.255.252
```

### 2-3. 🧭 Route Summary 설정 (OSPF / EIGRP)

> **시나리오:** R1 하위에 `172.168.0.0/24 ~ 172.168.7.0/24` 8개 대역 존재 → **/21**로 요약해 광고
```cisco
! -------- OSPF : ABR에서 Inter-Area Summary --------
router ospf 1
 area 1 range 172.168.0.0 255.255.248.0                  ! 8개의 /24를 /21로 축약해 Backbone으로 광고
!
! -------- EIGRP : 인터페이스 단위 Summary --------
interface Serial1/0
 ip summary-address eigrp 100 172.168.0.0 255.255.248.0  ! Outgoing 인터페이스에서 요약해 광고

! -------- OSPF network 구문 : Wildcard 사용 --------
router ospf 1
 network 172.168.0.0 0.0.7.255 area 1                    ! 172.168.0.0~172.168.7.255 매칭 (Wildcard)
 network 192.168.1.0 0.0.0.255 area 0                    ! /24 전체 매칭
```

### 2-4. 🛡️ Wildcard 기반 ACL (Filtering)

```cisco
! 특정 서브넷(192.168.1.64/27)만 허용
access-list 10 permit 192.168.1.64 0.0.0.31              ! /27 매칭 → 64~95 범위만 허용
access-list 10 deny   any                                ! 그 외 전부 차단(암묵적 deny 명시화 - 트러블슈팅 편의)
!
interface GigabitEthernet0/0
 ip access-group 10 in                                   ! Inbound 방향 필터 적용
```

### 2-5. 🐧 Linux 측 IP 설정 (netplan 예시)

```cisco
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens160:
      addresses:
        - 192.168.1.10/27                                 # /27 = 30 hosts, 게이트웨이는 .1
      routes:
        - to: default
          via: 192.168.1.1                                # R1 LAN 인터페이스 = 기본 게이트웨이
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. ✅ 필수 검증 명령어

```cisco
! -- 라우팅 / IP 확인
show ip interface brief                                   ! I/F별 IP·상태 한눈에 (설계와 일치 여부 즉시 확인)
show ip route                                             ! 라우팅 테이블 - Summary 반영 여부 확인
show ip route 172.168.5.0                                 ! Longest Match 결과 추적
show ip protocols                                         ! OSPF/EIGRP network 구문·요약 정책 확인

! -- ACL / Wildcard 검증
show access-lists                                         ! ACE별 hit count 확인 (매칭 여부 즉시 판별)
show ip interface Gi0/0 | include access list             ! 어느 방향에 어느 ACL이 걸렸는지

! -- 실통신 검증
ping 192.168.1.65 source GigabitEthernet0/0               ! Source 지정 ping으로 라우팅 경로 검증
traceroute 172.168.7.10                                   ! 요약 경로/실제 경로 홉 확인
```

```cisco
# Linux 측 검증
ip -br addr                          # 인터페이스 IP/Mask 요약 출력
ip route show                        # 라우팅 테이블
ipcalc 192.168.1.0/27                # 네트워크/브로드캐스트/범위 자동 계산 (설계 검증 필수 도구)
ping -c 3 192.168.1.1                # 게이트웨이 도달성
mtr 8.8.8.8                          # 경로별 손실률 모니터링
```

### 3-2. 🚨 대표 장애 시나리오

#### 🚨 시나리오 A. "서브넷은 맞게 잡은 것 같은데 통신이 안 됩니다"

- **증상:** `192.168.1.30` 호스트가 게이트웨이 `192.168.1.33`(R2 /27)로 통신 불가.
- **원인 분석:**
    1. R2 인터페이스는 `192.168.1.32/27` 대역 (범위: 32~63) → 호스트 `.30`은 **다른 서브넷(0~31)**에 속함.
    2. 호스트 입장에서는 GW가 같은 서브넷이 아니라 판단 → ARP 실패.
- **트러블슈팅 단계:**
    1. `ipcalc 192.168.1.30/27` 로 소속 네트워크 재계산 → `192.168.1.0/27` 확인.
    2. 호스트를 올바른 서브넷의 GW(`192.168.1.1`)로 재설정하거나, IP를 `192.168.1.34~62` 범위로 변경.
    3. `show arp` 로 GW MAC 학습 여부 최종 확인.
- **예방책:** 설계 단계에서 **각 세그먼트의 "사용 가능 범위(First/Last Host)"를 문서로 명시**. `Network-ID`, `Broadcast`는 절대 호스트에 할당 금지.

#### 🚨 시나리오 B. "OSPF Summary를 걸었는데 특정 대역만 통신이 끊깁니다"

- **증상:** `172.168.0.0/21`로 요약 후, `172.168.6.0/24` 트래픽이 블랙홀됨.
- **원인 분석:**
    1. Summary 범위(`/21` = 0~7)에는 포함되지만, **실제 하위 라우터에 해당 대역이 존재하지 않는 경우** → 상위 라우터는 요약 경로로 트래픽을 보내지만, 도착지 라우터에서 drop.
    2. 또는 요약 범위 밖 대역이 존재(예: `172.168.8.0/24`)해서 광고 누락.
- **트러블슈팅 단계:**
    1. `show ip route 172.168.6.0` → 어떤 요약 경로로 매칭되는지 확인.
    2. 실제 대역 리스트를 뽑아 **CIDR 범위와 100% 일치**하는지 검증 (`ipcalc` 활용).
    3. Summary를 걸 라우터에 **`ip route 172.168.0.0 255.255.248.0 Null0`** 을 미리 잡아 **블랙홀 라우트 + 재귀 루프 방지** 설정.
    4. 광고 누락 대역이 있다면 요약 범위를 `/20`으로 확장하거나, 두 개의 요약 경로로 분리.
- **예방책:** Summary는 **"연속되고 완전히 채워진 대역"에 대해서만** 수행. IP 설계 시부터 **사이트/부서별로 연속 대역 확보**가 최우선.

---

> 관련: 2. ⚙️ Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리 · 3. 🕐 NTP & SSH & Telnet (원격 접속 & 시간 동기화)