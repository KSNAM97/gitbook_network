# OSPF 트러블슈팅 치트시트

> **Tag:** #Routing #OSPF #Troubleshooting #Adjacency #MTU #LSA  
> **핵심 요약:** OSPF 장애는 **인터페이스 → Hello → Neighbor State → LSDB → RIB → FIB** 순서로 확인한다. Neighbor State가 어디서 멈췄는지 먼저 확인하고, Full 상태라면 LSA 존재 여부와 더 좋은 경로의 존재를 확인한다.

---

## 1. 빠른 진단 순서

```text
① Interface Up/Up 확인
        ↓
② OSPF 적용 및 Area 확인
        ↓
③ Hello 수신 및 Neighbor State 확인
        ↓
④ MTU·인증·Timer·Network Type 확인
        ↓
⑤ LSDB에 Prefix가 존재하는지 확인
        ↓
⑥ Routing Table에서 더 좋은 경로 확인
        ↓
⑦ CEF/FIB와 실제 전달 경로 확인
```

```cisco
show ip interface brief
show ip ospf interface brief
show ip ospf neighbor
show ip ospf interface <interface>
show ip ospf database
show ip route ospf
show ip route <prefix>
show ip cef <prefix>
```

---

## 2. Neighbor State별 진단

| State | 주요 원인 | 확인 및 조치 |
|---|---|---|
| Down | Hello 미수신 | 인터페이스, ACL, Passive, OSPF 적용 확인 |
| Init | 단방향 Hello | 멀티캐스트/NBMA Mapping, L2 통신 확인 |
| 2-Way | DROTHER 간 정상 또는 DR 문제 | Neighbor 역할과 DR/BDR 존재 확인 |
| ExStart | MTU, 중복 RID | MTU 및 Router-ID 확인 |
| Exchange | MTU, DBD 오류 | 양단 MTU와 로그 확인 |
| Loading | LSR/LSU 손실 | 링크 손실, ACL, CPU 확인 |
| Full | 인접성 정상 | LSDB와 RIB 단계로 이동 |

---

## 3. 증상별 즉시 대응표

### 3-1. 인접성 문제

| 증상 | 가능 원인 | 조치 |
|---|---|---|
| Neighbor가 전혀 없음 | OSPF 미적용 | `show ip ospf interface brief` |
| Neighbor가 전혀 없음 | Passive Interface | `no passive-interface` |
| Neighbor가 전혀 없음 | Area 불일치 | 양단 Area 번호 통일 |
| Neighbor가 전혀 없음 | Hello/Dead 불일치 | 양단 Timer 확인 |
| Neighbor가 전혀 없음 | 인증 불일치 | 인증 Mode, Key ID, Key 확인 |
| Init 상태 유지 | 단방향 통신 | ACL, 멀티캐스트, NBMA Mapping 확인 |
| 2-Way 상태 | DROTHER 간 정상 | DR/BDR 관계인지 확인 |
| DR과도 2-Way | DR 미선출 또는 Type 문제 | Priority와 Network Type 확인 |
| ExStart/Exchange 반복 | MTU 불일치 | MTU 통일 |
| Neighbor가 반복 초기화 | 중복 Router-ID | Router-ID 변경 후 Process 재시작 |
| NBMA Neighbor 없음 | `neighbor` 또는 Mapping 누락 | Network Type에 맞게 설정 |

### 3-2. 경로 문제

| 증상 | 가능 원인 | 조치 |
|---|---|---|
| Full인데 Prefix 없음 | 원본 인터페이스가 OSPF에 미포함 | OSPF 적용 상태 확인 |
| LSDB에도 Prefix 없음 | 광고 라우터 설정 문제 | 원본 라우터의 Type 1/3/5/7 확인 |
| LSDB에는 있지만 RIB에 없음 | 더 낮은 AD 경로 존재 | `show ip route <prefix>` |
| External 경로 없음 | Stub Area | NSSA 전환 또는 설계 변경 |
| NSSA 외부 경로 없음 | 재분배 또는 Type 7 문제 | NSSA DB와 Redistribution 확인 |
| Inter-Area 경로 없음 | ABR, Area 0 연결 문제 | ABR 및 Backbone 상태 확인 |
| 요약 경로만 보임 | ABR/ASBR Summary | `area range`, `summary-address` 확인 |
| Loopback이 /32로 보임 | Loopback 기본 동작 | 필요 시 P2P Network Type 적용 |
| 경로 Cost가 예상과 다름 | Reference BW 불일치 | 전 라우터 설정 통일 |

---

## 4. 대표 장애 시나리오

### 시나리오 A. Neighbor가 표시되지 않음

```cisco
show ip interface brief
show ip ospf interface brief
show ip protocols
show running-config | section router ospf
show ip ospf interface <interface>
```

확인 항목:

```text
□ 인터페이스가 Up/Up인가?
□ 양단이 동일 Subnet인가?
□ 양단에 OSPF가 활성화됐는가?
□ Area 번호가 같은가?
□ Passive Interface가 아닌가?
□ Hello/Dead Timer가 같은가?
□ Authentication 설정이 같은가?
□ ACL이 OSPF Protocol 89를 차단하지 않는가?
```

### 시나리오 B. Init 상태에서 멈춤

Init은 상대방의 Hello는 받았지만, 수신된 Hello의 Neighbor List에 자신의 Router-ID가 없는 상태다.

```cisco
show ip ospf neighbor
show ip ospf interface <interface>
debug ip ospf hello
```

주요 원인:

- 단방향 L2 연결
- NBMA Mapping 누락
- Multicast Replication 문제
- ACL 또는 방화벽이 한 방향 OSPF 패킷 차단
- Frame Relay `broadcast` Mapping 누락

### 시나리오 C. 2-Way에서 멈춤

```cisco
show ip ospf neighbor
show ip ospf interface <interface>
```

판단 기준:

- DROTHER ↔ DROTHER가 2-Way: **정상**
- DROTHER ↔ DR/BDR가 2-Way: 추가 점검 필요
- 세그먼트에 DR/BDR가 없음: 모든 라우터의 Priority가 0인지 확인

```cisco
interface <interface>
 ip ospf priority 100
```

Priority 변경만으로는 기존 DR 선출이 즉시 바뀌지 않을 수 있다. DR 재선출은 인접성에 영향을 주므로 운영 환경에서는 신중하게 수행한다.

### 시나리오 D. ExStart/Exchange 반복

```cisco
show ip ospf neighbor
show ip ospf interface <interface>
show interface <interface>
debug ip ospf adj
```

주요 원인:

1. IP MTU 불일치
2. 중복 Router-ID
3. DBD Packet 손실
4. 불안정한 링크

근본 조치:

```cisco
interface <interface>
 ip mtu 1500
```

임시 우회:

```cisco
interface <interface>
 ip ospf mtu-ignore
```

> [!WARNING]
> `ip ospf mtu-ignore`는 DBD의 MTU 검사를 우회할 뿐 실제 데이터 패킷의 MTU 문제를 해결하지 않는다. 단편화, PMTUD 실패 또는 데이터 전달 장애가 남을 수 있다.

### 시나리오 E. Full인데 경로가 없음

```cisco
show ip ospf database
show ip ospf database router
show ip ospf database summary
show ip ospf database external
show ip ospf database nssa-external
show ip route <prefix>
show ip route ospf
```

진단 순서:

1. LSDB에 Prefix 관련 LSA가 있는지 확인
2. 없다면 원본 라우터의 OSPF 포함 여부 확인
3. Area Summary 또는 External Summary 여부 확인
4. Stub/NSSA 정책 확인
5. RIB에 Static, Connected, EIGRP 등 더 선호되는 경로가 있는지 확인
6. 동일 Prefix Length 기준으로 AD와 Metric 비교

> [!IMPORTANT]
> **Area 불일치 상태에서는 일반적으로 Neighbor가 Full까지 형성되지 않는다.** Full인데 경로가 없는 문제를 Area 불일치로 진단하지 않는다.

### 시나리오 F. External 경로가 없음

```cisco
show ip ospf database external
show ip ospf database nssa-external
show ip ospf border-routers
show ip ospf | include Area
show running-config | section router ospf
```

확인 항목:

- 재분배 명령에 `subnets`가 있는가?
- Route-map 또는 Prefix-list가 차단하지 않는가?
- 해당 Area가 Stub인가?
- NSSA라면 Type 7 LSA가 생성되는가?
- NSSA ABR에서 Type 7 → Type 5 변환이 이뤄지는가?
- ASBR까지의 경로가 존재하는가?

### 시나리오 G. Router-ID 변경이 적용되지 않음

```cisco
router ospf 100
 router-id 1.1.1.1
```

유지보수 시간에 다음 명령을 사용한다.

```cisco
clear ip ospf process
```

> [!WARNING]
> OSPF Process를 초기화하면 모든 인접성이 재형성되고 경로가 일시적으로 제거될 수 있다.

---

## 5. Debug 명령어

```cisco
debug ip ospf adj
debug ip ospf hello
debug ip ospf packet
debug ip ospf events
undebug all
```

운영 장비에서는 CPU와 로그 발생량을 고려하여 제한적으로 사용한다.

---

## 6. 변경 전후 체크리스트

### 변경 전

```text
□ 현재 Neighbor와 DR/BDR 상태 저장
□ LSDB와 Routing Table 저장
□ OSPF 설정 및 Interface 설정 백업
□ 영향받는 Area와 Neighbor 확인
□ 롤백 명령 준비
```

### 변경 후

```text
□ Neighbor State가 정상인가?
□ 필요한 Neighbor가 Full인가?
□ DROTHER 간 2-Way를 장애로 오인하지 않았는가?
□ LSDB에 필요한 LSA가 존재하는가?
□ Routing Table과 CEF에 경로가 설치됐는가?
□ Ping/Traceroute가 정상인가?
```

---

## 7. 핵심 요약

- 첫 명령은 `show ip ospf neighbor`
- Down/Init은 Hello와 단방향 통신 확인
- DROTHER 간 2-Way는 정상
- ExStart/Exchange는 MTU와 중복 Router-ID 확인
- Area 불일치는 인접성 실패 원인
- Full인데 경로가 없으면 **LSDB → RIB → FIB** 순서로 확인
- `mtu-ignore`는 근본 해결책이 아님
- Stub에서 외부 재분배가 필요하면 NSSA 검토

> - 관련: 3-1. 🌲 OSPF (Open Shortest Path First) · 3-2. 🌲 OSPF 통합 정리 · 3-4. ⚡ OSPF 명령어 퀵 레퍼런스
