# STP Fast 계열 (PortFast / UplinkFast / BackboneFast)

> **Tag:** #network #stp #portfast #uplinkfast #backbonefast #convergence
> **핵심 요약:** STP의 느린 Convergence(30~50초)를 단축하는 3가지 가속 기능

---

## 1. 핵심 기술 개념 (Concept)

### PortFast의 적용 대상과 위험성

PortFast는 **PC/서버처럼 Loop를 만들지 않는 단말 포트**를 Listening/Learning 없이 즉시 Forwarding으로 전환시킨다. 하지만 이 포트에 실수로 **스위치를 연결하면 STP가 우회되어 Loop가 발생**하므로, 반드시 **BPDUguard와 함께** 써야 한다.

3가지 Fast 기능의 용도 구분은 다음과 같다. **PortFast**는 단말 포트 즉시 활성화, **UplinkFast**는 RP 장애 시 Block 포트 즉시 승격, **BackboneFast**는 간접 링크 장애 시 Max-age 20초를 생략한다. PortFast 적용 포트는 `show spanning-tree`에서 Type이 **P2p Edge**로 표시된다.

### UplinkFast와 BackboneFast의 차이

**UplinkFast**는 자신의 **직접 연결된 RP(직접 링크)** 장애에 대응해 Block 포트를 **즉시 Forwarding**으로 전환하고, **BackboneFast**는 **간접(원거리) 링크** 장애를 Inferior BPDU로 감지해 **Max-age 20초 대기를 생략**한다(50초→30초).

UplinkFast는 Block 포트를 가진 **Access 계층 스위치에서만** 설정하며 Root-bridge에서는 금지된다. BackboneFast는 **모든 스위치에 공통 설정**이 필수다.

---

## 2. 표준 설정 템플릿 (Configuration)

### PortFast + BPDUguard (단말 포트 표준 세트)

```cisco
! 단말(PC/Server) 연결 포트의 정석 조합
Switch(config)# interface fastethernet 0/1
Switch(config-if)# switchport mode access
Switch(config-if)# spanning-tree bpduguard enable    ! 스위치 오연결 시 err-disable로 차단 (Loop 방지)
Switch(config-if)# spanning-tree portfast            ! 즉시 Forwarding 전환

! === 전체 포트 일괄 적용 (Access 포트에만 자동 적용, 운영망에선 주의) ===
Switch(config)# spanning-tree portfast default
```
### UplinkFast (Access 계층 스위치에서만)

```cisco
! Block 포트를 가진 스위치에서 전역 설정. RP 장애 시 즉시 전환
SW1(config)# spanning-tree uplinkfast
```

### BackboneFast (모든 스위치 공통)

```cisco
! 반드시 모든 스위치에 동일 설정해야 효과 발생
SW1(config)# spanning-tree backbonefast
SW2(config)# spanning-tree backbonefast
SW3(config)# spanning-tree backbonefast
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Switch# show spanning-tree                       ! Type의 'Edge' 표기로 PortFast 확인
Switch# show interfaces status err-disabled      ! BPDUguard로 차단된 포트 확인
Switch# debug spanning-tree events               ! Fast 기능 전환 로그 확인
```

### Convergence 시간 비교 (핵심 암기)

|기능|적용 전|적용 후|
|---|---|---|
|**PortFast**|30초 (Listening+Learning)|즉시 Forwarding|
|**UplinkFast**|30초|즉시 Forwarding|
|**BackboneFast**|50초 (Max-age 20 포함)|30초 (Max-age 생략)|

### 장애 시나리오: PortFast 포트가 err-disabled 됨 (BPDUguard 동작)

- **증상:** 로그에 `%SPANTREE-2-BLOCK_BPDUGUARD ... putting Fa0/x in err-disable state` 출력, 포트 다운.
- **원인 파악:** PortFast+BPDUguard 설정된 포트에 **스위치가 연결되어 BPDU가 수신됨** (정상적인 보호 동작).
- **해결 (수동 복구):**
    
    ```cisco
    interface fastethernet 0/x
     shutdown
     no shutdown      ! err-disable 해제는 반드시 shut → no shut 필요
    ```
    
- **해결 (자동 복구):**
    
    ```cisco
    Switch(config)# errdisable recovery cause bpduguard   ! bpduguard 원인 자동복구 활성화
    Switch(config)# errdisable recovery interval 300       ! 300초 후 자동 복구
    ```
    

> 💡 **개념 보충 (Inferior/Superior BPDU):** Inferior BPDU는 기존 Root보다 **후순위** Bridge-ID를 가진 BPDU. BackboneFast 미설정 시 스위치는 이를 무시하고 Max-age를 기다리지만, 설정 시 이를 **간접 장애 신호로 해석**해 즉시 수렴을 시작한다.


---

]