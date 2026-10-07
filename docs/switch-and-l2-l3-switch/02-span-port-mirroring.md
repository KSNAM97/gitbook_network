# SPAN (Switched Port Analyzer / Port Mirroring)

> **Tag:** #network #span #rspan #mirroring #monitoring #cisco
> **핵심 요약:** 특정 포트의 트래픽을 모니터링 장비로 복사(미러링)하여 패킷 분석·감사에 활용

---

## 1. 핵심 기술 개념 (Concept)

### Q. 스위치 환경에서 왜 SPAN이 필요한가요? (허브와의 차이)

- 👉 **핵심 답변:** 허브는 모든 포트로 트래픽을 뿌려 아무 포트에서나 스니핑이 됐지만, **스위치는 MAC 기반으로 목적지 포트로만 전송**하므로 모니터링 장비를 아무 포트에 꽂아도 남의 트래픽이 보이지 않습니다. SPAN은 지정 포트의 송/수신 트래픽을 **모니터링 포트로 복사**해 이 문제를 해결합니다.
- **꼬리 질문 대비 설명:**
  - 방향 지정: **rx**(수신), **tx**(송신), **both**(양방향). IDS/포렌식은 보통 both.
  - Destination 포트는 미러링 전용이 되어 일반 통신은 하지 못함.

### Q. Local SPAN과 RSPAN(Remote SPAN)의 차이는?

- 👉 **핵심 답변:** **Local SPAN**은 감시 대상과 모니터링 장비가 **같은 스위치**에 있을 때, **RSPAN**은 **서로 다른 스위치**에 있을 때 씁니다. RSPAN은 트래픽을 전용 **RSPAN VLAN**에 실어 트렁크를 통해 원격 스위치까지 전달합니다.
- **꼬리 질문 대비 설명:**
  - RSPAN VLAN은 반드시 `remote-span`으로 지정하고, 경로상의 모든 스위치 트렁크에서 이 VLAN이 허용되어야 함.
  - **Reflector-port**는 미러 트래픽 복사를 소스 포트 CPU가 아닌 별도 포트의 CPU로 처리하게 하는 것으로, **미사용 포트**로 지정해야 함.

---

## 2. 표준 설정 템플릿 (Configuration)

### Local SPAN

```cisco
! Fa0/1의 양방향 트래픽을 Fa0/20(모니터링 장비)으로 복사
SW1(config)# monitor session 1 source interface fastethernet 0/1 both
SW1(config)# monitor session 1 destination interface fastethernet 0/20
```

### Remote SPAN (RSPAN)

```cisco
! === 감시 대상 스위치 (SW1) ===
SW1(config)# vlan 999
SW1(config-vlan)# remote-span                                ! RSPAN 전용 VLAN 지정
!
SW1(config)# monitor session 1 source interface fastethernet 0/1 both
SW1(config)# monitor session 1 destination remote vlan 999 reflector-port fastethernet 0/22
!                                                            ! reflector-port는 미사용 포트로

! === 모니터링 스위치 (SW2) ===
SW2(config)# vlan 999
SW2(config-vlan)# remote-span
!
SW2(config)# monitor session 1 source remote vlan 999
SW2(config)# monitor session 1 destination interface fastethernet 0/20
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Switch# show monitor session 1        ! 세션 타입/소스/목적지/방향 확인
Switch# show monitor                    ! 전체 SPAN 세션 목록
Switch# show vlan remote-span           ! RSPAN VLAN 지정 확인
```

### `show monitor session` 판독 포인트

- **Type**: Local Session / Remote Source Session / Remote Destination Session
- **Source Ports (Both/Rx/Tx)**: 미러링 대상 및 방향
- **Destination Ports** 또는 **Dest RSPAN VLAN**: 트래픽이 나가는 곳

### 장애 시나리오 1: 모니터링 장비에 트래픽이 안 잡힘

- **원인 파악:**
    1. 방향(rx/tx/both) 설정이 원하는 방향과 다름.
    2. Destination 포트가 다른 용도로 쓰이거나 링크 다운.
    3. Port-security/EtherChannel이 설정된 포트를 소스/목적지로 지정 (충돌).
- **해결:** `show monitor session`으로 소스/목적지/방향 재확인, 목적지 포트를 전용으로 비움.

### 장애 시나리오 2: RSPAN인데 원격 스위치에서 트래픽이 안 옴

- **원인 파악:**
    1. 양쪽 스위치의 **RSPAN VLAN 번호 불일치** 또는 `remote-span` 미지정.
    2. **경로상 트렁크에서 RSPAN VLAN이 allowed에 없음** (`switchport trunk allowed vlan`).
    3. Reflector-port를 이미 사용 중인 포트로 지정.
- **해결:** 양쪽 RSPAN VLAN 일치 확인 → 경로 트렁크에 해당 VLAN 허용 → reflector-port를 미사용 포트로 변경.

> 💡 **면접 한 줄 요약:** SPAN은 스위치의 MAC 기반 전송 특성 때문에 불가능해진 스니핑을 합법적으로 가능케 하는 미러링 기능이며, 같은 스위치면 Local SPAN, 원격이면 RSPAN VLAN을 통해 전달한다. 단, Port-security/EtherChannel과는 함께 쓸 수 없다.

---

> 관련: 6-1. 🔐 Port-security & Storm-control · 6-3. 🧩 Switch 보안 통합 정리 · 6-4. 🚑 Switch 보안 트러블슈팅 치트시트 · 6-5. ⚡ Switch 보안 명령어 퀵 레퍼런스