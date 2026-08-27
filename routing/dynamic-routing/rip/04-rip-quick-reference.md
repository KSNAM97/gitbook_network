# RIP 명령어 퀵 레퍼런스

> **Tag:** #Routing #RIP #QuickReference #RIPv2 #Authentication #CheatSheet
> **핵심 요약:** RIPv2 설정·인증·Passive Interface·Timer·검증 명령어를 빠르게 조회하기 위한 암기 카드. 기본 구성은 `version 2`와 `no auto-summary`이며, `network`는 Classic IOS에서 Classful Major Network 기준으로 입력한다.

---

## 1. RIPv2 기본 설정

```cisco
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 network 172.16.0.0
 passive-interface FastEthernet0/0
```

---

## 2. Passive Interface

특정 Interface:

```cisco
router rip
 passive-interface FastEthernet0/0
```

기본 Passive 후 Routing Link만 허용:

```cisco
router rip
 passive-interface default
 no passive-interface Serial1/0
 no passive-interface Serial1/1
```

```properties
Passive = 해당 Interface로 RIP Update 송신 억제
Passive ≠ 해당 Connected Network 광고 제거
```

---

## 3. RIPv2 MD5 인증

```cisco
key chain RIP_AUTH
 key 20110117
  key-string <RIP_SECRET>
!
interface Serial1/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP_AUTH
```

검증:

```cisco
show key chain
show running-config interface Serial1/0
debug ip rip
```

---

## 4. Unicast RIP Update

```cisco
router rip
 passive-interface Serial1/0
 neighbor 13.13.12.2
```

검증:

```cisco
ping 13.13.12.2
debug ip rip
```

---

## 5. Triggered RIP Extension

```cisco
interface Serial1/0
 ip rip triggered
```

제거:

```cisco
interface Serial1/0
 no ip rip triggered
```

```properties
용도 = 저속·비용 기반 Point-to-Point/On-demand WAN
효과 = Periodic Update 억제, 변화 시 Update
조건 = 양단 지원 및 동일 구성
```

> 일반 RIP의 Triggered Update는 기본 동작이며 별도로 켜는 기능이 아닙니다.

---

## 6. ⏱ RIP Timer

```cisco
router rip
 timers basic 30 180 180 240
```

| Timer | 기본값 | 역할 |
|---|---:|---|
| Update | 30초 | 주기적 Routing Update |
| Invalid | 180초 | 경로 무효 처리 |
| Holddown | 180초 | 불안정 정보 억제 |
| Flush | 240초 | 마지막 정상 Update부터 경로 제거 |

> Timer는 Domain 전체에서 일관되게 설계합니다.

---

## 7. 핵심 파라미터

| 항목 | 값 |
|---|---|
| Protocol | Distance Vector IGP |
| Algorithm | Bellman-Ford 계열 |
| Cisco AD | 120 |
| Metric | Hop Count |
| 최대 도달 Metric | 15 |
| Unreachable | 16 |
| Transport | UDP 520 |
| RIPv1 | Classful·Broadcast |
| RIPv2 | Classless·224.0.0.9 |
| RIPv2 인증 | Plain Text·MD5 |

---

## 8. 루프 방지

| 기능 | 동작 |
|---|---|
| Split Horizon | 배운 Interface로 재광고하지 않음 |
| Route Poison | 장애 경로를 Metric 16으로 광고 |
| Poison Reverse | 배운 방향으로 Metric 16 광고 |
| Triggered Update | Metric 변경 시 빠른 Update |
| Holddown | 불안정한 대체 정보 억제 |
| Maximum Metric | 16에서 Unreachable 처리 |

---

## 9. RIPv1 vs RIPv2

| 항목 | RIPv1 | RIPv2 |
|---|---|---|
| Update Mask | 없음 | 포함 |
| VLSM | 불가 | 가능 |
| 전송 | Broadcast | Multicast |
| 목적지 | 255.255.255.255 | 224.0.0.9 |
| 인증 | 없음 | Plain Text·MD5 |
| Route Tag | 없음 | 지원 |
| Next-hop Field | 없음 | 지원 |

---

## 10. 검증 명령어

```cisco
show ip protocols
show ip route rip
show ip rip database
show running-config | section router rip
show ip interface Serial1/0
show key chain
```

실시간 Update:

```cisco
debug ip rip
```

종료:

```cisco
undebug all
```

---

## 11. 연결성 검증

```cisco
show ip route <DESTINATION>
ping <DESTINATION>
ping <DESTINATION> source <SOURCE>
traceroute <DESTINATION> source <SOURCE>
```

---

## 12. 증상별 Quick Check

| 증상 | 먼저 확인 |
|---|---|
| RIP 경로 없음 | `show ip protocols` |
| Update 송신 없음 | Passive Interface |
| Update 수신 없음 | Version·ACL·Reachability |
| 인증 후 경로 삭제 | Key ID/String/Lifetime |
| 특정 Subnet 누락 | Auto Summary |
| Update는 받지만 RIB에 없음 | AD·Metric·Connected/Static |
| Metric 16 | 장애·Poison·Loop |
| Spoke 간 경로 없음 | Hub Split Horizon |
| Periodic Update가 사라짐 | `ip rip triggered` |
| 장애 경로 제거 지연 | Invalid/Flush Timer |

---

## 13. 핵심 주의사항

```properties
show ip protocol  ❌
show ip protocols ✅
```

```properties
version 2
+ no auto-summary
= 기본 RIPv2 Baseline
```

```properties
ip rip triggered
≠ 일반 Triggered Update 활성화
= Triggered Extension
```

```properties
Flush 240초
= Invalid 후 추가 240초가 아님
= 마지막 정상 Update부터 240초
```

```properties
Passive Interface
= Update 송신 억제
≠ Connected Prefix 광고 제거
```

---

> 📌 **핵심 요약**
> - 기본: `version 2` + `no auto-summary`
> - `network`는 Classic IOS에서 Major Network 기준
> - AD 120, UDP 520, Hop Count 최대 15
> - Metric 16은 Unreachable
> - 검증은 `show ip protocols`와 `debug ip rip`
> - Triggered RIP Extension은 특수 WAN에서만 사용
> - 관련: 1-1. ♟️ RIP (Routing Information Protocol) v1 & v2 · 1-2. ♟️ RIP 통합 정리 · 1-3. 🚑 RIP 트러블슈팅 치트시트 · 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol) · 3-1. 🌲 OSPF (Open Shortest Path First)
