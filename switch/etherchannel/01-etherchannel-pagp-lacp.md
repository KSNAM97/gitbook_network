# EtherChannel (Link Aggregation)

> **Tag:** #network #etherchannel #pagp #lacp #link-aggregation #cisco
> **핵심 요약:** 다수의 물리 링크를 하나의 논리 링크로 묶어 대역폭 증대 + STP Loop 회피 + 이중화를 동시 달성

---

## 1. 핵심 기술 개념 (Concept)

### 다중 링크를 EtherChannel로 묶어야 하는 이유

스위치 간 링크를 그냥 여러 개 연결하면 **STP가 Loop를 막기 위해 1개만 남기고 나머지를 Blocking**하므로 대역폭이 늘지 않는다. EtherChannel은 여러 물리 링크를 **STP가 하나의 논리 포트(Port-channel)로 인식**하게 만들어, Loop 없이 모든 링크를 동시에 사용해 대역폭을 합산한다.

EtherChannel은 대역폭 증대뿐 아니라 **링크 이중화**도 제공한다. 묶인 링크 중 하나가 죽어도 STP 재수렴 없이 나머지 링크로 즉시 트래픽이 흐른다. Sub-interface(하나의 물리 → 다수 논리)와 정반대 개념으로, EtherChannel은 **다수 물리 → 하나 논리**다.

### PAgP와 LACP의 차이 및 모드 조합

**PAgP는 Cisco 전용**, **LACP(802.3ad)는 업계 표준**이다. 타사 장비와 연동해야 하면 무조건 LACP를 쓴다. 양쪽 모드는 **최소 한쪽이 능동(desirable/active)** 이어야 채널이 형성되며, **양쪽 다 수동(auto/passive)** 이면 형성되지 않는다.

PAgP는 지원 포트 최대 8개, LACP는 최대 16개 구성 가능(**8개 active + 8개 standby/hot-standby**)하다. `on` 모드는 협상 프로토콜 없이 강제로 묶는 방식으로 상대 설정을 확인하지 못해 Loop 위험이 있어 **실무 비권장**이다. SPAN(포트 미러링)이나 Port-security가 설정된 포트에는 EtherChannel을 구성할 수 없다.

### 모드 호환성 매트릭스 (반드시 암기)

| SW1 | SW2 | 채널 형성? |
|-----|-----|:----------:|
| desirable | desirable | ✅ |
| desirable | auto | ✅ |
| auto | auto | ❌ |
| active | active | ✅ |
| active | passive | ✅ |
| passive | passive | ❌ |

> ⚠️ **PAgP와 LACP는 서로 호환되지 않음.** 양쪽 스위치가 동일 프로토콜이어야 함.

---

## 2. 표준 설정 템플릿 (Configuration)

### L2 EtherChannel - PAgP (Cisco 전용)

```cisco
! === 논리 인터페이스(Port-channel)에 Trunk 설정 ===
SW1(config)# interface port-channel 12
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
!
! === 물리 포트를 채널 그룹에 묶기 ===
SW1(config)# interface range fa0/23 - 24
SW1(config-if)# channel-protocol pagp                ! 프로토콜 명시
SW1(config-if)# channel-group 12 mode desirable      ! 능동 협상 (상대는 desirable/auto)
```

### L2 EtherChannel - LACP (표준, 타사 연동)

```cisco
SW2(config)# interface port-channel 23
SW2(config-if)# switchport trunk encapsulation dot1q
SW2(config-if)# switchport mode trunk
!
SW2(config)# interface range fa0/21 - 22
SW2(config-if)# channel-protocol lacp
SW2(config-if)# channel-group 23 mode active          ! 능동 협상 (상대는 active/passive)
```

### L3 EtherChannel (라우팅용 논리 링크)

```cisco
SW1(config)# ip routing                               ! L3 라우팅 활성화 필수
!
SW1(config)# interface port-channel 12
SW1(config-if)# no switchport                         ! L3 채널로 전환 (VLAN 소속 해제)
SW1(config-if)# ip address 192.168.12.1 255.255.255.0
!
SW1(config)# interface range gi3/2 - 3
SW1(config-if)# no switchport                         ! 물리 포트도 L3로 (권장)
SW1(config-if)# channel-protocol pagp
SW1(config-if)# channel-group 12 mode desirable
```

> 💡 **설정 순서 실무 팁:** 물리 포트를 먼저 `channel-group`으로 묶으면 Port-channel 인터페이스가 자동 생성됩니다. 이후 Port-channel에 준 설정이 멤버 포트로 상속됩니다. **Trunk/IP 설정은 개별 물리포트가 아니라 Port-channel 인터페이스에** 해야 합니다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Switch# show etherchannel summary          ! 채널 상태 한눈에 확인 (가장 중요)
Switch# show etherchannel port-channel     ! 채널 멤버 상세
Switch# show interface port-channel 12     ! 논리 포트의 대역폭(BW)/상태
Switch# show interface trunk               ! Po가 trunk로 동작하는지 확인
Switch# show spanning-tree                  ! STP가 Po를 단일 포트로 인식하는지
```

### `show etherchannel summary` Flag 판독 (핵심)

```text
12  Po12(SU)   PAgP   Gi3/2(P) Gi3/3(P)
```

- **(SU)** : S=Layer2, U=in use → **정상 동작**
- **(RU)** : R=Layer3, U=in use → L3 채널 정상
- **(P)** : 멤버 포트가 채널에 **bundled(정상 묶임)**
- **(I)** : stand-alone → 협상 실패로 **독립 동작 (비정상)**
- **(s)** : suspended → **일시 중단 (설정 불일치)**
- **(D)** : down

### 장애 시나리오 1: 채널이 형성 안 됨 (포트가 I 또는 s 상태)

- **원인 파악 (순서대로):**
    1. **모드 조합 오류** — 양쪽 다 auto 또는 passive면 형성 안 됨. 최소 한쪽은 능동으로.
    2. **프로토콜 불일치** — 한쪽 PAgP, 한쪽 LACP면 절대 안 묶임.
    3. **멤버 포트 설정 불일치** — 속도/듀플렉스/트렁크 모드/allowed VLAN/native VLAN이 멤버 포트 간 다르면 suspended.
- **해결:** 양쪽 모드/프로토콜 통일 후, `show etherchannel summary`로 (P) 확인.

### 장애 시나리오 2: 채널은 up인데 트래픽이 한쪽 링크로만 쏠림

- **증상:** 링크는 다 살아있는데 특정 물리 링크만 부하가 높음.
- **원인 파악:** EtherChannel은 **로드밸런싱 해시** 기준(src-mac / dst-mac / src-dst-ip 등)에 따라 링크를 선택하는데, 트래픽 패턴이 편중되면 특정 링크에 몰림.
- **해결:** `show etherchannel load-balance`로 현재 알고리즘 확인 후, 트래픽 특성에 맞게 변경:
    
    ```cisco
    Switch(config)# port-channel load-balance src-dst-ip   ! IP 기반 분산 (일반적으로 균등)
    ```
    

### 장애 시나리오 3: L2 EtherChannel 구성 시도했는데 거부됨

- **원인 파악:** 해당 포트에 **SPAN(모니터링) 또는 Port-security**가 설정되어 있음.
- **해결:** 해당 기능을 먼저 제거한 후 EtherChannel 구성.

> 💡 **면접 한 줄 요약:** EtherChannel의 본질은 "STP를 속여서 다중 링크를 하나로 보이게 하는 것"이며, 실무에서는 타사 호환을 위해 LACP를 기본으로, 모드는 양쪽 최소 한쪽 능동(active/desirable)으로 맞춘다.


---

]