# ⚡ EtherChannel 명령어 퀵 레퍼런스

> **Tag:** #Network #Switch #EtherChannel #LACP #PAgP #Cisco  
> **핵심 요약:** EtherChannel의 모드 조합, `channel-group` 설정, Port-channel 구성, Flag 판독, Load-balancing을 빠르게 확인하기 위한 암기 카드. 설정은 항상 Port-channel 인터페이스 기준으로 하고, 검증은 `show etherchannel summary`의 Flag가 `(SU)`/`(RU)` + `(P)`인지로 판단한다.

---

## 1. 🧠 목적별 선택표

| 요구사항 | 선택 |
|---|---|
| 타사 장비와 링크 집약 | LACP (802.3ad) |
| Cisco 장비끼리 링크 집약 | LACP 또는 PAgP |
| 협상 프로토콜 미지원 장비 연동 | Static(on), 양쪽 동일 필수 |
| 양쪽 능동 협상 | desirable+desirable 또는 active+active |
| 한쪽만 능동 협상 | desirable+auto 또는 active+passive |
| VLAN Traffic 전달 구간 | L2 EtherChannel |
| Routing 구간 | L3 EtherChannel (`no switchport` + IP) |
| Active 멤버 수 제한 | `lacp max-bundle` |
| 최소 멤버 미달 시 채널 down | `port-channel min-links` |
| 트래픽 편중 완화 | `port-channel load-balance` |

---

## 2. 📊 모드 조합 매트릭스

| SW1 | SW2 | 프로토콜 | 결과 |
|---|---|---|:---:|
| on | on | 없음 | ✅ |
| on | desirable | PAgP | ❌ |
| on | auto | PAgP | ❌ |
| on | active | LACP | ❌ |
| on | passive | LACP | ❌ |
| desirable | desirable | PAgP | ✅ |
| desirable | auto | PAgP | ✅ |
| auto | auto | PAgP | ❌ |
| active | active | LACP | ✅ |
| active | passive | LACP | ✅ |
| passive | passive | LACP | ❌ |
| desirable | active | 혼합 | ❌ |
| auto | passive | 혼합 | ❌ |
| desirable | passive | 혼합 | ❌ |
| auto | active | 혼합 | ❌ |

```properties
[1] 최소 한쪽은 능동 (desirable / active)
[2] 수동 + 수동 = 실패
[3] PAgP + LACP = 실패
[4] on은 on끼리만
```

---

## 3. 🔤 프로토콜 비교

| 항목 | PAgP | LACP | on |
|---|---|---|---|
| 표준 | Cisco 전용 | IEEE 802.3ad | 없음 |
| 능동 | desirable | active | 해당 없음 |
| 수동 | auto | passive | 해당 없음 |
| 협상 패킷 | O | O | X |
| 타사 연동 | X | O | 조건부 |
| 불일치 감지 | O | O | X |
| Hot-standby | 제한적 | 지원 | 없음 |
| 지정 명령 | `channel-protocol pagp` | `channel-protocol lacp` | 없음 |

---

## 4. 🔧 기본 설정 명령

```cisco
interface <INTERFACE>
 channel-group <NUMBER> mode {on|auto|desirable|active|passive}
```

프로토콜 명시 및 범위 지정:

```cisco
interface range <INTERFACE-RANGE>
 channel-protocol lacp
 channel-group 1 mode active
```

해제:

```cisco
interface <INTERFACE>
 no channel-group <NUMBER>
!
no interface port-channel <NUMBER>
```

검증:

```cisco
show etherchannel summary
show running-config interface <INTERFACE>
```

---

## 5. 🔗 L2 EtherChannel (LACP)

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```cisco
configure terminal
!
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30
!
interface range <INTERFACE-RANGE>
 channel-protocol lacp
 channel-group 1 mode active
```

상대 Switch는 동일한 Po 설정에 멤버만 `mode passive` 또는 `mode active`로 지정합니다.

검증:

```cisco
show etherchannel summary
show lacp neighbor
show interfaces trunk
show spanning-tree interface port-channel 1
```

---

## 6. 🔗 L2 EtherChannel (PAgP)

```cisco
configure terminal
!
interface port-channel 12
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface range <INTERFACE-RANGE>
 channel-protocol pagp
 channel-group 12 mode desirable
```

상대 Switch는 `desirable` 또는 `auto`로 설정합니다.

검증:

```cisco
show etherchannel summary
show pagp neighbor
show pagp 12 internal
show interfaces trunk
```

---

## 7. 🔒 Static(on) EtherChannel

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 shutdown
 channel-group 3 mode on
!
interface port-channel 3
 switchport mode trunk
!
interface range <INTERFACE-RANGE>
 no shutdown
```

검증:

```cisco
show etherchannel summary
show interfaces status
show mac address-table
show spanning-tree
```

```properties
on 모드 위험
협상 없음 → 상대 상태 확인 불가
한쪽만 설정 → L2 Loop 발생 가능
설정 중에는 멤버 포트 shutdown 유지
```

---

## 8. 🌐 L3 EtherChannel

```cisco
configure terminal
!
ip routing
!
interface port-channel 20
 no switchport
 ip address <IP-ADDRESS> <SUBNET-MASK>
!
interface range <INTERFACE-RANGE>
 no switchport
 channel-protocol lacp
 channel-group 20 mode active
```

검증:

```cisco
show etherchannel summary
show ip interface brief
show interfaces port-channel 20
show ip route
ping <IP-ADDRESS>
```

```properties
L2 정상 = Po(SU)
L3 정상 = Po(RU)

L3 필수 조건
ip routing 활성화 / Po에 no switchport / 멤버 포트에도 no switchport 권장
```

---

## 9. 🚩 Flag 판독표

```properties
Group  Port-channel  Protocol    Ports
------+-------------+-----------+---------------------
1      Po1(SU)       LACP        Gi0/1(P)  Gi0/2(P)
```

| 채널 Flag | 의미 |
|---|---|
| S | Layer2 채널 |
| R | Layer3 채널 |
| U | in use (정상) |
| D | down |
| M | min-links 미충족으로 미사용 |
| u | unsuitable for bundling |

| 멤버 Flag | 의미 | 대응 |
|---|---|---|
| P | bundled (정상) | 조치 불필요 |
| I | stand-alone (협상 실패) | 모드·프로토콜 확인 |
| s | suspended (설정 불일치) | Speed·Duplex·Trunk·VLAN 확인 |
| D | down | 케이블·포트 상태 확인 |
| H | Hot-standby (LACP 대기) | Max Bundle·Port Priority 확인 |
| w | waiting to be aggregated | 수 초 대기 후 재확인 |

```properties
정상 판정
L2 → Po(SU) + 전 멤버 (P)
L3 → Po(RU) + 전 멤버 (P)
```

> Packet Tracer는 `H`, `w`, `M`, `u` 등 일부 Flag를 실장비와 다르게 표시하거나 표시하지 않을 수 있다.

---

## 10. ⚖️ Load-balancing

```cisco
port-channel load-balance src-dst-ip
```

확인:

```cisco
show etherchannel load-balance
show interfaces counters
show interfaces port-channel <CHANNEL-NUMBER>
```

| 기준 | 적합한 환경 |
|---|---|
| `src-mac` | 다수 단말이 소수 서버로 향하는 L2 구간 |
| `dst-mac` | 소수 출발지가 다수 목적지로 향하는 구간 |
| `src-dst-mac` | 동일 L2 세그먼트 내 다수 대 다수 |
| `src-ip` | 다수 IP가 소수 목적지로 향하는 구간 |
| `dst-ip` | 소수 출발지가 다수 목적지로 향하는 구간 |
| `src-dst-ip` | L3 라우팅 경유 일반 환경 (가장 무난) |

```properties
Load-balancing 본질
Flow 단위 Hash 배정 ≠ 패킷 단위 균등 분산
결과: 단일 세션 처리량 ≤ 멤버 링크 1개 대역폭
```

> 지원 알고리즘 목록은 플랫폼과 IOS 버전에 따라 다르며 Packet Tracer는 옵션이 제한적이다. 플랫폼별 지원 여부 확인 필요.

---

## 11. 🎚️ LACP 세부 튜닝

```cisco
lacp system-priority 100
!
interface <INTERFACE>
 lacp port-priority 100
 lacp rate fast
!
interface port-channel <NUMBER>
 lacp max-bundle 4
 port-channel min-links 2
```

검증:

```cisco
show lacp sys-id
show lacp neighbor
show lacp <CHANNEL-NUMBER> internal
show etherchannel summary
```

```properties
System Priority (기본 32768) = 낮은 쪽이 멤버 선택 결정권, 동률이면 System MAC
Port Priority   (기본 32768) = 낮은 포트가 Active 우선, 초과분은 Hot-standby(H)
```

> `lacp rate fast`, `lacp max-bundle`, `port-channel min-links`의 지원 여부와 동작은 플랫폼마다 다르다. 플랫폼별 지원 여부 확인 필요.

---

## 12. 🧱 멤버 포트 일치 조건

| 항목 | 불일치 결과 | 확인 명령 |
|---|---|---|
| Speed | 미형성 또는 suspended | `show interfaces status` |
| Duplex | 미형성 또는 suspended | `show interfaces status` |
| Switchport 모드 | suspended | `show interfaces <INTERFACE> switchport` |
| Trunk Encapsulation | 미형성 | `show interfaces <INTERFACE> switchport` |
| Allowed VLAN | suspended | `show interfaces trunk` |
| Native VLAN | suspended, VLAN 누수 | `show interfaces trunk` |
| Access VLAN | suspended | `show vlan brief` |
| MTU | 미형성 가능 | `show interfaces <INTERFACE>` |
| STP Cost / Priority | 경로 판정 왜곡 | `show spanning-tree interface <INTERFACE>` |

일괄 정렬 후 검증:

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30
 speed auto
 duplex auto
```

```cisco
show interfaces trunk
show interfaces status
show etherchannel summary
```

---

## 13. 🛑 err-disabled 복구

```cisco
interface <INTERFACE>
 shutdown
 no shutdown
!
errdisable recovery cause channel-misconfig
errdisable recovery interval 300
```

검증:

```cisco
show interfaces status err-disabled
show errdisable recovery
show etherchannel summary
show logging
```

```properties
발생 흐름
한쪽만 EtherChannel 구성 → 동일 BPDU가 여러 포트 수신
        → Misconfig Guard 동작 → 포트 err-disabled
```

> 원인을 해소하지 않고 복구만 반복하면 즉시 재발한다.

---

## 14. 🌳 STP 연계 확인

```cisco
show spanning-tree
show spanning-tree interface port-channel <CHANNEL-NUMBER>
show spanning-tree vlan <VLAN-ID>
```

```properties
STP 판정 순서
Root Bridge → Root Port → Designated Port → Alternate

EtherChannel 적용 후
판정 단위 = Port-channel 논리 포트
멤버 개별 Blocking은 발생하지 않음
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

```cisco
configure terminal
!
spanning-tree vlan <VLAN-ID> root primary
!
interface port-channel <CHANNEL-NUMBER>
 spanning-tree cost 10
```

검증:

```cisco
show spanning-tree vlan <VLAN-ID>
show etherchannel summary
```

---

## 15. 🧰 통합 실습 부가기술

DHCP 서버 및 Relay:

```cisco
configure terminal
!
ip dhcp excluded-address <IP-ADDRESS>
!
ip dhcp pool VLAN100
 network <NETWORK-ADDRESS> <SUBNET-MASK>
 default-router <IP-ADDRESS>
 dns-server <IP-ADDRESS>
 lease 5 0 0
!
interface vlan <VLAN-ID>
 ip helper-address <IP-ADDRESS>
```

NTP 및 Anti-Fragmentation ACL:

```cisco
configure terminal
!
ntp authenticate
ntp authentication-key 1 md5 cisco1234
ntp trusted-key 1
ntp server <IP-ADDRESS> key 1
!
ip access-list extended ANTI-FRAG
 deny ip any host <IP-ADDRESS> fragments
 permit ip any any
!
interface port-channel <CHANNEL-NUMBER>
 ip access-group ANTI-FRAG in
```

검증:

```cisco
show ip dhcp binding
show ip dhcp pool
show ntp status
show ntp associations
show access-lists ANTI-FRAG
show ip interface port-channel <CHANNEL-NUMBER>
```

```properties
부가기술 핵심
DHCP = Gateway IP를 excluded-address로 제외
NTP  = 인증 키로 신뢰 소스만 허용
ACL  = fragments 키워드로 비초기 조각 차단
적용 위치 = 물리 멤버가 아닌 Port-channel 인터페이스
```

---

## 16. 🔍 검증 명령어

```cisco
show etherchannel summary
show etherchannel <CHANNEL-NUMBER> detail
show etherchannel load-balance
show lacp neighbor
show lacp sys-id
show lacp <CHANNEL-NUMBER> internal
show pagp neighbor
```

```cisco
show interfaces port-channel <CHANNEL-NUMBER>
show interfaces status
show interfaces <INTERFACE> switchport
show interfaces trunk
show spanning-tree interface port-channel <CHANNEL-NUMBER>
show ip interface brief
show ip route
show interfaces status err-disabled
show logging
```

---

## 17. 🔄 안전한 변경 및 Rollback

```properties
현재 확인 → 백업 → 경로 확인 → 적용 → 검증 → 저장
```

변경 전:

```cisco
show etherchannel summary
show interfaces status
show interfaces trunk
show spanning-tree
show running-config interface <INTERFACE>
```

Rollback 및 재검증:

```cisco
configure terminal
!
interface range <INTERFACE-RANGE>
 no channel-group <CHANNEL-NUMBER>
!
no interface port-channel <CHANNEL-NUMBER>
```

```cisco
show etherchannel summary
show interfaces status
show interfaces trunk
show spanning-tree
copy running-config startup-config
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 18. ⚠️ 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| auto + auto | 채널 형성 안 됨 (양쪽 수동) |
| passive + passive | 채널 형성 안 됨 (양쪽 수동) |
| on + desirable | 채널 형성 안 됨, Loop 위험 |
| desirable + active | PAgP와 LACP 혼합으로 실패 |
| on 모드 | 협상 없음, 상대 상태 확인 불가 |
| 물리 포트에 Trunk 설정 | Po에 반영되지 않아 불일치 유발 |
| 물리 포트에 IP 설정 | L3 채널이 되지 않음 |
| `no switchport` 미적용 | Po에 IP 부여 불가 |
| `ip routing` 미설정 | L3 채널이 있어도 라우팅 불가 |
| Po 번호 불일치 | 로컬 의미이므로 채널은 형성 가능 |
| 멤버가 `(I)` | 협상 실패이지 설정 불일치가 아님 |
| 멤버가 `(s)` | 설정 불일치이지 협상 문제가 아님 |
| 멤버가 `(H)` | 정상 대기 멤버, 장애 아님 |
| 채널 Up | VLAN 통신 보장 아님 |
| 링크 4개 묶음 | 단일 세션은 여전히 링크 1개 속도 |
| Load-balance 변경 | 장비 전역 설정이며 Flow 재배치 발생 |
| err-disabled 복구 | 원인 미해소 시 즉시 재발 |
| 멤버 개별 Blocking | 채널이 실제로 형성되지 않은 상태 |
| SPAN 또는 Port-security 포트 | EtherChannel 멤버로 사용 불가 |
| Packet Tracer 출력 | 실장비와 Flag·옵션이 다를 수 있음 |

---

## 19. 🚑 빠른 장애 진단 순서

```properties
물리 링크 Up 확인
    ↓
show etherchannel summary Flag 확인
    ↓
(D)  → 케이블·SFP·shutdown 확인
(I)  → 프로토콜·모드 조합 확인
(s)  → Speed·Duplex·Trunk·VLAN 확인
(w)  → 수 초 대기 후 재확인
(H)  → Max Bundle·Port Priority 확인
    ↓
err-disabled 여부 확인
    ↓
Port-channel의 Trunk·IP 설정 위치 확인
    ↓
Allowed VLAN 및 Native VLAN 확인
    ↓
STP에서 Po 단일 논리 포트 인식 확인
    ↓
L3라면 ip routing·no switchport·Route 확인
    ↓
상위 서비스(DHCP·NTP·ACL) 확인
```

---

> 📌 **핵심 요약**
> - 묶기: `channel-group <NUMBER> mode {on|auto|desirable|active|passive}`
> - 프로토콜 명시: `channel-protocol {pagp|lacp}`
> - PAgP 능동/수동: `desirable` / `auto`
> - LACP 능동/수동: `active` / `passive`
> - 최소 한쪽이 능동이어야 하며 on은 on끼리만 묶인다.
> - PAgP와 LACP는 혼합할 수 없다.
> - L2 설정: `interface port-channel <NUMBER>`에 Trunk 적용
> - L3 설정: `ip routing` → `no switchport` → `ip address <IP-ADDRESS>`
> - 상태 확인: `show etherchannel summary`
> - 정상 Flag: L2 `(SU)`, L3 `(RU)`, 멤버 `(P)`
> - 이상 Flag: `(I)` 협상 실패, `(s)` 설정 불일치, `(D)` 물리 Down
> - 부하 분산: `port-channel load-balance src-dst-ip`
> - LACP 튜닝: `lacp system-priority`, `lacp port-priority`, `lacp max-bundle`
> - 복구: err-disabled는 원인 해소 후 `shutdown` / `no shutdown`
> - 관련: 5-1. 🔗 EtherChannel 핵심 (PAgP & LACP & L2·L3) · 5-2. 🧩 통합 실습 부가기술 (DHCP & NTP & Anti-Frag ACL) · 5-3. 🧩 EtherChannel 통합 정리 · 5-4. 🚑 EtherChannel 트러블슈팅 치트시트 · 5-5. ⚡ EtherChannel 명령어 퀵 레퍼런스 · 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence
