# NAT & PAT 명령어 퀵 레퍼런스

> **Tag:** #Network #NAT #PAT #IPServices #Cisco  
> **핵심 요약:** NAT 설정은 **방향 지정 → 대상 선별 → 변환 규칙 → 검증** 4단계로 고정됩니다. `ip nat inside`/`ip nat outside`가 없으면 어떤 규칙도 동작하지 않고, `overload` 유무가 PAT와 Dynamic NAT를 가르며, NAT 대상 ACL의 `permit`은 허용이 아니라 변환 대상 선정입니다.

---

## 1. 요구사항별 선택 가이드

| 요구사항 | 사용할 방식 |
|---|---|
| 내부 다수 단말이 공인 IP 1개로 인터넷 사용 | PAT (`overload`) |
| 공인 IP 여유가 있어 1:1 임시 배정 | Dynamic NAT (Pool) |
| 내부 서버를 공인 IP 1개로 완전 공개 | Static NAT |
| 공인 IP 1개로 특정 Port만 서버 공개 | Port Forwarding (Static PAT) |
| 외부 서버 주소를 내부에서 다르게 보이게 | `ip nat outside source` |
| VPN 대상 트래픽만 변환 제외 | NAT ACL `deny` 또는 route-map |
| Interface 주소가 동적으로 바뀌는 환경 | `interface <INTERFACE> overload` |
| 진행 중 변환 Entry 확인 | `show ip nat translations` |
| Pool 사용량과 Miss 확인 | `show ip nat statistics` |

---

## 2. NAT 4대 용어

| 용어 | 누구의 주소 | 누구의 시선 | 예시 |
|---|---|---|---|
| **Inside Local** | 내부 호스트 | 내부망 | `192.168.10.10` |
| **Inside Global** | 내부 호스트 | 외부망 | `121.160.23.2` |
| **Outside Local** | 외부 호스트 | 내부망 | `200.1.1.1` |
| **Outside Global** | 외부 호스트 | 외부망 | `200.1.1.1` |

```properties
Inside = 내부 호스트의 주소 / Outside = 외부 호스트의 주소
Local  = 내부망 관점 표현   / Global  = 외부망 관점 표현

일반 Inside NAT 환경    : Outside Local = Outside Global
ip nat outside source 시 : Outside Local ≠ Outside Global
```

---

## 3. NAT 유형 비교

| 유형 | 매핑 | 필요 공인 IP | 외부→내부 최초 접속 | Entry 생성 시점 |
|---|---|:---:|:---:|---|
| **Static NAT** | 1:1 고정 | 호스트 수만큼 | 가능 | 설정 즉시(상시) |
| **Dynamic NAT** | 다수 ↔ Pool | Pool 크기 | 불가 | 내부 트래픽 발생 시 |
| **PAT (Overload)** | 다수 → 1 (Port 구분) | 1개 | 불가 | 내부 트래픽 발생 시 |
| **Static PAT** | 1:Port ↔ 1:Port | 1개 공유 | 지정 Port만 가능 | 설정 즉시(상시) |

```properties
overload 있음 = PAT (Port로 세션 구분, 공인 IP 1개 공유)
overload 없음 = Dynamic NAT (주소 1:1 임시 배정, Pool 고갈 위험)
```

---

## 4. 1단계 · 방향 지정 (필수)

```cisco
interface <INTERFACE>
 ip nat inside
!
interface <INTERFACE>
 ip nat outside
```

검증:

```cisco
show ip nat statistics
show ip interface <INTERFACE>
show run | section ip nat
```

```properties
방향 지정 누락 = 변환 규칙이 완벽해도 NAT 미동작 = NAT 장애의 최다 원인
```

> `show ip nat statistics` 출력의 Inside interfaces / Outside interfaces 목록을 반드시 눈으로 확인하십시오.

---

## 5. 2단계 · 변환 대상 선별 ACL

```cisco
access-list <ACL-NUMBER> permit <NETWORK> <WILDCARD-MASK>
!
ip access-list extended <ACL-NAME>
 permit ip <NETWORK> <WILDCARD-MASK> any
```

검증:

```cisco
show access-lists
show access-lists <ACL-NUMBER>
show run | section ip nat
```

```properties
[NAT 대상 선별 ACL]
permit = 변환 대상으로 선정 (NAT 수행)
deny   = 변환하지 않음 (NAT 제외)

[Interface 적용 ACL]
permit = 통과 허용 / deny = 폐기
```

> 같은 키워드가 적용 위치에 따라 전혀 다른 의미를 가집니다. 상세 내용은 3-1-4. 🧩 ACL 통합 정리를 참고하십시오.

---

## 6. 3단계 · 변환 규칙 4종

Static NAT:

```cisco
ip nat inside source static <INSIDE-LOCAL-IP> <INSIDE-GLOBAL-IP>
ip nat inside source static 192.168.10.50 121.160.23.50
```

Dynamic NAT:

```cisco
ip nat pool <POOL-NAME> <START-IP> <END-IP> netmask <SUBNET-MASK>
ip nat pool <POOL-NAME> <START-IP> <END-IP> prefix-length <LENGTH>
ip nat inside source list <ACL-NUMBER> pool <POOL-NAME>
```

PAT (Overload):

```cisco
ip nat inside source list <ACL-NUMBER> pool <POOL-NAME> overload
ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
```

Port Forwarding (Static PAT):

```cisco
ip nat inside source static tcp <INSIDE-IP> <PORT> <OUTSIDE-IP> <PORT>
ip nat inside source static udp <INSIDE-IP> <PORT> <OUTSIDE-IP> <PORT>
ip nat inside source static tcp <INSIDE-IP> <PORT> interface <INTERFACE> <PORT>
!
ip nat inside source static tcp 192.168.10.10 80 121.160.23.2 80
ip nat inside source static tcp 192.168.10.11 443 121.160.23.2 443
ip nat inside source static tcp 192.168.10.10 80 121.160.23.2 8080
```

삭제:

```cisco
no ip nat inside source static 192.168.10.50 121.160.23.50
no ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
no ip nat pool <POOL-NAME>
```

검증(공통):

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
```

```properties
Static Entry = 트래픽 없어도 표시, clear로 삭제되지 않음(설정 삭제 필요)
Pool 주소 수 < 동시 통신 호스트 수 → 배정 실패 및 NAT Miss 증가
Interface 기반 PAT 조건 = 해당 Interface에 실제 공인 IP + ip nat outside
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Router 관리 Port(SSH 22 등)를 Forwarding하면 원격 접속이 불가능해질 수 있으므로 Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 7. Outside NAT

```cisco
ip nat outside source static <OUTSIDE-GLOBAL-IP> <OUTSIDE-LOCAL-IP>
ip nat outside source list <ACL-NUMBER> pool <POOL-NAME>
```

검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
show ip route <IP-ADDRESS>
```

```properties
ip nat inside source  = 내부 호스트 주소 변환
ip nat outside source = 외부 호스트 주소 변환
```

> 대역 중복 통합처럼 특수한 요구가 있을 때만 사용하며, 방향별 라우팅 참조 주소가 달라지므로 적용 전 경로를 확인하십시오. 세부 동작은 플랫폼별 지원 여부 확인이 필요합니다.

---

## 8. NAT 예외 (VPN 병행)

```cisco
ip access-list extended NAT_TARGET
 deny   ip 10.10.0.0 0.0.255.255 10.20.0.0 0.0.255.255
 permit ip 10.10.0.0 0.0.255.255 any
!
ip nat inside source list NAT_TARGET interface <INTERFACE> overload
!
route-map NAT_EXEMPT permit 10
 match ip address NAT_TARGET
!
ip nat inside source route-map NAT_EXEMPT interface <INTERFACE> overload
```

검증:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
show crypto ipsec sa
```

```properties
VPN 대상 사설 대역 간 트래픽이 NAT됨
        ↓
Crypto ACL / Proxy ID 불일치 → Tunnel 미통과
```

> 여기서 `deny`는 차단이 아니라 변환 제외입니다. route-map 기반 NAT와 `overload` 조합의 세부 동작은 IOS 버전별로 다르므로 플랫폼별 지원 여부 확인이 필요하고, Packet Tracer는 route-map 기반 NAT를 지원하지 않을 수 있습니다. 관련 설계는 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화)를 참고하십시오.

---

## 9. 사설 대역 유출 차단 (ISP 방향)

```cisco
ip access-list standard ANTI_PRIVATE
 deny   10.0.0.0 0.255.255.255
 deny   172.16.0.0 0.15.255.255
 deny   192.168.0.0 0.0.255.255
 permit any
!
interface <INTERFACE>
 ip access-group ANTI_PRIVATE out
```

검증:

```cisco
show access-lists ANTI_PRIVATE
show ip interface <INTERFACE>
show run | section ip nat
```

```properties
이 ACL의 deny        = 실제 트래픽 차단
NAT 대상 ACL의 deny  = 변환 제외
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 10. Timeout 조정과 Table 해제

```cisco
ip nat translation timeout <SECONDS>
ip nat translation tcp-timeout <SECONDS>
ip nat translation udp-timeout <SECONDS>
ip nat translation icmp-timeout <SECONDS>
ip nat translation finrst-timeout <SECONDS>
ip nat translation syn-timeout <SECONDS>
ip nat translation dns-timeout <SECONDS>
!
no ip nat translation udp-timeout
```

선별 해제와 전체 해제:

```cisco
clear ip nat translation inside <INSIDE-GLOBAL-IP> <INSIDE-LOCAL-IP>
clear ip nat translation outside <OUTSIDE-LOCAL-IP> <OUTSIDE-GLOBAL-IP>
clear ip nat translation *
```

검증:

```cisco
show ip nat translations verbose
show ip nat statistics
show run | section ip nat
```

```properties
Timeout 과소 = 유지되어야 할 세션 조기 종료
Timeout 과대 = Entry 누적, 메모리·Port 자원 부담

Dynamic Entry = clear로 삭제 가능
Static Entry  = clear로 삭제되지 않음 (no 명령 필요)
```

> ⚠️ `clear ip nat translation *`은 유지 중인 모든 Dynamic 변환 Entry를 삭제하므로 진행 중인 사용자 세션과 원격 관리 접속이 끊길 수 있습니다. 선별 삭제를 우선하고, 전체 삭제는 영향 범위를 확인한 뒤 Maintenance Window에 실행하십시오.

> 기본값과 설정 범위는 플랫폼별로 다르며, Packet Tracer는 Timeout 명령을 지원하지 않거나 다르게 동작할 수 있습니다.

---

## 11. `show ip nat translations` 판독

```properties
Pro   Inside global        Inside local        Outside local       Outside global
---   ------------------   -----------------   -----------------   -----------------
---   121.160.23.50        192.168.10.50       ---                 ---
tcp   121.160.23.2:1500    192.168.10.10:1500  200.1.1.1:80        200.1.1.1:80
icmp  121.160.23.2:8       192.168.10.10:8     200.1.1.1:8         200.1.1.1:8
udp   121.160.23.2:53      192.168.10.20:53    8.8.8.8:53          8.8.8.8:53
```

```properties
Pro 열이 ---          = Static NAT (Protocol 무관 1:1)
Pro 열에 tcp/udp/icmp = Port 또는 Query ID로 구분되는 변환
Outside 열이 ---      = 아직 외부 세션 정보가 없는 Static Entry
Inside global의 :번호 = PAT가 할당한 Port 또는 ICMP Query ID

TCP/UDP → Source Port로 세션 구분
ICMP    → ICMP Query ID로 세션 구분
```

---

## 12. `show ip nat statistics` 확인 항목

```properties
Total active translations : 현재 활성 변환 수 (static / dynamic 구분 표시)
Outside interfaces        : ip nat outside 지정 목록
Inside interfaces         : ip nat inside 지정 목록
Hits                      : 기존 Entry 재사용 횟수
Misses                    : 새 Entry 생성 시도 또는 변환 실패
Dynamic mappings          : ACL·Pool·Interface 연결 정보
Pool 사용량                : 할당된 주소 수 및 남은 주소
```

```properties
Inside/Outside interfaces 비어 있음 → 방향 지정 누락
Misses 급증                        → Pool 고갈 또는 ACL 불일치
Total active translations 과다      → Timeout 또는 비정상 트래픽 검토
```

---

## 13. NAT · 라우팅 · ACL 처리 순서

```properties
[Inside → Outside]
Inside Inbound ACL (Inside Local 기준)
    ↓
Routing → NAT (Inside Local → Inside Global)
    ↓
Outside Outbound ACL (Inside Global 기준)

[Outside → Inside]
Outside Inbound ACL (Inside Global 기준)
    ↓
NAT (Inside Global → Inside Local) → Routing
    ↓
Inside Outbound ACL (Inside Local 기준)

Inside → Outside = 라우팅 후 NAT
Outside → Inside = NAT 후 라우팅
```

ACL 작성 기준:

```cisco
ip access-list extended FROM_INTERNET
 permit tcp any host 121.160.23.2 eq 80
 deny   ip any any log
!
interface <INTERFACE>
 ip access-group FROM_INTERNET in
```

검증:

```cisco
show access-lists FROM_INTERNET
show ip nat translations
show ip nat statistics
```

> 외부에서 들어오는 트래픽을 허용할 때는 목적지를 사설 IP가 아니라 공인 IP로 작성해야 합니다. 상세 예외 처리는 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리를 참고하십시오.

---

## 14. 라우팅 확인과 경로 선택 원칙

```cisco
ip route 0.0.0.0 0.0.0.0 <NEXT-HOP-IP>
!
show ip route
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
```

```properties
경로 선택 순서
[1] Longest Prefix Match  : 가장 구체적인 Prefix 우선
[2] Administrative Distance : 동일 Prefix일 때 비교
[3] Metric                : 동일 Prefix·동일 Protocol일 때 비교

AD만 단독 비교 = 잘못된 결론
반드시 Prefix 길이를 먼저 확인
```

> 실제 선택된 경로는 `show ip route <IP-ADDRESS>`로 확인하십시오. 정적 경로 설계는 2. 📍 Static Route (정적 라우팅)을 참고하십시오.

---

## 15. NAT-T (NAT Traversal)

```properties
IKE 협상        : UDP 500
NAT 감지 후 전환 : UDP 4500
ESP 캡슐화      : UDP 4500 Payload
ESP 직접 전송    : IP Protocol 50 (Port 없음)
AH             : IP Header 인증 포함 → NAT와 충돌

PAT + ESP 직접 = 세션 구분 불가로 실패 가능
PAT + NAT-T    = UDP 4500으로 통과 가능
NAT + AH       = 인증 실패 (권장하지 않음)
```

경로 허용:

```cisco
ip access-list extended VPN_ALLOW
 permit udp any host <IP-ADDRESS> eq 500
 permit udp any host <IP-ADDRESS> eq 4500
 permit esp any host <IP-ADDRESS>
```

검증:

```cisco
show access-lists VPN_ALLOW
show crypto isakmp sa
show crypto ipsec sa
show ip nat translations
```

> NAT-T 자동 감지 동작과 지원 범위는 플랫폼·IOS 버전에 따라 다르므로 확인이 필요하며, Packet Tracer는 NAT-T를 재현하지 않을 수 있습니다. 상세 내용은 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE)를 참고하십시오.

---

## 16. Debug와 부하 확인

```cisco
debug ip nat
debug ip nat detailed
!
no debug ip nat
no debug all
undebug all
```

검증:

```cisco
show processes cpu sorted
show logging
show ip nat statistics
```

> ⚠️ `debug ip nat`은 변환되는 모든 패킷에 로그를 생성하므로 트래픽이 많은 운영 장비에서 CPU 과부하와 Console 폭주로 장비가 응답하지 않을 수 있습니다. 대상 범위를 좁혀 짧게만 사용하고 즉시 해제하십시오.

`debug` 전에 먼저 시도할 순서:

```cisco
show ip nat translations
show ip nat statistics
show run | section ip nat
show access-lists
show ip route <IP-ADDRESS>
```

---

## 17. 안전한 변경 순서

```properties
[1] 현재 확인 : show run | section ip nat / show ip nat translations / show ip nat statistics
[2] 백업     : copy running-config flash:pre-nat-change.cfg
[3] 경로 확인 : show ip route <IP-ADDRESS>
[4] 적용     : 방향 지정 → ACL → Pool → 변환 규칙
[5] 검증     : 변환 Entry 생성 및 실제 서비스 테스트
[6] 저장     : copy running-config startup-config
```

Rollback과 재검증:

```cisco
configure terminal
no ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
no ip nat inside source static <INSIDE-LOCAL-IP> <INSIDE-GLOBAL-IP>
no ip nat pool <POOL-NAME>
end
!
show run | section ip nat
show ip nat translations
show ip nat statistics
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

## 18. 암기용 함정표

| 설정 또는 상황 | 실제 의미 |
|---|---|
| NAT 대상 ACL의 `permit` | 트래픽 허용이 아니라 변환 대상 선정 |
| NAT 대상 ACL의 `deny` | 차단이 아니라 변환 제외 |
| Interface 적용 ACL의 `deny` | 실제 트래픽 폐기 |
| `ip nat inside` 누락 | 규칙이 완벽해도 NAT 미동작 |
| `overload` 누락 | Dynamic NAT로 동작해 Pool 고갈 |
| PAT 환경 외부→내부 접속 | 매핑이 없어 불가 (Port Forwarding 예외) |
| Static NAT | 변환만 수행, 접속 허용 정책은 ACL이 별도 |
| `show ip nat translations` 비어 있음 | Dynamic은 트래픽 없으면 정상적으로 비어 있음 |
| Static NAT Entry | `clear` 명령으로 삭제되지 않음 |
| `clear ip nat translation *` | 진행 중 세션과 관리 접속 단절 위험 |
| Outside Inbound ACL | 변환 전이므로 공인 IP 기준 작성 |
| Inside Outbound ACL | 변환 후이므로 사설 IP 기준 작성 |
| Inside → Outside 순서 | 라우팅 후 NAT |
| Outside → Inside 순서 | NAT 후 라우팅 |
| 변환 Entry 생성 성공 | 통신 성공을 의미하지 않음 (Return 경로 필요) |
| AD만 비교한 경로 판단 | Longest Prefix Match를 먼저 확인해야 함 |
| Ping 성공 | TCP 서비스 정상을 의미하지 않음 |
| PAT + IPsec ESP 직접 | Port가 없어 통과 실패 가능, NAT-T 필요 |
| NAT + AH | IP Header 인증 충돌로 실패 |
| VPN 대상 트래픽 NAT | Crypto 정책 불일치로 Tunnel 통과 실패 |
| `debug ip nat` 상시 실행 | CPU 과부하와 장비 응답 불가 위험 |
| Packet Tracer 동작 | route-map NAT·NAT-T·Timeout 등은 실장비와 다를 수 있음 |

---

## 19. 빠른 장애 진단 순서

```properties
ip nat inside 확인
    ↓
ip nat outside 확인
    ↓
show ip nat statistics의 Interface 목록 확인
    ↓
NAT 대상 ACL permit 범위 확인
    ↓
변환 규칙과 overload 유무 확인
    ↓
show ip nat translations에 Entry 생성 확인
    ↓
Entry 없음 → 방향·ACL·규칙·내부측 라우팅 재확인
Entry 있음 → Return 경로 확인 (show ip route <IP-ADDRESS>)
    ↓
Outside Inbound ACL을 공인 IP 기준으로 확인
    ↓
Inside Outbound ACL을 사설 IP 기준으로 확인
    ↓
목적지 서버·Firewall·서비스 Port 확인
    ↓
단말 IP·Gateway·DNS 확인
```

---

## 20. 검증 명령어 모음

```cisco
show ip nat translations
show ip nat translations verbose
show ip nat statistics
show run | section ip nat
show access-lists <ACL-NUMBER>
```

```cisco
show ip interface brief
show ip interface <INTERFACE>
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
show processes cpu sorted
show logging
```

---

> 📌 **핵심 요약**
> - 방향 지정: `ip nat inside` / `ip nat outside`
> - Static NAT: `ip nat inside source static <INSIDE-LOCAL-IP> <INSIDE-GLOBAL-IP>`
> - Dynamic NAT: `ip nat pool <POOL-NAME> ...` + `ip nat inside source list <ACL-NUMBER> pool <POOL-NAME>`
> - PAT (Pool): `ip nat inside source list <ACL-NUMBER> pool <POOL-NAME> overload`
> - PAT (Interface): `ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload`
> - Port Forwarding: `ip nat inside source static tcp <INSIDE-IP> <PORT> <OUTSIDE-IP> <PORT>`
> - Outside NAT: `ip nat outside source static <OUTSIDE-GLOBAL-IP> <OUTSIDE-LOCAL-IP>`
> - Timeout: `ip nat translation tcp-timeout <SECONDS>`
> - 변환 확인: `show ip nat translations` / `show ip nat statistics` / `show run | section ip nat`
> - 변환 해제: `clear ip nat translation inside ...` (선별) / `clear ip nat translation *` (세션 단절 위험)
> - NAT 대상 ACL의 `permit`은 허용이 아니라 변환 대상 선정이며 `deny`는 변환 제외다.
> - Inside→Outside는 라우팅 후 NAT, Outside→Inside는 NAT 후 라우팅이다.
> - 경로 선택은 Longest Prefix Match → AD → Metric 순서로 판단한다.
> - `debug ip nat`은 CPU 부하가 크므로 범위를 좁혀 짧게 쓰고 즉시 해제한다.
> - 관련: 5-2. 🔀 NAT & PAT (주소 변환) · 5-2-1. 🧩 NAT & PAT 통합 정리 · 5-2-2. 🚑 NAT & PAT 트러블슈팅 치트시트 · 5-2-3. ⚡ NAT & PAT 명령어 퀵 레퍼런스 · 5-1. 💾 DHCP (동적 IP 할당) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리
