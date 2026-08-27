# 🚑 NAT & PAT 트러블슈팅 치트시트

> **Tag:** #Network #NAT #PAT #IPServices #Cisco  
> **핵심 요약:** NAT 장애는 **Interface 방향 지정 → 대상 선별 ACL → 변환 규칙 → 변환 Table → 라우팅 양방향 → ACL → 서비스** 순서로 확인합니다. `show ip nat translations`에 Entry가 생기는지가 1차 분기점이며, Entry가 있는데도 실패하면 NAT 문제가 아니라 경로·ACL·서버 문제일 가능성이 높습니다.

---

## 1. 🎯 공통 진단 순서

```properties
[1] 내부 Interface에 ip nat inside가 있는가?
[2] 외부 Interface에 ip nat outside가 있는가?
[3] NAT 대상 ACL이 해당 Source를 permit하는가?
[4] 변환 규칙(static / list+pool / list+interface)이 존재하는가?
[5] overload 유무가 의도와 일치하는가?
[6] show ip nat translations에 Entry가 생성되는가?
[7] 내부 → 외부 목적지 경로가 존재하는가?
[8] 외부 → Inside Global 주소로 오는 Return 경로가 존재하는가?
[9] Inbound/Outbound ACL이 변환 전·후 주소 기준으로 올바른가?
[10] 목적지 서버와 단말 Gateway·Firewall이 정상인가?
```

```properties
Entry 없음 → 방향 지정 / ACL / 변환 규칙 / 내부측 라우팅 확인
Entry 있음 → Return 경로 / ACL / 서버 서비스 / Firewall 확인
```

---

## 2. 🛠️ 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 | 조치 |
|---|---|---|---|
| 내부→외부 전부 불가, 변환 Entry 없음 | `ip nat inside`/`outside` 누락 | `show ip nat statistics` | 두 Interface에 방향 지정 |
| 일부 대역만 통신 불가 | NAT ACL 범위 누락 | `show access-lists` | 해당 대역 `permit` 추가 |
| 초기 몇 대만 되고 이후 실패 | `overload` 누락으로 Pool 고갈 | `show ip nat statistics` | `overload` 추가 또는 Pool 확장 |
| 외부→내부 서버 접속 불가 | PAT는 역방향 미지원 | `show ip nat translations` | Static NAT 또는 Port Forwarding |
| Port Forwarding했는데 접속 불가 | Outside Inbound ACL이 사설 IP 기준 | `show access-lists` | 공인 IP 기준으로 재작성 |
| 설정 변경했는데 이전 동작 유지 | 기존 변환 Entry 잔존 | `show ip nat translations` | 선별적 `clear` 후 재확인 |
| Ping은 되는데 웹 접속 불가 | ACL Port 차단 또는 서버 서비스 | `show access-lists` | TCP Port 허용·서버 확인 |
| VPN 구간 통신 불가 | VPN 트래픽이 NAT됨 | `show ip nat translations` | NAT ACL `deny`로 예외 처리 |
| 변환은 되는데 응답 없음 | Return 경로 부재 | `show ip route <IP-ADDRESS>` | ISP 경로·Default Route 확인 |
| 장비 CPU 급상승 | `debug ip nat` 방치 | `show processes cpu sorted` | `no debug all` 즉시 실행 |
| 일정 시간 후 세션 끊김 | Timeout 과소 설정 | `show run \| section ip nat` | Timeout 재조정 |
| Static 설정 시 충돌 오류 | 공인 IP 중복 사용 | `show run \| section ip nat` | 주소 정리 후 재설정 |
| Traceroute 중간 Hop 사설 IP | ICMP Error 처리·정책 차이 | `show ip nat translations` | 정상 가능성 검토 후 판단 |

---

## 3. 🔍 핵심 진단 명령어

```cisco
show ip nat translations
show ip nat translations verbose
show ip nat statistics
show run | section ip nat
show access-lists <ACL-NUMBER>
show ip interface <INTERFACE>
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
show processes cpu sorted
show logging
```

최후 수단:

```cisco
debug ip nat
no debug all
```

> ⚠️ `debug ip nat`은 변환되는 모든 패킷에 로그를 생성하므로 운영 장비에서 CPU 과부하와 Console 폭주를 유발할 수 있습니다. 대상 범위를 좁혀 짧게만 사용하고 즉시 해제하십시오.

---

## 4. 🚨 대표 장애 시나리오

### 시나리오 1. 내부 → 외부 통신이 전혀 안 되고 변환 Entry도 없음

**증상:** 내부 단말의 인터넷 Ping·웹 접속이 모두 실패하고 `show ip nat translations`이 비어 있습니다.

**원인 후보:** `ip nat inside` 누락, `ip nat outside` 누락, NAT 대상 ACL이 해당 Source를 `permit`하지 않음, 변환 규칙 자체 부재, Default Route 부재로 라우팅 단계에서 폐기, 잘못된 Interface에 방향 지정, Interface Down.

**확인:**

```cisco
show ip nat statistics
show run | section ip nat
show access-lists
show ip interface brief
```

```properties
Inside interfaces:  (비어 있음)
Outside interfaces: (비어 있음)
        ↓
변환 규칙이 있어도 NAT 미동작
```

**조치와 검증:**

```cisco
configure terminal
!
interface <INTERFACE>
 ip nat inside
!
interface <INTERFACE>
 ip nat outside
!
access-list <ACL-NUMBER> permit 192.168.10.0 0.0.0.255
ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
ip route 0.0.0.0 0.0.0.0 <NEXT-HOP-IP>
end
!
show ip nat translations
show ip nat statistics
show run | section ip nat
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 2. 일부 대역만 인터넷이 되지 않음

**증상:** 192.168.10.0/24는 정상인데 192.168.20.0/24 단말만 실패합니다.

**원인 후보:** NAT 대상 ACL에 해당 대역 누락, Wildcard Mask 오류, 상위 `deny`에 먼저 매칭되어 변환 제외, 해당 대역 SVI·Sub-interface에 `ip nat inside` 누락.

**핵심 판단:**

```properties
[NAT 대상 선별 ACL]
permit = 변환 대상으로 선정
deny   = 변환하지 않음 (차단이 아님)

ACL은 위에서 아래로 평가 → 상위 deny가 먼저 매칭되면 하위 permit은 평가되지 않음
```

**조치와 검증:**

```cisco
configure terminal
!
ip access-list extended NAT_TARGET
 permit ip 192.168.10.0 0.0.0.255 any
 permit ip 192.168.20.0 0.0.0.255 any
end
!
show access-lists NAT_TARGET
show ip nat translations
show ip nat statistics
```

---

### 시나리오 3. 처음 몇 대만 통신되고 나머지는 실패함

**증상:** 단말 몇 대는 정상인데 이후 접속하는 단말은 실패하고, 시간이 지나면 다른 단말이 성공합니다.

**원인:** `overload`가 없어 Dynamic NAT로 동작하며 Pool 주소가 1:1로 소진된 상태입니다.

```properties
Pool 주소 수 = 3, 동시 통신 호스트 = 30
        ↓
3대만 주소 배정 → 나머지는 NAT Miss로 실패
```

**확인:**

```cisco
show ip nat statistics
show ip nat translations
```

**조치와 검증:**

```cisco
configure terminal
no ip nat inside source list <ACL-NUMBER> pool <POOL-NAME>
ip nat inside source list <ACL-NUMBER> pool <POOL-NAME> overload
!
ip nat pool <POOL-NAME> 121.160.23.10 121.160.23.40 netmask 255.255.255.0
end
!
show ip nat translations
show ip nat statistics
show run | section ip nat
```

> ⚠️ 변환 방식 변경 후 기존 Entry가 남아 즉시 반영되지 않을 수 있습니다. `clear ip nat translation *`은 진행 중인 모든 세션을 끊을 수 있으므로 영향 범위를 확인하고 실행하십시오.

---

### 시나리오 4. 외부에서 내부 서버로 접속이 안 됨

**증상:** 인터넷에서 공인 IP로 접속하면 Router까지만 도달하고 내부 서버로 들어오지 않습니다.

**원인:** PAT와 Dynamic NAT는 **내부에서 시작된 세션에만 변환 Entry를 생성**하므로 외부에서 먼저 시작된 세션은 역변환 대상이 없습니다.

```properties
외부 → 내부 최초 접속
Static NAT           = 가능
Static PAT(Port F/W) = 지정 Port만 가능
Dynamic NAT / PAT    = 불가
```

**조치와 검증:**

```cisco
configure terminal
!
ip nat inside source static 192.168.10.50 121.160.23.50
!
ip nat inside source static tcp 192.168.10.10 80 121.160.23.2 80
ip nat inside source static tcp 192.168.10.11 443 121.160.23.2 443
end
!
show ip nat translations
show ip nat statistics
show run | section ip nat
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Router 관리 Port를 Forwarding 대상으로 지정하면 원격 접속이 불가능해질 수 있으므로 Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

### 시나리오 5. Port Forwarding을 설정했는데도 외부 접속이 안 됨

**원인 후보:** Outside Inbound ACL을 **사설 IP** 기준으로 작성, 필요한 TCP/UDP Port 미허용, 내부 서버 Gateway가 NAT Router가 아님, 서버 자체 Firewall 차단, 서버 서비스 미기동, 지정 공인 IP가 ISP에서 우리 쪽으로 라우팅되지 않음.

**핵심 판단:** 방향에 따라 ACL이 보는 주소가 다릅니다.

```properties
[Outside → Inside]
Outside Inbound ACL 검사   ← 변환 전 (공인 IP 기준)
    ↓
NAT 역변환 (Inside Global → Inside Local)
    ↓
Routing 결정
    ↓
Inside Outbound ACL 검사   ← 변환 후 (사설 IP 기준)
```

**잘못된 예:**

```cisco
ip access-list extended FROM_INTERNET
 permit tcp any host 192.168.10.10 eq 80
```

**올바른 예와 검증:**

```cisco
configure terminal
!
ip access-list extended FROM_INTERNET
 permit tcp any host 121.160.23.2 eq 80
 permit tcp any host 121.160.23.2 eq 443
 deny   ip any any log
!
interface <INTERFACE>
 ip access-group FROM_INTERNET in
end
!
show access-lists FROM_INTERNET
show ip nat translations
show ip nat statistics
```

---

### 시나리오 6. 설정을 바꿨는데 이전 NAT 동작이 계속됨

**증상:** 변환 규칙을 수정했는데 여전히 이전 공인 IP로 나가거나 이전 매핑이 유지됩니다.

**원인:** 기존 Dynamic 변환 Entry가 Table에 남아 Timeout까지 유지됩니다.

```properties
설정 변경 ≠ 기존 변환 Entry 즉시 삭제
```

**조치와 검증:**

```cisco
show ip nat translations verbose
!
clear ip nat translation inside <INSIDE-GLOBAL-IP> <INSIDE-LOCAL-IP>
clear ip nat translation *
!
show ip nat translations
show ip nat statistics
```

> ⚠️ `clear ip nat translation *`은 유지 중인 모든 Dynamic 변환 Entry를 삭제하므로 진행 중인 사용자 세션과 원격 관리 접속이 끊길 수 있습니다. 가능하면 선별 삭제를 사용하고, 전체 삭제는 Maintenance Window에 실행하십시오.

Static Entry는 `clear`로 제거되지 않으므로 설정에서 삭제해야 합니다.

```cisco
configure terminal
no ip nat inside source static 192.168.10.10 121.160.23.10
end
!
show run | section ip nat
```

---

### 시나리오 7. 변환 Entry는 생성되는데 응답이 돌아오지 않음

**증상:** `show ip nat translations`에 Entry가 정상 생성되지만 Ping 응답이 없습니다.

```properties
변환 Entry 생성 성공 = NAT 규칙과 방향 지정은 정상
응답 없음 → Return 경로 / 상대측 정책 / 중간 ACL 문제 가능성
```

**원인 후보:** ISP에서 Inside Global 주소로 오는 경로 부재, Default Route Next-hop 오류, 할당받지 않은 공인 IP 사용, 상대 서버 ICMP 차단, 중간 방화벽 차단, 비대칭 경로.

**확인:**

```cisco
show ip route <IP-ADDRESS>
show ip cef <IP-ADDRESS>
show ip nat translations
show access-lists
```

```properties
경로 판정 원칙
[1] Longest Prefix Match  : 가장 구체적인 Prefix 우선
[2] Administrative Distance : 동일 Prefix일 때 비교
[3] Metric                : 동일 Prefix·동일 Protocol일 때 비교
```

AD만 비교하면 `/32` 같은 구체적 경로가 선택되는 상황을 놓치므로, 반드시 `show ip route <IP-ADDRESS>`로 실제 선택 경로를 확인하십시오.

---

### 시나리오 8. Ping은 되는데 웹·업무 서비스만 안 됨

**원인 후보:** ACL이 ICMP만 허용하고 TCP Port 차단, Port Forwarding Port 오류, 서버 서비스 미기동, MTU·MSS 문제로 데이터 전송 단계 실패, Payload에 IP를 포함하는 프로토콜.

```properties
ICMP 성공     ≠ TCP/UDP 서비스 정상
NAT 변환 성공 ≠ Application 정상 동작
```

**확인:**

```cisco
show access-lists
show ip nat translations
```

FTP Active Mode, SIP, H.323처럼 Payload에 IP·Port를 포함하는 프로토콜은 NAT ALG 지원 여부에 따라 동작이 달라지며, 지원 범위는 플랫폼별 확인이 필요합니다.

---

### 시나리오 9. VPN(GRE/IPsec) 구간 통신이 안 됨

**증상:** Tunnel은 정상인데 본사·지사 사설 대역 간 통신이 실패하거나 Tunnel 자체가 형성되지 않습니다.

**원인:** VPN으로 보낼 사설 대역 간 트래픽이 PAT로 변환되어 상대측 Crypto 정책과 주소가 맞지 않습니다.

```properties
10.10.1.10 → 10.20.1.10 (VPN 대상)
        ↓
PAT로 Source가 공인 IP로 변환
        ↓
Crypto ACL / Proxy ID 불일치 → Tunnel 미통과 또는 상대측 폐기
```

**확인:** `show ip nat translations`에 VPN 대상 사설 주소가 변환된 Entry로 나타나면 예외 처리 누락입니다.

```cisco
show ip nat translations
show run | section ip nat
show crypto ipsec sa
show crypto isakmp sa
```

**조치와 검증:**

```cisco
configure terminal
!
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
end
!
show ip nat translations
show ip nat statistics
show run | section ip nat
```

여기서 `deny`는 차단이 아니라 **변환 제외**입니다. 상세 설계는 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE)와 5-3-4. 🔗 GRE-Over-IPsec (라우팅 암호화)를 참고하십시오.

> route-map 기반 NAT의 세부 동작과 `overload` 조합은 IOS 버전에 따라 차이가 있어 플랫폼별 지원 여부 확인이 필요하며, Packet Tracer는 route-map 기반 NAT를 실장비와 동일하게 지원하지 않을 수 있습니다.

---

### 시나리오 10. IPsec VPN이 PAT 구간을 통과하지 못함

**증상:** IKE 협상이 완료되지 않거나, SA는 형성됐는데 데이터가 전달되지 않습니다.

**원인:** ESP(IP Protocol 50)에는 TCP/UDP Port가 없어 PAT가 세션을 구분할 수 없습니다.

```properties
IKE      : UDP 500
NAT-T    : UDP 4500 (NAT 감지 시 전환)
ESP 직접 : Port 없음 → PAT 구분 불가
AH       : IP Header 인증 포함 → NAT와 근본 충돌
```

**조치와 검증:**

```cisco
configure terminal
!
ip access-list extended VPN_ALLOW
 permit udp any host <IP-ADDRESS> eq 500
 permit udp any host <IP-ADDRESS> eq 4500
 permit esp any host <IP-ADDRESS>
end
!
show access-lists VPN_ALLOW
show crypto isakmp sa
show ip nat translations
```

필요하면 VPN Peer에 1:1 Static NAT를 적용하고 AH 대신 ESP 사용을 검토하십시오.

> NAT-T 자동 감지와 UDP 4500 전환 동작은 플랫폼·IOS 버전에 따라 다르므로 확인이 필요하며, Packet Tracer는 NAT-T를 재현하지 않을 수 있습니다.

---

### 시나리오 11. 특정 시간이 지나면 세션이 끊김

**증상:** 장시간 유지해야 하는 세션이 일정 시간 후 단절되거나, 반대로 NAT Entry가 과도하게 누적됩니다.

**원인 후보:** Timeout이 세션 특성보다 짧음, 반대로 지나치게 길어 Entry 누적, Keepalive 없는 Application, PAT Port 자원 고갈.

**조치와 검증:**

```cisco
show ip nat translations verbose
!
configure terminal
ip nat translation timeout <SECONDS>
ip nat translation tcp-timeout <SECONDS>
ip nat translation udp-timeout <SECONDS>
ip nat translation icmp-timeout <SECONDS>
ip nat translation finrst-timeout <SECONDS>
end
!
show run | section ip nat
show ip nat statistics
```

```properties
Timeout 과소 = 정상 세션 조기 종료
Timeout 과대 = Entry 누적, 메모리·Port 자원 부담
```

> 기본값과 설정 범위는 플랫폼별로 다르므로 변경 전 현재 값을 기록하십시오. Packet Tracer는 Timeout 관련 명령을 지원하지 않거나 다르게 동작할 수 있습니다.

---

### 시나리오 12. NAT 설정 후 장비 CPU가 급상승함

**원인 후보:** `debug ip nat` 방치, Entry 과다 누적, PAT Port 자원 경합, NAT ACL이 지나치게 넓어 모든 트래픽 평가, 비정상 트래픽으로 대량 Entry 생성.

**즉시 조치와 확인:**

```cisco
no debug all
!
show processes cpu sorted
show ip nat statistics
show logging
```

> ⚠️ `debug ip nat`은 트래픽이 많은 운영 장비에서 CPU 과부하와 Console 폭주를 유발해 장비가 응답하지 않을 수 있습니다. 원격 접속 중이라면 Console 확보 후에만 사용하십시오.

후속으로 NAT 대상 ACL 범위를 필요한 대역으로 축소하고, Timeout 조정으로 Entry 누적을 완화하며, 비정상 Source 대역 차단을 검토하십시오.

---

### 시나리오 13. NAT는 정상인데 단말만 인터넷이 안 됨

**원인 후보:** 단말 IP·Subnet Mask·Gateway 오류, DNS 오류, DHCP가 잘못된 Gateway·DNS 배포, 단말이 다른 VLAN에 속함, Access Port VLAN 불일치.

```properties
Router에서 외부 Ping 성공 + 단말에서 실패
→ NAT보다 단말·L2·Gateway·DNS 문제 우선 검토
```

**확인:**

```cisco
show ip nat translations
show ip interface brief
show ip dhcp binding
show ip arp
```

IP는 되는데 이름 해석만 실패한다면 DNS 문제이므로 공인 IP로 직접 Ping해 구분하십시오. 주소 배포 문제는 5-1. 💾 DHCP (동적 IP 할당)을 참고하십시오.

---

### 시나리오 14. Static NAT 설정 시 충돌 오류가 발생함

**증상:** Static 변환 설정 중 이미 사용 중인 주소라는 오류가 표시되거나 설정이 반영되지 않습니다.

**원인 후보:** 지정한 Inside Global 주소를 다른 Static 규칙이 사용 중, 해당 공인 IP가 Interface 주소와 동일, 동일 Inside Local에 다른 매핑 존재, 기존 Dynamic Entry가 주소 점유.

**조치와 검증:**

```cisco
show run | section ip nat
show ip interface brief
!
configure terminal
no ip nat inside source static 192.168.10.10 121.160.23.10
ip nat inside source static 192.168.10.10 121.160.23.11
end
!
clear ip nat translation inside <INSIDE-GLOBAL-IP> <INSIDE-LOCAL-IP>
!
show ip nat translations
show ip nat statistics
show run | section ip nat
```

> ⚠️ `clear ip nat translation *`는 기존 세션을 끊을 수 있으므로 선별 삭제를 우선 검토하십시오.

---

### 시나리오 15. Traceroute 결과가 이상하게 보임

**증상:** 중간 Hop이 사설 IP로 표시되거나 `* * *`가 연속으로 나타납니다.

**원인 후보:** 중간 Router가 ICMP Time Exceeded를 자신의 사설 Interface 주소로 생성, ICMP Error가 ACL에서 차단, Rate-limit으로 ICMP Error 생성 제한, NAT의 ICMP Error Payload 처리 방식 차이, 목적지가 Probe에 무응답. PAT는 ICMP를 Query ID로 구분하며, ICMP Error는 원본 Header를 Payload에 포함하므로 NAT가 내부 주소까지 수정해야 정상 표시됩니다.

```properties
Traceroute 중간 * 표시 ≠ 해당 구간 통신 단절 확정
Traceroute 이상 → ICMP 정책 문제 가능성을 먼저 검토
```

**확인:**

```cisco
show ip nat translations
show access-lists
show ip route <IP-ADDRESS>
```

실제 서비스 가능 여부는 목적지 Port로 직접 확인하는 것이 정확합니다.

> Packet Tracer의 Traceroute와 ICMP Error 표시는 실장비와 다를 수 있으므로 학습 결과를 그대로 운영에 적용하지 마십시오.

---

## 5. 🔧 안전한 변경 및 복구 절차

```properties
[1] 현재 확인 : show run | section ip nat / show ip nat translations / show ip nat statistics
[2] 백업     : Configuration 저장 및 현재 변환 Entry 출력 보관
[3] 경로 확인 : show ip route / show ip route <IP-ADDRESS>
[4] 영향 검토 : 관리 세션 및 서비스 세션 단절 가능성 평가
[5] 적용     : 방향 지정 → ACL → Pool → 변환 규칙 순서
[6] 검증     : 변환 Entry 생성 및 실제 서비스 테스트
[7] 저장     : copy running-config startup-config
[8] Rollback : 실패 시 준비된 절차로 즉시 원복
```

변경 전 수집과 백업:

```cisco
show run | section ip nat
show ip nat translations
show ip nat statistics
show access-lists
!
copy running-config flash:pre-nat-change.cfg
```

Rollback과 재검증:

```cisco
configure terminal
no ip nat inside source list <ACL-NUMBER> interface <INTERFACE> overload
no ip nat inside source static tcp 192.168.10.10 80 121.160.23.2 80
no ip nat pool <POOL-NAME>
end
!
show run | section ip nat
show ip nat translations
show ip nat statistics
```

> ⚠️ 원격 변경 시 관리 연결이 끊길 수 있습니다. Console 또는 OOB 접근과 Rollback을 준비하십시오.

---

> 📌 **핵심 요약**
> - NAT 장애는 `ip nat inside`/`ip nat outside` 방향 지정부터 확인한다.
> - `show ip nat statistics`의 Interface 목록이 비어 있으면 규칙이 있어도 변환되지 않는다.
> - 변환 Entry 유무가 1차 분기점이며, Entry가 있으면 NAT 밖의 문제를 본다.
> - NAT 대상 ACL의 `permit`은 변환 대상 선정이고 `deny`는 변환 제외이며 차단이 아니다.
> - 공인 IP가 1개인데 `overload`가 없으면 몇 대만 통신되고 나머지는 실패한다.
> - 외부→내부 최초 접속은 Static NAT 또는 Port Forwarding만 가능하다.
> - Outside Inbound ACL은 공인 IP, Inside Outbound ACL은 사설 IP 기준으로 작성한다.
> - Inside→Outside는 라우팅 후 NAT, Outside→Inside는 NAT 후 라우팅이다.
> - 경로 판단은 Longest Prefix Match → AD → Metric 순서이며 AD 단독 비교는 금지한다.
> - VPN 대상 사설 대역 간 트래픽은 NAT에서 반드시 제외한다.
> - IPsec ESP는 Port가 없어 PAT를 통과하기 어렵고 NAT-T는 UDP 4500을 사용한다.
> - `clear ip nat translation *`은 진행 중인 세션과 관리 접속을 끊을 수 있어 선별 삭제를 우선한다.
> - `debug ip nat`은 CPU 과부하 위험이 크므로 범위를 좁혀 짧게 쓰고 즉시 해제한다.
> - 변경은 현재 확인 → 백업 → 경로 확인 → 적용 → 검증 → 저장 순서로 진행한다.
> - 관련: 5-2. 🔀 NAT & PAT (주소 변환) · 5-2-1. 🧩 NAT & PAT 통합 정리 · 5-2-2. 🚑 NAT & PAT 트러블슈팅 치트시트 · 5-2-3. ⚡ NAT & PAT 명령어 퀵 레퍼런스 · 5-1. 💾 DHCP (동적 IP 할당) · 3-1-2. 🎯  Extended ACL & 라우팅 프로토콜 예외 처리
