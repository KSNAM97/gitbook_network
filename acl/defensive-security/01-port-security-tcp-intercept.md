# 3-3-1. 🔒  스위치 & 방어 보안 - Port-security & TCP Intercept

> **Tag:** #Network #Cisco #ACL #Security #AccessControl #PortSecurity #DefensiveSecurity #Switch #VLAN #L2
> **핵심 요약:** L2 경계 방어(Port-security)와 L4 세션 방어(TCP Intercept)
> ACL 기반 정책의 최종 응용 - 실무 인프라 보안의 마지막 관문

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Port-security와 802.1X의 차이 및 선택 기준

Port-security는 **MAC 주소 기반** 접근 제어로 누가 연결됐는지만 보는 반면, 802.1X는 **사용자/장비 인증 기반**으로 진짜 누구인지를 검증한다. Port-security는 회의실/공용 PC 대역 같은 정적 환경에, 802.1X는 노트북/BYOD 등 이동성 있는 환경에 사용한다.

Port-security는 MAC 스푸핑에 취약하므로 진짜 보안은 802.1X + Dynamic ACL 조합이 필요하다. 실무에서는 데이터센터/서버룸 스위치는 Port-security(sticky), 사무실은 802.1X로 분리 설계하며, Port-security는 저비용으로 즉시 적용 가능해 최소한의 방어선으로는 여전히 필수다.

### Violation Mode(protect/restrict/shutdown) 실무 선택 기준

**restrict가 실무 표준**이다. `protect`는 조용히 버리기만 해서 사고 추적이 불가능하고, `shutdown`은 관리자 개입 없이는 복구가 안 되어 운영 부담이 크다. `restrict`는 차단, 로그, 카운터를 남기면서 포트는 살려두기 때문에 감사와 운영의 균형점이 된다.

`shutdown`은 데이터센터, 금융망 등 보안 최우선 환경에 적합하며 err-disable 자동 복구(`errdisable recovery`)와 함께 사용한다. `protect`는 로그가 없어 실무에서는 거의 쓰지 않으며 트러블슈팅 시 원인 파악이 불가능하다. 위반 시 알림이 필요하면 SNMP trap 또는 Syslog 연동이 필수다.

### TCP Intercept의 현재 실무 활용성

대규모 환경에서는 대부분 **DDoS 방어 장비/차세대 방화벽**으로 대체됐지만, IOS 라우터에서 SYN Flood 1차 방어가 필요한 소규모 환경이나 방화벽 앞단의 추가 방어선으로는 여전히 유효하다. 특히 **원격 지사/브랜치 라우터**에서 서버로 향하는 SYN Flood를 조기 차단하는 용도로 쓰인다.

Intercept 모드는 라우터가 프록시 역할을 해 CPU 부담이 크고 대용량 환경에 부적합하며, Watch 모드는 감시만 하므로 성능 부담은 적지만 실시간 차단 효과는 약하다. 현대 인프라에서는 **CDN + WAF + DDoS Scrubbing Service** 조합이 표준으로 자리잡고 있다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Port-security - Sticky 방식 (실무 권장)

```cisco
! [Sticky] 처음 접속한 MAC을 자동 학습 후 running-config에 저장
!          → 관리자가 일일이 MAC 입력할 필요 없음
!
interface range FastEthernet0/1 - 20
 spanning-tree portfast                                 ! Access 포트 고속 활성화
 spanning-tree bpduguard enable                         ! BPDU 유입 시 err-disable (STP 공격 차단)
 !
 switchport mode access                                 ! Trunk 자동 협상 차단
 switchport port-security maximum 1                     ! 포트당 허용 MAC 1개
 switchport port-security mac-address sticky            ! 자동 학습 + 저장
 switchport port-security violation restrict            ! 위반 시 차단 + 로그 (실무 표준)
 switchport port-security                               ! ★ 마지막 활성화 명령
!
end
write memory                                            ! sticky MAC이 저장되도록 반드시 저장
```
### 📌 Port-security - 정적 MAC 지정 방식

```cisco
! [Static] 특정 MAC만 허용 - 서버룸/네트워크 장비 포트에 사용
!
interface FastEthernet0/1
 switchport mode access
 switchport port-security maximum 1
 switchport port-security mac-address 00D0.BA58.EE9A    ! 명시적 MAC 등록
 switchport port-security violation shutdown            ! 위반 시 err-disable
 switchport port-security
!
```
### 📌 Violation Mode 비교표

| 모드           | 위반 프레임 | 포트 상태        | Syslog | SNMP trap | 카운터 | 관리자 복구 |
| ------------ | ------ | ------------ | ------ | --------- | --- | ------ |
| **protect**  | Drop   | up 유지        | ❌      | ❌         | ❌   | 불필요    |
| **restrict** | Drop   | up 유지        | ✅      | ✅         | ✅   | 불필요    |
| **shutdown** | Drop   | err-disabled | ✅      | ✅         | ✅   | **필요** |
### 📌 err-disable 자동 복구 (shutdown 모드 필수 세트)

```cisco
! [errdisable recovery] shutdown으로 잠긴 포트를 일정 시간 후 자동 복구
!
errdisable recovery cause psecure-violation             ! port-security 원인만 복구
errdisable recovery interval 300                        ! 5분 후 자동 복구 시도
!
! ▶ 확인 명령
! show errdisable recovery
```
### 📌 TCP Intercept - Watch 모드 (실무 권장)

```cisco
! [Watch mode] 세션을 감시만 하다가 timeout 시 강제 종료
!              → Intercept 모드보다 CPU 부담 적음
!
! (1) 보호할 서버 대상을 ACL로 지정
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2                     ! Server IP
!
! (2) TCP Intercept 활성화
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode watch                            ! Watch 모드
ip tcp intercept watch-timeout 20                      ! 20초 내 3-way 미완료 시 RST 전송
!
! (3) 임계치 튜닝
ip tcp intercept max-incomplete high 1100              ! 불완전 세션 상한 (기본 1100)
ip tcp intercept max-incomplete low 900                ! 정상 복귀 임계값
ip tcp intercept one-minute high 1100                  ! 1분간 신규 세션 상한
ip tcp intercept drop-mode oldest                      ! 임계 초과 시 오래된 세션부터 삭제
!
```

### 📌 TCP Intercept - Intercept 모드 (강력한 방어)

```cisco
! [Intercept mode] 라우터가 프록시로 개입 - 3-way 완료된 세션만 서버로 전달
!                  → SYN Flood 완전 차단, 다만 CPU 부담 큼
!
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2
!
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode intercept                        ! Intercept 모드 (default)
ip tcp intercept connection-timeout 30                 ! 3-way 완료 후 30초 미사용 시 종료
!
```
### 📌 종합 실무 템플릿 - Access Layer Switch

```cisco
! [운영 스위치 표준 설정] 모든 Access 포트에 일괄 적용
!
! (1) 관리 대역 SSH 접근 제한
ip access-list standard ACL_MGMT
 permit 10.10.10.0 0.0.0.255
 deny   any log
!
line vty 0 4
 access-class ACL_MGMT in
 transport input ssh
 login local
!
! (2) 모든 Access 포트에 Port-security 일괄 적용
interface range FastEthernet0/1 - 24
 switchport mode access
 switchport nonegotiate                                 ! DTP 비활성화 (VLAN Hopping 방지)
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security maximum 2                    ! IP Phone + PC 고려
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security aging time 60                ! 60분 미사용 MAC 만료
 switchport port-security aging type inactivity
 switchport port-security
!
! (3) err-disable 자동 복구
errdisable recovery cause psecure-violation
errdisable recovery cause bpduguard
errdisable recovery interval 300
!
! (4) Uplink 포트는 Trunk로 별도 설정
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 ! ★ Uplink에는 port-security 걸지 않음
!
end
write memory
Copy
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
! Port-security 전체 상태 확인
show port-security
!
! 출력 예시:
! Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
!    Fa0/1          1              1             0               Restrict
!    Fa0/2          1              1             1               Shutdown  ← 위반 발생

! 특정 포트 상세 확인
show port-security interface FastEthernet0/1
!  Port Security              : Enabled
!  Port Status                : Secure-up | Secure-shutdown
!  Violation Mode             : Restrict | Shutdown | Protect
!  Last Source Address:Vlan   : 000D.BDAA.2468:1     ← 위반한 MAC 확인
!  Security Violation Count   : 3

! 학습된 MAC 주소 확인 (sticky/static 모두)
show port-security address

! err-disable 상태 확인
show interfaces status err-disabled
show errdisable recovery

! TCP Intercept 통계
show tcp intercept connections                         ! 현재 감시 중인 세션
show tcp intercept statistics                          ! 누적 통계 (drop, intercept count)

! TCP Intercept 실시간 디버깅 (운영망 주의)
debug ip tcp intercept
```

### 🚨 트러블슈팅 시나리오 1: 포트가 err-disabled로 잠겨서 통신 불가

- **증상:** 사용자 PC 교체 후 네트워크 안 됨. `show interfaces status`에 `err-disabled` 표시.
- **원인 파악:**
    1. `show port-security interface Fa0/x` → Violation Count 확인.
    2. `Last Source Address` 에 새 PC의 MAC이 찍혔는지 확인.
    3. `show mac address-table interface Fa0/x` → 기존 학습된 MAC과 대조.
- **해결법:**
    
    ```cisco
    ! 1단계: 기존 sticky MAC 제거
    interface FastEthernet0/x
     no switchport port-security mac-address sticky
     switchport port-security mac-address sticky            ! 재설정 (기존 MAC 지워짐)
    !
    ! 2단계: 포트 리셋 (err-disable 해제)
    interface FastEthernet0/x
     shutdown
     no shutdown
    !
    ! 3단계: 저장
    write memory
    ```
    
- **재발 방지:** PC 교체 프로세스에 "네트워크팀 통보 및 MAC 재등록" 절차 명문화.

### 🚨 트러블슈팅 시나리오 2: Sticky MAC이 재부팅 후 사라진다

- **증상:** 스위치 재부팅하면 학습된 MAC이 초기화되고 포트가 계속 학습 상태.
- **원인 파악:**
    - Sticky MAC은 running-config에 저장되지만 `write memory` 없으면 startup-config에 반영 안 됨.
- **해결법:**
    
    ```cisco
    ! Sticky 학습 완료 확인 후 반드시 저장
    show run interface FastEthernet0/x                    ! MAC이 config에 있는지 확인
    write memory                                            ! 또는 copy running-config startup-config
    ```
    
- **재발 방지:** Port-security 활성화 후 반드시 `write memory` 를 SOP에 포함.

### 🚨 트러블슈팅 시나리오 3: IP Phone + PC 환경에서 포트가 계속 잠긴다
- **증상:** IP Phone 뒤에 PC 붙였더니 MAC 2개 학습 → 위반 발생.
- **원인 파악:**
    - `switchport port-security maximum 1` 이 기본값 → IP Phone 환경에는 부적합.
- **해결법:**
    
    ```cisco
    interface FastEthernet0/x
     switchport voice vlan 100                              ! Voice VLAN 지정
     switchport port-security maximum 2                     ! Data + Voice 각 1개
     ! Voice VLAN에는 별도 카운트 적용됨
    ```
    
- **재발 방지:** Voice VLAN 사용 환경은 `maximum 2` 이상으로 설계 표준화.

### 🚨 트러블슈팅 시나리오 4: TCP Intercept 활성화 후 정상 사용자 접속 지연

- **증상:** SYN Flood 방어 활성화 후 일반 사용자도 접속 지연 발생.
- **원인 파악:**
    1. `show tcp intercept statistics` → drop count가 비정상적으로 높음.
    2. Intercept 모드는 모든 SYN을 라우터가 프록시 처리 → CPU 부담.
    3. 임계치가 너무 낮게 설정돼 정상 트래픽까지 aggressive drop.
- **해결법:**
    
    ```cisco
    ! 1단계: Watch 모드로 변경 (CPU 부담 감소)
    ip tcp intercept mode watch
    !
    ! 2단계: 임계치 상향
    ip tcp intercept max-incomplete high 5000
    ip tcp intercept max-incomplete low 4000
    !
    ! 3단계: 근본 대책 - 전문 DDoS 방어 장비 도입 검토
    ! 라우터 TCP Intercept는 소규모/임시 방어용
    ```
    
- **재발 방지:** TCP Intercept 활성화 전 서비스 정상 트래픽의 SYN rate를 벤치마킹.
### 🚨 트러블슈팅 시나리오 5: BPDU Guard로 포트가 잠겼는데 원인 불명

- **증상:** 사용자가 스위치를 몰래 붙였다가 뽑았는데 포트가 계속 err-disabled.
- **원인 파악:**
    
    ```cisco
    show interfaces status err-disabled
    ! → reason: bpduguard
    !
    show logging | include BPDU
    ! → Received BPDU on port Fa0/x - 사용자가 스위치를 붙였던 증거
    ```
    
- **해결법:**
    
    ```cisco
    interface FastEthernet0/x
     shutdown
     no shutdown
    ```
    
- **재발 방지:** BPDU Guard는 **Access 포트에 스위치 연결을 원천 차단** 하는 강력한 정책 → 반드시 유지.

### ⚠️ 스위치/방어 보안 실무 체크리스트

- [ ]  모든 Access 포트에 **Port-security + sticky** 를 적용했는가?
- [ ]  Violation Mode는 **restrict** (또는 shutdown + err-disable recovery) 로 설정했는가?
- [ ]  **BPDU Guard + PortFast** 로 사용자 임의 스위치 연결을 차단했는가?
- [ ]  **switchport nonegotiate** 로 DTP를 비활성화해 VLAN Hopping을 방어했는가?
- [ ]  Uplink/Trunk 포트에는 Port-security를 걸지 **않았는지** 확인했는가?
- [ ]  Sticky 학습 후 반드시 **`write memory`** 로 저장했는가?
- [ ]  TCP Intercept는 소규모 방어용임을 이해하고, 대규모 환경에서는 **전용 DDoS 방어 장비**를 사용하는가?
- [ ]  err-disable 자동 복구(`errdisable recovery`)로 운영 부담을 줄였는가?


---

]