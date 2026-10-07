# HSRP & VRRP 명령어 퀵 레퍼런스

> **Tag:** #Network #FHRP #HSRP #VRRP #Redundancy #Cisco  
> **핵심 요약:** HSRP와 VRRP의 설정·검증 명령어와 기본값 함정을 빠르게 확인하기 위한 암기 카드. HSRP는 Preempt를 반드시 명시하고, Tracking Decrement는 Priority 차이보다 크게 설계하며, VRRP는 IOS 버전별 문법 차이를 먼저 확인한다.

---

## 1. 요구사항별 선택

| 요구사항 | 사용할 기능 또는 프로토콜 |
|---|---|
| Gateway 단일 장애점 제거 | HSRP 또는 VRRP |
| 순수 Cisco 환경 | HSRP |
| 멀티벤더 또는 표준 요구 | VRRP |
| 단일 그룹 내 자동 부하 분산 | GLBP |
| 그룹 분할 부하 분산 | MHSRP 또는 VRRP 그룹 2개 |
| WAN 장애 감지 | Interface Tracking |
| 복구 후 원래 Active 복귀 | Preempt |
| 복구 직후 Black Hole 방지 | Preempt Delay |
| Group 번호 256 이상 | HSRPv2 |
| IPv6 Gateway 이중화 | HSRPv2 또는 VRRPv3 |
| IPsec SA 승계 | HSRP `name` + `crypto map redundancy` |

---

## 2. HSRP vs VRRP vs GLBP 비교

| 항목 | HSRP | VRRP | GLBP |
|---|---|---|---|
| 표준 | Cisco 전용 | RFC 표준 | Cisco 전용 |
| 역할 | Active / Standby | Master / Backup | AVG / AVF |
| Priority 기본값 | 100 | 100 | 100 |
| IP Owner Priority | 해당 없음 | 255 (고정) | 해당 없음 |
| Preempt 기본값 | 비활성 | 활성 | 활성 |
| 가상 MAC | v1 `0000.0c07.acXX`<br>v2 `0000.0c9f.fXXX` | `0000.5e00.01XX` | `0007.b400.xxyy` 계열 |
| 전송 | UDP 1985 | IP Protocol 112 | UDP 3222 |
| 멀티캐스트 | v1 224.0.0.2<br>v2 224.0.0.102 | 224.0.0.18 | 224.0.0.102 |
| Hello | 3초 | 1초 | 3초 |
| Hold / Dead | 10초 | Master Down 계산값 | 10초 |
| Group 범위 | v1 0~255<br>v2 0~4095 | 1~255 | 0~1023 |
| 동시 Forwarding | 그룹당 1대 | 그룹당 1대 | 그룹 내 다수 |

```properties
암기 포인트
HSRP Preempt = 꺼져 있다 (반드시 켠다)
VRRP Preempt = 켜져 있다
HSRP = UDP 1985
VRRP = IP Protocol 112 (UDP 아님)
```

> GLBP 지원 여부는 플랫폼별 확인이 필요하며, Packet Tracer에서는 제공되지 않거나 동작이 제한될 수 있습니다.

---

## 3. HSRPv1 vs HSRPv2

| 항목 | HSRPv1 | HSRPv2 |
|---|---|---|
| Group 범위 | 0 ~ 255 | 0 ~ 4095 |
| 가상 MAC | `0000.0c07.acXX` | `0000.0c9f.fXXX` |
| 멀티캐스트 | 224.0.0.2 | 224.0.0.102 |
| UDP Port | 1985 | 1985 |
| Millisecond 타이머 | 제한적 | 지원 |
| IPv6 | 미지원 | 지원 |
| 상호 호환 | v2와 호환 불가 | v1과 호환 불가 |

설정:

```cisco
interface <INTERFACE>
 standby version 2
```

검증:

```cisco
show standby all
show running-config interface <INTERFACE>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```properties
Version 불일치 결과
서로 다른 멀티캐스트 주소 사용
→ 상대 Hello 미인식
→ 양쪽 Active
→ 가상 IP 중복
```

---

## 4. HSRP 기본 설정

가상 IP:

```cisco
interface <INTERFACE>
 standby <GROUP> ip <IP-ADDRESS>
```

Priority:

```cisco
interface <INTERFACE>
 standby <GROUP> priority 150
```

Preempt:

```cisco
interface <INTERFACE>
 standby <GROUP> preempt
```

Preempt Delay:

```cisco
interface <INTERFACE>
 standby <GROUP> preempt delay minimum 60
```

Timer:

```cisco
interface <INTERFACE>
 standby <GROUP> timers 3 10
```

Millisecond Timer(지원 플랫폼):

```cisco
interface <INTERFACE>
 standby <GROUP> timers msec 250 msec 800
```

그룹 이름:

```cisco
interface <INTERFACE>
 standby <GROUP> name IPSEC_R
```

검증:

```cisco
show standby brief
show standby <INTERFACE> <GROUP>
```

---

## 5. HSRP Active 전체 구성

```cisco
configure terminal
!
interface <INTERFACE>
 ip address 192.168.10.252 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 timers 3 10
 standby 10 authentication md5 key-string cisco123
 standby 10 track 10 decrement 100
```

검증:

```cisco
show standby brief
show standby all
show track brief
show ip route
```

---

## 6. HSRP Standby 전체 구성

```cisco
configure terminal
!
interface <INTERFACE>
 ip address 192.168.10.253 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 100
 standby 10 preempt
 standby 10 timers 3 10
 standby 10 authentication md5 key-string cisco123
```

검증:

```cisco
show standby brief
show standby all
```

```properties
Standby 측에 Preempt가 없으면
Active 복구 후 Tracking 절체가 불가능합니다.
Priority가 낮은 쪽에도 Preempt는 필요합니다.
```

---

## 7. Interface Tracking

Track Object 방식(권장):

```cisco
track 10 interface <INTERFACE> line-protocol
!
interface <INTERFACE>
 standby <GROUP> track 10 decrement 100
```

간이 문법:

```cisco
interface <INTERFACE>
 standby <GROUP> track <INTERFACE> 100
```

다중 Track:

```cisco
track 10 interface <INTERFACE> line-protocol
track 20 interface <INTERFACE> line-protocol
!
interface <INTERFACE>
 standby <GROUP> track 10 decrement 60
 standby <GROUP> track 20 decrement 60
```

IP Routing 상태 추적:

```cisco
track 30 interface <INTERFACE> ip routing
```

검증:

```cisco
show track
show track brief
show track 10
show standby all
```

```properties
Decrement 설계
Decrement > (Active Priority - Standby Priority)

예시
150 vs 100 → Decrement 60 이상
150 vs 140 → Decrement 20 이상
Decrement 기본값 10 → 대부분 절체 실패
```

> Track Object의 세부 옵션(`ip routing`, `delay` 등) 지원 여부는 플랫폼별 확인이 필요합니다.

---

## 8. MHSRP 부하 분산

R1 (VLAN 10 Active, VLAN 20 Standby):

```cisco
interface vlan 10
 ip address 192.168.10.252 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 track 10 decrement 60
!
interface vlan 20
 ip address 192.168.20.252 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 priority 100
 standby 20 preempt
```

R2 (VLAN 20 Active, VLAN 10 Standby):

```cisco
interface vlan 10
 ip address 192.168.10.253 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 100
 standby 10 preempt
!
interface vlan 20
 ip address 192.168.20.253 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 priority 150
 standby 20 preempt delay minimum 60
 standby 20 track 20 decrement 60
```

검증:

```cisco
show standby brief
show spanning-tree vlan 10
show spanning-tree vlan 20
show ip route
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 9. STP Root와 FHRP Active 정렬

```cisco
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
```

수동 Priority 지정:

```cisco
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 8192
```

검증:

```cisco
show spanning-tree vlan 10
show spanning-tree root
show standby brief
show interfaces trunk
```

```properties
STP 판정 순서
Root Bridge → Root Port → Designated Port → Alternate/Blocking

정합 설계
VLAN 10 : STP Root = R1, FHRP Active = R1
VLAN 20 : STP Root = R2, FHRP Active = R2
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

관련 개념은 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence와 3-4. 🚀 RSTP & MSTP (초대형망 STP)를 참고하십시오.

---

## 10. HSRP 인증

Plain Text:

```cisco
interface <INTERFACE>
 standby <GROUP> authentication cisco123
```

MD5 Key String:

```cisco
interface <INTERFACE>
 standby <GROUP> authentication md5 key-string cisco123
```

MD5 Key Chain:

```cisco
key chain HSRP_KEY
 key 1
  key-string cisco123
!
interface <INTERFACE>
 standby <GROUP> authentication md5 key-chain HSRP_KEY
```

검증:

```cisco
show standby all
show key chain
show logging
```

> 인증 옵션 지원 범위는 플랫폼별 확인이 필요합니다. Packet Tracer는 MD5 Key Chain 등 일부 옵션을 지원하지 않을 수 있습니다.

---

## 11. VRRP 구 문법

Master:

```cisco
interface <INTERFACE>
 ip address 192.168.10.252 255.255.255.0
 vrrp 10 ip 192.168.10.254
 vrrp 10 priority 150
 vrrp 10 preempt delay minimum 60
 vrrp 10 timers advertise 1
 vrrp 10 authentication md5 key-string cisco123
 vrrp 10 track 10 decrement 100
```

Backup:

```cisco
interface <INTERFACE>
 ip address 192.168.10.253 255.255.255.0
 vrrp 10 ip 192.168.10.254
 vrrp 10 priority 100
```

검증:

```cisco
show vrrp brief
show vrrp all
show vrrp interface <INTERFACE>
```

```properties
VRRP 기본값
Priority          = 100
IP Owner Priority = 255 (가상 IP == 물리 IP일 때)
Preempt           = 활성
Advertisement     = 1초
Multicast         = 224.0.0.18
IP Protocol       = 112
VRID              = 1 ~ 255
```

---

## 12. 🆕 VRRP 신규 FHRP 문법

```cisco
fhrp version vrrp v3
!
interface <INTERFACE>
 vrrp 10 address-family ipv4
  address 192.168.10.254 primary
  priority 150
  preempt delay minimum 60
  timers advertise 1000
```

검증:

```cisco
show vrrp
show vrrp detail
show fhrp
show fhrp verbose
```

```properties
문법 확인 방법
interface <INTERFACE>
 vrrp ?
→ 사용 가능한 Keyword로 구 문법/신규 문법 판별
```

> VRRP 문법은 IOS 버전과 플랫폼에 따라 다릅니다. 구 문법과 신규 문법이 공존하지 않는 장비도 있으므로 플랫폼별 지원 여부 확인이 필요합니다. Packet Tracer에서는 VRRP가 지원되지 않거나 옵션이 제한될 수 있습니다.

---

## 13. SVI 및 L3 Switch 연동

VLAN 및 SVI:

```cisco
vlan 10
 name USERS
!
interface vlan 10
 ip address 192.168.10.252 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 150
 standby 10 preempt delay minimum 60
```

L3 Routing 활성화:

```cisco
ip routing
```

Inter-Switch Trunk:

```cisco
interface <INTERFACE>
 description L3SW-INTERCONNECT
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

검증:

```cisco
show ip interface brief
show vlan brief
show interfaces trunk
show standby brief
show ip route
```

```properties
SVI Up 조건
[1] VLAN이 VLAN Database에 존재
[2] 해당 VLAN에 Up 상태 Port 존재
[3] SVI가 no shutdown
[4] Trunk Allowed VLAN에 VLAN 포함
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

상세 개념은 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing과 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3)을 참고하십시오.

---

## 14. DHCP Gateway 배포

```cisco
ip dhcp excluded-address 192.168.10.250 192.168.10.254
!
ip dhcp pool VLAN10_POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 8.8.8.8
```

검증:

```cisco
show ip dhcp pool
show ip dhcp binding
show running-config | section dhcp
```

```properties
default-router = FHRP 가상 IP
default-router ≠ 장비 물리 IP

물리 IP를 배포하면
평시 통신은 정상
장애 시 절체 효과 없음
```

상세 내용은 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

## 15. IPsec 연동

HSRP 그룹 이름 지정:

```cisco
interface <INTERFACE>
 standby 2 ip <IP-ADDRESS>
 standby 2 priority 120
 standby 2 preempt delay minimum 60
 standby 2 name IPSEC_R
 standby 2 track <INTERFACE> 100
```

crypto map 연동:

```cisco
crypto isakmp keepalive 10
!
interface <INTERFACE>
 no crypto map IPSEC_REDUNDANCY
 crypto map IPSEC_REDUNDANCY redundancy IPSEC_R
```

검증:

```cisco
show standby all
show crypto isakmp peer
show crypto isakmp sa
show crypto engine connections active
```

> Standby 장비에는 SA가 잡히지 않는 것이 정상입니다. 상세 절차는 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동)을 참고하십시오.

---

## 16. 검증 명령어

HSRP 요약:

```cisco
show standby brief
```

HSRP 상세:

```cisco
show standby all
show standby <INTERFACE> <GROUP>
```

VRRP:

```cisco
show vrrp brief
show vrrp all
show vrrp interface <INTERFACE>
```

Tracking:

```cisco
show track
show track brief
```

Interface:

```cisco
show ip interface brief
show interfaces <INTERFACE> status
```

L2:

```cisco
show vlan brief
show interfaces trunk
show mac address-table vlan <VLAN-ID>
```

STP:

```cisco
show spanning-tree vlan <VLAN-ID>
show spanning-tree root
```

L3:

```cisco
show ip route
show ip route <IP-ADDRESS>
show ip ospf neighbor
show arp
```

Log 및 Debug:

```cisco
show logging
debug standby events
undebug all
```

---

## 17. show standby brief 판독

```properties
Interface  Grp  Pri  P  State    Active          Standby         Virtual IP
Vl10       10   150  P  Active   local           192.168.10.253  192.168.10.254
Vl20       20   100  P  Standby  192.168.20.253  local           192.168.20.254
```

| 열 | 의미 | 판독 포인트 |
|---|---|---|
| Grp | Group 번호 | 양쪽 일치 여부 |
| Pri | 현재 Priority | Tracking 감소가 반영된 실시간 값 |
| P | Preempt 활성 | `P`가 없으면 Preempt 비활성 |
| State | 현재 역할 | Active / Standby / Listen / Init |
| Active | Active 장비 IP | `local`은 자기 자신 |
| Standby | Standby 장비 IP | `unknown`이면 상대 미인식 |
| Virtual IP | 가상 IP | 양쪽 완전 일치 필수 |

```properties
Standby 열이 unknown
= 상대 Hello를 못 받고 있음
= 그룹/가상 IP/Version/인증/L2 경로 확인

Pri가 설정값보다 낮음
= Tracking이 동작 중
= show track brief로 대상 확인
```

---

## 18. 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| HSRP Preempt 미설정 | 복구 후 원래 Active로 돌아오지 않는다 |
| VRRP Preempt 미설정 | 기본이 활성이므로 자동 탈환한다 |
| Priority 양쪽 100 | 어느 장비가 Active인지 예측 불가 |
| Priority 동일 | 더 높은 Interface IP를 가진 장비가 Active |
| Decrement 생략 | 기본값 10이 적용되어 절체가 안 될 수 있다 |
| Decrement < Priority 차이 | Priority만 떨어지고 절체는 안 된다 |
| Standby에 Preempt 없음 | Tracking이 동작해도 탈취하지 못한다 |
| Tracking 미설정 | WAN 장애 시 Black Hole이 발생한다 |
| State가 Active | 상위 경로가 있다는 뜻은 아니다 |
| 가상 IP Ping 성공 | 외부 통신 가능을 의미하지 않는다 |
| HSRPv1과 v2 혼재 | 호환 불가로 Active-Active가 된다 |
| Group 번호 256 이상 | HSRPv1에서 사용 불가 |
| 타이머 한쪽만 변경 | Flapping 또는 Active-Active 유발 |
| 인증 한쪽만 설정 | 상대 Hello 거부로 Active-Active |
| 가상 IP가 다른 Subnet | 그룹이 정상 동작하지 않는다 |
| STP Root와 Active 불일치 | 통신은 되지만 경로가 우회한다 |
| DHCP default-router가 물리 IP | 평시 정상, 장애 시 절체 무효 |
| Preempt Delay 미설정 | 재부팅 직후 미수렴 상태로 탈환 |
| VRRP 문법 오류 | 플랫폼별 구/신규 문법 차이 |
| Packet Tracer 검증 완료 | 실장비 동작 보장이 아니다 |

---

## 19. 빠른 장애 진단 순서

```properties
Interface 및 SVI Up 확인
    ↓
Group 번호 일치 확인
    ↓
가상 IP 일치 확인
    ↓
HSRP Version 일치 확인
    ↓
인증 방식 및 Key 일치 확인
    ↓
Hello/Hold 타이머 일치 확인
    ↓
Priority 및 Preempt 확인
    ↓
Tracking 대상 및 Decrement 확인
    ↓
새 Active의 Routing Table 확인
    ↓
STP Root와 Active 위치 정합성 확인
    ↓
단말 Gateway 및 DHCP 배포 확인
```

한 줄 진단 세트:

```cisco
show standby brief
show standby all
show track brief
show ip route
show spanning-tree vlan <VLAN-ID>
```

```properties
계층 판정
down/down          → L1 (Cable, Port)
up/up + MAC 미학습 → L2 (VLAN, Trunk, STP)
L2 정상 + 외부 불통 → L3 (Routing, ACL, NAT)
L2/L3 정상 + 역할 이상 → FHRP
```

---

## 20. 안전한 변경 원칙

변경 순서:

```properties
[1] 현재 확인
[2] 백업
[3] 경로 확인 (Console/OOB 확보)
[4] 적용 (Standby 측 먼저)
[5] 검증
[6] 저장
```

변경 전:

```cisco
show running-config interface <INTERFACE>
show standby brief
show standby all
show track brief
show ip route
show spanning-tree vlan <VLAN-ID>
```

변경 후:

```cisco
show standby brief
show standby all
show track brief
show ip route
show arp
show logging
```

저장:

```cisco
copy running-config startup-config
```

Rollback:

```cisco
interface <INTERFACE>
 no standby <GROUP> priority
 no standby <GROUP> preempt
 no standby <GROUP> track 10 decrement 100
 no standby version 2
```

그룹 제거:

```cisco
interface <INTERFACE>
 no standby <GROUP> ip <IP-ADDRESS>
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

경로 판정은 항상 다음 순서를 따릅니다.

```properties
[1] Longest Prefix Match
[2] Administrative Distance
[3] Metric
```

---

> 📌 **핵심 요약**
> - 가상 IP: `standby <GROUP> ip <IP-ADDRESS>` / `vrrp <GROUP> ip <IP-ADDRESS>`
> - Priority: `standby <GROUP> priority 150` (기본 100)
> - Preempt: `standby <GROUP> preempt` (HSRP 기본 비활성)
> - Preempt Delay: `standby <GROUP> preempt delay minimum 60`
> - Timer: `standby <GROUP> timers 3 10` (기본 Hello 3초 / Hold 10초)
> - Version: `standby version 2` (Group 0~4095, 224.0.0.102)
> - Tracking: `standby <GROUP> track <INTERFACE> <DECREMENT>`
> - Track Object: `track 10 interface <INTERFACE> line-protocol`
> - 인증: `standby <GROUP> authentication md5 key-string <KEY>`
> - IPsec 연동: `standby <GROUP> name IPSEC_R`
> - HSRP 상태: `show standby brief` / `show standby all`
> - VRRP 상태: `show vrrp brief` / `show vrrp all`
> - Tracking 상태: `show track brief`
> - HSRP는 UDP 1985, VRRP는 IP Protocol 112를 사용한다.
> - Decrement는 Priority 차이보다 크게, STP Root와 Active 위치는 VLAN별로 맞춘다.
> - 관련: 7-1. ⚖️ HSRP & VRRP (Gateway Redundancy) · 7-2. 🧩 HSRP & VRRP 통합 정리 · 7-3. 🚑 HSRP & VRRP 트러블슈팅 치트시트 · 7-4. ⚡ HSRP & VRRP 명령어 퀵 레퍼런스 · 4-1. 🏢 Routed Interface & SVI & Inter-VLAN Routing · 5-3-6. ⚖️ IPsec 이중화 (HSRP 연동)
