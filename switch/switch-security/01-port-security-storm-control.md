# 🔐 Port-security & Storm-control

> **Tag:** #network #security #port-security #storm-control #err-disable #cisco
> **핵심 요약:** 액세스 포트에 붙는 장비(MAC)를 제한하고, 폭주 트래픽으로부터 스위치 CPU를 보호하는 2대 방어 기능

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Q. Port-security의 3가지 Violation 모드는 실무에서 어떻게 선택하나요?

- 👉 **핵심 답변:** **protect**(위반 트래픽 조용히 폐기), **restrict**(폐기 + 로그/카운터 기록), **shutdown**(포트를 err-disable로 완전 차단, 기본값)입니다. 실무에서는 **감사 추적이 필요하면 restrict**, **보안이 최우선이면 shutdown**을 쓰고, protect는 로그가 없어 사고 추적이 안 되므로 거의 쓰지 않습니다.
- **꼬리 질문 대비 설명:**
  - shutdown 모드로 err-disable된 포트는 자동 복구되지 않으므로, `shutdown → no shutdown` 수동 조치 또는 `errdisable recovery`로 자동 복구를 걸어야 합니다.
  - Port-security가 설정된 포트에는 **EtherChannel/SPAN을 함께 쓸 수 없습니다**.

### Q. Sticky 옵션은 왜 쓰고, 운영상 주의점은?

- 👉 **핵심 답변:** MAC을 일일이 수동 입력하는 대신, **포트에 처음 들어온 MAC을 자동 학습해 Secure MAC으로 등록**하는 기능입니다. 포트 수가 많은 액세스 계층에서 관리 부담을 크게 줄여줍니다.
- **꼬리 질문 대비 설명:**
  - Sticky로 학습된 MAC은 running-config에 기록되므로, `write memory`를 해야 재부팅 후에도 유지됩니다.
  - 주의: 처음 연결된 아무 장비나 학습되므로, **초기 배선이 정확한 상태에서 활성화**해야 엉뚱한 MAC이 고정되는 사고를 막습니다.

### Q. Storm-control은 왜 필요하고 임계치는 어떻게 설계하나요?

- 👉 **핵심 답변:** Broadcast Storm 등 폭주 트래픽이 발생하면 스위치 CPU가 100%까지 치솟아 정상 통신이 마비됩니다. Storm-control은 포트별로 **Upper(차단 시작)/Lower(재개) 임계치**를 걸어, 임계치 초과 트래픽을 차단하거나 포트를 err-disable로 전환해 스위치를 보호합니다.
- **꼬리 질문 대비 설명:**
  - 임계치는 **%(대역폭 비율)** 또는 **pps(초당 패킷 수)** 로 지정 가능. 일반적으로 **Broadcast**에 우선 적용.
  - Upper/Lower를 나눠 설정하면 임계치 부근에서 껐다 켜졌다 하는 플래핑(flapping)을 방지합니다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Port-security (수동 MAC 지정)

```cisco
SW1(config)# interface fastethernet 0/1
SW1(config-if)# switchport mode access                      ! 반드시 access 포트여야 함
SW1(config-if)# switchport port-security maximum 1          ! 허용 MAC 최대 개수
SW1(config-if)# switchport port-security mac-address 00D0.BC6D.42E3   ! 허용할 MAC 지정
SW1(config-if)# switchport port-security violation shutdown ! 위반 시 err-disable
SW1(config-if)# switchport port-security                    ! 기능 활성화 (마지막에)
```


### 📌 Port-security Sticky (자동 학습)

```cisco
SW1(config)# interface fastethernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address sticky ! 첫 MAC 자동 학습·고정
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security
```

### 📌 Storm-control (Broadcast/Unicast 임계치)

```cisco
! === % 기반: CPU 30% 초과 시 차단 (Upper/Lower 동일) ===
SW1(config)# interface fastethernet 0/1
SW1(config-if)# storm-control broadcast level 30

! === Upper 30% 차단, Lower 20% 이하면 재개 (플래핑 방지) ===
SW1(config-if)# storm-control broadcast level 30 20

! === pps 기반: 초당 300개 초과 차단, 100개 이하면 재개 ===
SW1(config-if)# storm-control broadcast level pps 300 100

! === 임계치 초과 시 포트를 err-disable로 전환 ===
SW1(config-if)# storm-control broadcast level 30
SW1(config-if)# storm-control action shutdown
```

### 📌 err-disable 자동 복구

```cisco
! 위반으로 err-disable된 포트를 일정 시간 후 자동 복구
SW1(config)# errdisable recovery cause psecure-violation   ! Port-security 원인
SW1(config)# errdisable recovery cause storm-control        ! Storm-control 원인
SW1(config)# errdisable recovery interval 60                ! 60초 후 자동 복구
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
Switch# show port-security                        ! 전체 포트 보안 상태 요약
Switch# show port-security interface fa0/1        ! 특정 포트 상세 (위반 카운터 등)
Switch# show port-security address                 ! 학습/등록된 Secure MAC 목록
Switch# show storm-control broadcast               ! Storm-control 임계치/현재값
Switch# show interfaces status err-disabled        ! err-disable된 포트 확인
```

### 📊 `show port-security interface` 판독 포인트

- **Port Status: Secure-up** → 정상 / **Secure-shutdown** → 위반으로 차단됨
- **Violation Count** → 위반 발생 횟수 (0이 정상)
- **Last Source Address** → 마지막으로 들어온 MAC (위반 시 범인 MAC 확인)

### 🚨 장애 시나리오 1: PC를 교체했더니 포트가 죽음 (Secure-shutdown)

- **증상:** 정상 PC를 새 것으로 바꿨는데 통신 불가, 포트 err-disable.
- **원인 파악:** Port-security에 **기존 PC의 MAC이 고정**되어 있어, 새 PC의 다른 MAC이 위반으로 처리됨. `show port-security interface`의 Last Source Address가 새 PC MAC.
- **해결:**
    
    ```cisco
    interface fa0/1
     shutdown
     no shutdown                                   ! err-disable 해제
    ! + 새 MAC 재등록 또는 sticky MAC 초기화
    Switch# clear port-security sticky interface fa0/1
    ```
    

### 🚨 장애 시나리오 2: Storm-control로 포트가 계속 다운됨

- **증상:** `storm-control action shutdown` 설정 후 포트가 반복적으로 err-disable.
- **원인 파악:** 실제 Broadcast Storm(Loop 등) 또는 **임계치를 너무 낮게** 설정. `show storm-control`의 Current 값이 Upper에 근접.
- **해결:** 근본 원인(STP Loop, 오연결 NIC) 우선 제거 → 임계치를 트래픽 패턴에 맞게 상향 → `errdisable recovery`로 자동 복구 설정.

> 💡 **실무 연계 포인트:** Storm-control로 상위 트렁크 포트를 err-disable시키면 STP 재수렴이 필요한데, 이때 **UplinkFast나 RSTP**가 걸려 있으면 Blocking 포트가 즉시 Forwarding으로 전환되어 통신 단절을 최소화한다. (STP Fast 계열과 연계)

---

> 관련: 6-2. 🔎 SPAN (Port Mirroring) · 6-3. 🧩 Switch 보안 통합 정리 · 6-4. 🚑 Switch 보안 트러블슈팅 치트시트 · 6-5. ⚡ Switch 보안 명령어 퀵 레퍼런스