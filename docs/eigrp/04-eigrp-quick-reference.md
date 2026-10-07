# EIGRP 명령어 퀵 레퍼런스

> **Tag:** #Routing #EIGRP #QuickReference #DUAL #Authentication #CheatSheet
> **핵심 요약:** EIGRP 설정·인증·Stub·Summary·Split-horizon·검증 명령어를 빠르게 조회하기 위한 암기 카드. 기본 구성은 `no auto-summary` + `passive-interface default`이며, VLSM 환경에서는 Host Wildcard(`0.0.0.0`)로 정밀 지정이 권장된다.

---

## 1. 기본 설정

```cisco
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface Serial1/0
 no passive-interface Serial1/1
 network 13.13.1.1 0.0.0.0
 network 13.13.10.1 0.0.0.0
 network 150.1.13.1 0.0.0.0
```

---

## 2. Passive Interface

특정 Interface:

```cisco
router eigrp 100
 passive-interface FastEthernet0/0
```

기본 Passive 후 Routing Link만 허용 (권장):

```cisco
router eigrp 100
 passive-interface default
 no passive-interface Serial1/0
 no passive-interface Serial1/1
```

```properties
Passive = 해당 Interface로 Hello 미송신 → Neighbor 미형성
Passive ≠ 해당 Connected Network 광고 제거
```

---

## 3. EIGRP MD5 인증

```cisco
key chain EIGRP_AUTH
 key 20240101
  key-string <EIGRP_SECRET>
!
interface Serial1/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_AUTH
```

검증:

```cisco
show key chain
show running-config interface Serial1/0
show ip eigrp neighbors detail
```

---

## 4. Split-Horizon 비활성화 (FR Multipoint Hub)

```cisco
interface Serial1/0.123 multipoint
 no ip split-horizon eigrp 100
```

> Hub-and-Spoke의 Hub Multipoint Sub-if에서만 적용. Spoke나 P2P에는 적용하지 않는다.

---

## 5. Summary (인터페이스 단위)

```cisco
interface Serial1/0
 ip summary-address eigrp 100 13.13.0.0 255.255.0.0
```

> Summary 광고 시 해당 인터페이스에 Null0 Summary 경로 자동 생성 (AD 5). 상세 경로는 Topology Table에서 숨겨짐.

---

## 6. Stub 설정

```cisco
router eigrp 100
 eigrp stub connected summary
```

| 옵션 | 의미 |
|---|---|
| `connected` | Connected 경로만 광고 |
| `summary` | Summary 경로도 광고 |
| `static` | Static 재분배 경로도 광고 |
| `receive-only` | 광고 없이 수신만 |

> Stub 라우터는 Query에 즉시 INACCESSIBLE Reply를 반환 → SIA 방지.

---

## 7. Unequal-cost Load Balancing

```cisco
router eigrp 100
 maximum-paths 4
 variance 2
```

조건: 대체 경로 Metric ≤ Successor FD × Variance, AND Feasibility Condition 만족.

---

## 8. 핵심 파라미터

| 항목 | 값 |
|---|---|
| 프로토콜 | Hybrid (Distance-Vector + Link-State) |
| 알고리즘 | DUAL |
| Cisco AD (Internal) | 90 |
| Cisco AD (External) | 170 |
| Cisco AD (Summary Null0) | 5 |
| Metric 기준 | Bandwidth + Delay (기본 K1, K3) |
| Transport | IP Protocol 88 |
| Multicast | 224.0.0.10 |
| Hello (LAN) | 5초 |
| Hold (LAN) | 15초 |
| Hello (WAN/NBMA) | 60초 |
| Hold (WAN/NBMA) | 180초 |

---

## 9. 검증 명령어

```cisco
show ip eigrp neighbors
show ip eigrp neighbors detail
show ip eigrp interfaces detail
show ip eigrp topology
show ip eigrp topology all-links
show ip eigrp topology active
show ip route eigrp
show ip protocols
show running-config | section router eigrp
```

---

## 10. 연결성 검증

```cisco
show ip route <DESTINATION>
ping <DESTINATION>
ping <DESTINATION> source <SOURCE>
traceroute <DESTINATION> source <SOURCE>
```

---

## 11. 증상별 Quick Check

| 증상 | 먼저 확인 |
|---|---|
| Neighbor 없음 | AS 불일치, K값, Passive Interface |
| Hello 미송신 | `passive-interface` 확인 |
| Neighbor Uptime 반복 Reset | Link Loss, 인증 Key Lifetime, SIA |
| 경로 없음 (Neighbor 정상) | `show ip eigrp topology all-links` |
| FS 없음 | Feasibility Condition (`RD < Successor FD`) |
| SIA 발생 | Stub/Summary/WAN 품질 개선 |
| Spoke 간 경로 없음 | Hub Multipoint Split-horizon |
| Unequal LB 안 됨 | Variance, maximum-paths, FC 확인 |
| EIGRP External 경로 없음 | Seed Metric 5개 항목 확인 |

---

## 12. 핵심 주의사항

```properties
no auto-summary
= VLSM 환경 필수 (기본 활성화 상태)
```

```properties
network 13.13.1.1 0.0.0.0
= Host Wildcard로 정밀 지정 (권장)
network 13.0.0.0
= Classful Major Network (의도치 않은 If 포함 위험)
```

```properties
Hello/Hold 값 불일치
≠ Neighbor 형성 실패 조건
= 실제 Hello 손실 여부를 확인할 것
```

```properties
Feasibility Condition
= Neighbor RD < 현재 Successor FD
FS 없음 → Backup 없이 Query 확산 → SIA 위험
```

---

> 📌 **핵심 요약**
> - 기본: `no auto-summary` + `passive-interface default` + Host Wildcard `network`
> - AD 90 (Internal) / 170 (External) / 5 (Summary Null0)
> - DUAL: Topology Table에 FS 사전 계산 → 빠른 Failover
> - SIA 예방: Stub + Summary + 계층화
> - FR Multipoint Hub: `no ip split-horizon eigrp <AS>`
> - 관련: 2-1. 🐎  EIGRP (Enhanced Interior Gateway Routing Protocol) · 2-2. 🐎 EIGRP 통합 정리 · 2-3. 🚑 EIGRP 트러블슈팅 치트시트 · 1-4. ⚡ RIP 명령어 퀵 레퍼런스 · 3-4. ⚡ OSPF 명령어 퀵 레퍼런스
