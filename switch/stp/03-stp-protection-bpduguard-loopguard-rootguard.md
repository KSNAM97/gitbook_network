# 🛡️ STP 보호 기술 (BPDUguard / BPDUfilter / LoopGuard / RootGuard)

> **Tag:** #network #stp #security #bpduguard #loopguard #rootguard
> **핵심 요약:** STP 안정성과 토폴로지를 보호하는 4대 방어 기능

---

## 1. 🎯 핵심 기술 개념 (Concept)

### BPDUguard와 RootGuard의 차이

**BPDUguard**는 BPDU가 들어오면 안 되는 포트(단말/PortFast 포트)에서 **BPDU 수신 시 포트를 err-disable로 완전 차단**한다. 반면 **RootGuard**는 정상 스위치는 허용하되, **기존 Root보다 우선하는(Superior) BPDU가 들어올 때만** 해당 포트를 root-inconsistent로 차단해 **Root-bridge 자리를 보호**한다.

한 줄로 요약하면 BPDUguard는 "여긴 스위치 오면 안 돼"(Access 포트 보호)이고, RootGuard는 "너는 Root가 될 수 없어"(토폴로지 안정성)다. RootGuard는 기존 Root와 연결된 포트를 **제외한 나머지 하위 포트**에 설정한다.

### LoopGuard의 필요성과 BPDU 유실 시나리오

Block 포트는 BPDU가 안 들어오면 "장애가 복구됐다"고 오판해 Forwarding으로 전환하는데, **단방향 링크 장애 등으로 BPDU만 유실된 경우 실제로는 Loop가 발생**한다. LoopGuard는 이 경우 포트를 loop-inconsistent(BKN)로 유지해 Loop를 원천 차단한다.

LoopGuard는 **진짜 장애(Max-age 초과)** 와 **BPDU만 유실된 경우**를 구분해, 진짜 장애면 정상 전환하고 유실이면 Blocking을 유지한다. **BPDUfilter**는 반대로 특정 포트로 **BPDU 송신 자체를 차단**하는 기능으로 단말 포트에 사용한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 BPDUguard (단말/PortFast 포트 보호)

```cisco
! 스위치가 연결되면 안 되는 포트에 설정 → BPDU 수신 시 err-disable
SW1(config)# interface fastethernet 0/24
SW1(config-if)# spanning-tree bpduguard enable

! 자동 복구 설정 (선택)
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 60      ! 60초 후 자동 복구
```
### 📌 BPDUfilter (단말 포트로 BPDU 송신 차단)

```cisco
! PC/Server/Router 등 BPDU가 불필요한 포트에 설정
SW3(config)# interface fastethernet 0/20
SW3(config-if)# spanning-tree bpdufilter enable
```

### 📌 LoopGuard (Block 포트에서 설정)

```cisco
! STP에 의해 Block된 포트에서 설정. BPDU 유실 시 Blocking 유지
SW1(config)# interface fastethernet 0/20
SW1(config-if)# spanning-tree guard loop
```

### 📌 RootGuard (하위 스위치 방향 포트에서 설정)

```cisco
! 기존 Root 연결 포트를 제외한 나머지 포트에 설정
SW2(config)# interface range fa0/22 , fa0/24
SW2(config-if)# spanning-tree guard root
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
Switch# show interface status                    ! err-disable 여부 확인 (BPDUguard)
Switch# show spanning-tree                        ! BKN* (LOOP_Inc / ROOT_Inc) 상태 확인
Switch# show spanning-tree inconsistentports      ! 불일치로 차단된 포트 목록
```

### 📊 보호 기능별 요약 (면접 단골 표)

|기능|설정 위치|트리거|결과|
|---|---|---|---|
|**BPDUguard**|단말/PortFast 포트|BPDU 수신|err-disable|
|**BPDUfilter**|단말 포트|-|BPDU 송신 차단|
|**LoopGuard**|Block 포트|BPDU 유실|loop-inconsistent (BKN*)|
|**RootGuard**|하위 방향 포트|Superior BPDU 수신|root-inconsistent (BKN*)|

### 🚨 장애 시나리오 1: LoopGuard로 포트가 계속 BKN 상태

- **증상:** `show spanning-tree`에서 `Fa0/20 Desg BKN*19 ... *LOOP_Inc` 표시, 통신 안 됨.
- **원인 파악:** 해당 포트로 BPDU가 수신되지 않아(BPDUfilter나 단방향 장애) LoopGuard가 Blocking을 유지 중. (정상적 보호 동작)
- **해결:** BPDU가 정상 수신되도록 원인(케이블/단방향 링크/상대 BPDUfilter) 제거. BPDU가 다시 들어오면 자동으로 정상 전환.

### 🚨 장애 시나리오 2: RootGuard로 포트가 ROOT_Inc 상태

- **증상:** 하위 스위치에서 `spanning-tree vlan 1 priority 0`을 설정해 Root를 뺏으려 하자, 상위 포트가 `*ROOT_Inc`로 차단됨.
- **원인 파악:** RootGuard가 Superior BPDU를 감지해 Root-bridge 보호를 위해 포트를 차단 (정상 동작).
- **해결:** 하위 스위치의 잘못된 Priority 설정을 원복하면, Superior BPDU가 사라져 포트가 자동 복구됨.

---

> 관련: 3-1. 🌳 STP 기본 원리 & Port 상태 & Convergence · 3-2. ⚡ Fast 계열 (PortFast, UplinkFast, BackboneFast) · 3-4. 🚀 RSTP & MSTP (초대형망 STP) · 3-5. 🧩 STP·RSTP·MSTP 통합 정리 · 3-6. 🚑 STP·RSTP·MSTP 트러블슈팅 치트시트 · 3-7. ⚡ STP·RSTP·MSTP 퀵 레퍼런스