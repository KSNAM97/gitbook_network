# Switchport & Trunk (Trunk / DTP / Sub-interface)

> **Tag:** #network #trunk #dot1q #isl #dtp #inter-vlan
> **핵심 요약:** 하나의 링크로 다수 VLAN을 전송하는 Trunk와 자동협상(DTP), 라우터 온 어 스틱

---

## 1. 핵심 기술 개념 (Concept)

### 스위치 간 연결에 Trunk를 써야 하는 이유

Access 모드는 링크 하나당 VLAN 하나만 전송하므로, VLAN이 N개면 스위치 간 케이블도 N개가 필요해 물리 포트 낭비가 극심하다. **Trunk는 단일 링크로 모든 VLAN을 Tagging하여 전송**하므로, 스위치 간 백본을 효율적으로 설계할 수 있다.

Trunk는 프레임에 **4Byte의 802.1Q Tag(VLAN-ID 12bit 포함)**를 삽입해 어느 VLAN 소속인지 구분한다. **Native VLAN(기본 VLAN 1)**은 Tag 없이 전송되며, Dot1q는 Tag가 없는 프레임을 Native VLAN으로 처리하고, ISL은 이를 자신의 정보가 아닌 것으로 판단해 **Drop**한다. 이것이 Dot1q와 ISL의 가장 큰 차이점이다.

### 실무에서 DTP(자동 Trunk 협상)를 권장하지 않는 이유

DTP는 편리하지만 **보안 취약점(VLAN Hopping)**과 예측 불가능한 협상 결과를 낳는다. 실무 운영망에서는 DTP를 끄고(`switchport nonegotiate`) **모드를 수동으로 고정**하는 것이 정석이다.

`desirable`은 DTP를 송/수신하며 능동적으로 Trunk를 시도하고, `auto`는 수신만 하며 상대가 요청할 때만 Trunk가 된다. **auto-auto는 Trunk가 형성되지 않는다.** 협상 결과에서 `n-802.1q`, `n-isl`의 `n-`은 negotiated(협상됨)를 의미하며, 한쪽이 Access면 상대는 Trunk가 되지 않는다.

---

## 2. 표준 설정 템플릿 (Configuration)

### Trunk 표준 구성 (IEEE 802.1Q 권장)

```cisco
! === 표준 Trunk (Dot1q) : 양쪽 스위치 동일 적용 ===
Switch(config)# interface fastethernet 0/24
Switch(config-if)# switchport trunk encapsulation dot1q   ! 캡슐화를 표준 dot1q로 지정
Switch(config-if)# switchport mode trunk                  ! 모드를 Trunk로 고정

! === ISL Trunk (Cisco 전용, 레거시) ===
Switch(config-if)# switchport trunk encapsulation isl
Switch(config-if)# switchport mode trunk

! === 2950 시리즈 이하 : ISL 미지원 → encapsulation 명령 없이 바로 trunk ===
Switch(config-if)# switchport mode trunk                  ! 자동으로 dot1q 적용

### Allowed VLAN (Trunk 통과 VLAN 제어 = 보안/트래픽 최적화)

```cisco
! 기본값은 all(1-4094) 허용. 필요한 VLAN만 통과시켜 불필요한 Flooding 차단
interface fa0/23
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 1,10,20,30      ! 지정 VLAN만 허용 (기존 목록을 덮어씀)
!
 switchport trunk allowed vlan add 40          ! 기존 허용 목록에 40 추가
 switchport trunk allowed vlan remove 10       ! 기존 허용 목록에서 10 제거
 switchport trunk allowed vlan except 100      ! 100만 제외하고 나머지 전부 허용
```

### Inter-VLAN Routing : Router-on-a-Stick (Sub-interface)

```cisco
! === R1 : 물리 인터페이스 1개를 논리적으로 분할해 각 VLAN의 게이트웨이 역할 ===
interface fastethernet 0/0
 no shutdown                                    ! 물리 포트는 IP 없이 활성화만
!
interface fastethernet 0/0.10                   ! sub-interface 생성 (.VLAN번호 관례)
 encapsulation dot1q 10                         ! 이 서브인터페이스가 VLAN 10 태그 처리
 ip address 192.168.10.254 255.255.255.0        ! VLAN 10의 게이트웨이 IP
!
interface fastethernet 0/0.20
 encapsulation dot1q 20
 ip address 192.168.20.254 255.255.255.0
!
interface fastethernet 0/0.30
 encapsulation dot1q 30
 ip address 192.168.30.254 255.255.255.0

! === SW1 : 라우터와 연결된 포트는 반드시 Trunk로 구성 ===
interface fastethernet 0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Switch# show interface trunk       ! Trunk 상태, Encapsulation, Native VLAN, 허용 VLAN 확인 (핵심)
Switch# show dtp interface fa0/24  ! DTP 협상 상태 확인
Router# show ip route              ! 서브인터페이스가 Connected 라우트로 잡히는지 확인
```

### `show interface trunk` 판독 포인트

- `Mode` : on(수동) / desirable / auto
- `Encapsulation` : 802.1q, isl, `n-802.1q`(협상된 dot1q)
- `Status` : **trunking** 이어야 정상 (not-trunking이면 실패)
- `Native vlan` : 양쪽 스위치가 반드시 일치해야 함

### 장애 시나리오 1: Trunk가 형성되지 않음 (Status: not-trunking)

- **원인 파악:**
    1. 한쪽만 `mode trunk`, 다른 쪽은 `auto`/`access`인 경우. **auto-auto 또는 access가 낀 조합**은 협상 실패.
    2. 한쪽은 dot1q, 다른 쪽은 isl 등 **Encapsulation 불일치**.
- **해결:** 양쪽 모두 `switchport trunk encapsulation dot1q` + `switchport mode trunk`로 **수동 고정** 후 `show interface trunk`로 재확인.

### 장애 시나리오 2: Trunk는 됐는데 특정 VLAN만 통신 안 됨

- **원인 파악:**
    1. `show interface trunk`의 "Vlans allowed on trunk"에서 **해당 VLAN이 allowed 목록에 없음**. (`allowed vlan` 설정 시 실수로 덮어씀)
    2. **Native VLAN 불일치** → CDP가 `Native VLAN mismatch` 경고 로그 발생.
- **해결:** `switchport trunk allowed vlan add [VLAN]`으로 누락 VLAN 추가. Native VLAN을 양쪽 동일하게 맞춤.

### 장애 시나리오 3: Router-on-a-Stick 구성 후 VLAN 간 통신 불가

- **원인 파악:** 서브인터페이스의 `encapsulation dot1q` VLAN 번호와 스위치 VLAN이 불일치하거나, 스위치-라우터 연결 포트가 **Access로 남아있음**.
- **해결:** 스위치 연결 포트를 Trunk로 변경하고, PC들의 **게이트웨이가 서브인터페이스 IP로 지정**되었는지 확인.

---

> 관련: 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-3. 🧩 스위치 기본 & VLAN·Trunk 통합 정리 · 1-4. 🚑 스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트 · 1-5. ⚡ 스위치 기본 & VLAN·Trunk 퀵 레퍼런스 · 1-6. 🔑 Switch Password 복구 · 1-7. 🧩 Switch Password 복구 통합 정리 · 1-8. 🚑 Switch Password 복구 트러블슈팅 치트시트 · 1-9. ⚡ Switch Password 복구 퀵 레퍼런스