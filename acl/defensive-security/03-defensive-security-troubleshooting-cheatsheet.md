# 🚑 스위치 & 방어 보안 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #PortSecurity #ErrDisable #BPDUGuard #TCPIntercept
> **핵심 요약:** Port-security 장애는 **포트 상태 → Err-disable 원인 → Secure MAC → Maximum/Violation Mode → 실제 연결 장비** 순서로 확인한다. TCP Intercept 장애는 **ACL 매칭 → Mode → Connection/Drop 통계 → Threshold → CPU/Memory → NAT/ZBF 충돌** 순서로 진단한다.

---

## 1. 🎯 공통 진단 순서

### Port-security

```properties
[1] 포트가 connected인가 err-disabled인가?
[2] Err-disable 원인이 psecure인가 bpduguard인가?
[3] 현재 Secure MAC과 Last Source MAC은 무엇인가?
[4] Maximum 값이 정상 단말 수보다 작은가?
[5] Sticky MAC이 Startup-config에 저장됐는가?
[6] 포트에 Phone/AP/Switch/Hypervisor가 연결돼 있는가?
```

### TCP Intercept

```properties
[1] TCP_PROTECT ACL이 실제 Server/Port를 매칭하는가?
[2] Watch인가 Intercept인가?
[3] Incomplete/Drop Count가 증가하는가?
[4] Threshold가 정상 Traffic보다 낮은가?
[5] Router CPU/Memory가 과부하 상태인가?
[6] NAT/ZBF/CBAC와 함께 구성돼 있는가?
```

---

## 2. 🛠️ 증상별 즉시 대응표

| 증상 | 원인 | 확인 | 조치 |
|---|---|---|---|
| PC 교체 후 통신 불가 | 기존 Sticky MAC | Port-security 상세 | 기존 MAC 삭제 후 재학습 |
| 포트 Err-disabled | Shutdown Violation | Err-disable 원인 | 원인 제거 후 Shut/No Shut |
| Phone+PC 연결 시 위반 | Maximum 1 | Current/Maximum | Maximum 2 이상 |
| Sticky가 재부팅 후 사라짐 | Config 미저장 | Running/Startup | Config 저장 |
| Aging이 동작하지 않음 | Sticky Aging 미지원 | 플랫폼 문서 | Dynamic/Static 정책 재설계 |
| BPDU Guard Err-disable | 비인가 Switch/Bridge | Syslog | 연결 장비 제거 |
| Restrict인데 일부만 통신 | 비허용 MAC Drop | Violation Count | MAC/Maximum 수정 |
| TCP 연결 지연 | Intercept CPU 부담 | CPU/통계 | Watch 또는 전용 장비 |
| TCP Intercept 미동작 | ACL 미매칭 | ACL Hit Count | Server/Port 수정 |
| 정상 연결까지 Reset | Timeout 과도하게 짧음 | 통계/Packet Capture | Timeout 조정 |
| 설정 후 NAT 통신 이상 | 기능 조합 제한 | Running-config | TCP Intercept 제거/재설계 |

---

## 3. 🔍 필수 확인 명령어

### Port-security

```cisco
show port-security
show port-security interface FastEthernet0/1
show port-security address
show port-security interface FastEthernet0/1 address
show running-config interface FastEthernet0/1
show mac address-table interface FastEthernet0/1
```

### Err-disable

```cisco
show interfaces status err-disabled
show interfaces FastEthernet0/1 status
show errdisable recovery
show errdisable detect
show logging | include ERR_DISABLE|PSECURE|BPDU
```

### TCP Intercept

```cisco
show tcp intercept connections
show tcp intercept statistics
show ip access-lists TCP_PROTECT
show running-config | include ip tcp intercept
show processes cpu sorted
show processes memory sorted
```

---

## 4. 🚨 대표 장애 시나리오

### 시나리오 1. PC 교체 후 Port-security 위반

#### 증상

- 기존 PC 제거
- 새 PC 연결 후 통신 불가
- `Security Violation Count` 증가 또는 Err-disabled

#### 확인

```cisco
show port-security interface FastEthernet0/1
show port-security address
show running-config interface FastEthernet0/1
```

#### 해결

Running-config에 기록된 정확한 Sticky MAC을 삭제합니다.

```cisco
interface FastEthernet0/1
 no switchport port-security mac-address sticky 000D.BDAA.2468
```

Shutdown Mode로 Err-disable된 경우:

```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
```

새 MAC 학습 후 저장:

```cisco
show port-security address
copy running-config startup-config
```

> 전체 `no switchport port-security mac-address sticky`를 사용하면 해당 인터페이스의 Sticky 주소가 Dynamic Secure 주소로 변환될 수 있습니다. 단일 장비 교체라면 정확한 MAC만 삭제하는 방식을 권장합니다.

---

### 시나리오 2. 포트를 복구해도 다시 Err-disable

#### 원인

Root Cause가 해결되지 않았습니다.

- 비허용 MAC이 계속 연결
- Maximum보다 많은 단말 존재
- Rogue Switch가 BPDU 계속 전송
- 동일 Secure MAC이 다른 Secure Port에서 감지
- IP Phone 뒤에 여러 단말 연결

#### 확인

```cisco
show port-security interface FastEthernet0/1
show mac address-table interface FastEthernet0/1
show logging | include FastEthernet0/1
```

#### 해결 순서

```properties
1. 비인가 장비 또는 케이블 제거
2. 정상 MAC 개수 확인
3. Maximum/Static/Sticky 정책 수정
4. 마지막에 Port 복구
```

```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
```

---

### 시나리오 3. IP Phone + PC 연결 시 위반

#### 원인

Phone과 PC가 서로 다른 Source MAC을 사용하지만 Maximum이 1입니다.

#### 해결

```cisco
interface FastEthernet0/10
 switchport voice vlan 100
 switchport port-security maximum 2
```

지원 플랫폼에서 VLAN별 제한:

```cisco
interface FastEthernet0/10
 switchport port-security maximum 1 vlan access
 switchport port-security maximum 1 vlan voice
```

> USB Dock, Softphone Bridge, Hypervisor 등이 있으면 2개보다 많은 MAC이 정상일 수 있습니다.

---

### 시나리오 4. Sticky MAC이 재부팅 후 사라짐

#### 확인

```cisco
show running-config interface FastEthernet0/1
show startup-config | section interface FastEthernet0/1
```

#### 해결

```cisco
copy running-config startup-config
```

> 자동 저장 정책이 없는 환경에서는 Sticky 학습 후 저장 시점을 운영 절차로 관리해야 합니다.

---

### 시나리오 5. Port-security Aging이 동작하지 않음

#### 설정

```cisco
switchport port-security mac-address sticky
switchport port-security aging time 60
switchport port-security aging type inactivity
```

#### 원인

일부 Catalyst/IOS XE 플랫폼은 Sticky Secure MAC Aging을 지원하지 않습니다.

#### 확인

```cisco
show port-security interface FastEthernet0/1
show version
```

#### 해결 선택

- Sticky 대신 Dynamic Secure MAC 사용
- 장비 교체 시 Sticky MAC 수동 삭제
- Static Aging 지원 여부 확인
- 802.1X/MAB 기반 정책으로 전환

---

### 시나리오 6. BPDU Guard로 포트가 잠김

#### 확인

```cisco
show interfaces status err-disabled
show logging | include BPDU|ERR_DISABLE
```

예상 로그:

```text
Received BPDU on port FastEthernet0/1 with BPDU Guard enabled
bpduguard error detected, putting interface in err-disable state
```

#### 해결

먼저 비인가 Switch, Bridge 또는 Loop 장비를 제거합니다.

```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
```

> 사용자가 Switch를 제거하지 않은 상태에서 복구하면 다시 Err-disable됩니다.

---

### 시나리오 7. Restrict Mode인데 특정 장비만 통신 불가

#### 원인

Restrict는 포트 전체를 내리지 않고 비허용 Source MAC의 프레임만 Drop합니다.

#### 확인

```cisco
show port-security interface FastEthernet0/1
```

확인 항목:

```properties
Port Status
Violation Mode
Maximum MAC Addresses
Total MAC Addresses
Security Violation Count
Last Source Address
```

#### 해결

정상 장비라면 다음 중 하나를 수행합니다.

```cisco
interface FastEthernet0/1
 switchport port-security maximum 2
```

또는 기존 Sticky MAC 삭제 후 재학습:

```cisco
interface FastEthernet0/1
 no switchport port-security mac-address sticky <OLD_MAC>
```

---

### 시나리오 8. TCP Intercept가 동작하지 않음

#### 확인

```cisco
show ip access-lists TCP_PROTECT
show tcp intercept statistics
show running-config | include ip tcp intercept
```

#### 원인 후보

- ACL Destination IP 오류
- TCP Port 조건 오류
- 보호 서버가 NAT 이후 다른 주소로 보임
- ACL에 매칭되지 않음
- 플랫폼/이미지가 기능을 지원하지 않음

#### 정상 예시

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2 eq 443
!
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode watch
```

> TCP Intercept와 NAT를 같이 구성하지 말라는 플랫폼 제한이 있으므로 NAT 주소 기준으로 억지로 맞추기 전에 설계를 재검토합니다.

---

### 시나리오 9. TCP Intercept 적용 후 정상 접속 지연

#### 확인

```cisco
show tcp intercept statistics
show tcp intercept connections
show processes cpu sorted
show processes memory sorted
```

#### 원인 후보

- Intercept Mode의 Proxy 처리 부하
- 정상 SYN Rate보다 낮은 Threshold
- Connection Timeout 과도하게 짧음
- Router 처리 용량 부족

#### 임시 완화

```cisco
ip tcp intercept mode watch
```

Threshold 조정 예시:

```cisco
ip tcp intercept max-incomplete low 4000 high 5000
ip tcp intercept one-minute low 4000 high 5000
```

> 임의로 값을 크게 올리지 말고 정상 시간대와 Peak 시간대의 SYN Rate 및 Half-open 수를 측정한 후 설정합니다.

---

### 시나리오 10. Watch Mode인데 SYN Flood가 즉시 차단되지 않음

#### 원인

Watch Mode는 SYN을 Server로 전달한 뒤 연결 완성을 감시합니다. 지정 시간 내 연결되지 않으면 Server에 RST를 전송해 Half-open State를 제거합니다.

```cisco
ip tcp intercept watch-timeout 20
```

즉시 Proxy 검증이 필요하면:

```cisco
ip tcp intercept mode intercept
```

> Intercept Mode는 CPU 부담이 커질 수 있으므로 변경 전 Router 용량과 정상 Traffic을 확인합니다.

---

### 시나리오 11. TCP Intercept와 ZBF/NAT 적용 후 비정상 통신

#### 확인

```cisco
show running-config | section zone
show running-config | include ip nat
show running-config | include ip tcp intercept
```

#### 원인

Cisco에서는 TCP Intercept를 NAT, ZBF 또는 CBAC와 함께 구성하지 않도록 제한합니다.

#### 조치

- ZBF/NGFW 정책으로 SYN Flood Protection 통합
- TCP Intercept 제거
- NAT 이전/이후 기능 처리 순서를 고려한 별도 장비 배치
- 전용 DDoS 방어 서비스 사용

TCP Intercept 제거:

```cisco
no ip tcp intercept list TCP_PROTECT
```

---

### 시나리오 12. Err-disable 자동 복구가 반복됨

#### 증상

포트가 5분마다 잠겼다가 다시 열립니다.

#### 확인

```cisco
show errdisable recovery
show logging | include ERR_DISABLE
```

#### 원인

```cisco
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```

이 설정은 원인을 수정하지 않고 포트를 일정 시간마다 다시 활성화합니다.

#### 해결

1. 자동 복구를 임시 중단
2. 위반 장비와 MAC 확인
3. 정책 또는 연결 상태 수정
4. 필요 시 자동 복구 재활성화

```cisco
no errdisable recovery cause psecure-violation
```

---

## 5. 🔧 안전한 Port-security 변경 절차

```properties
[1] 현재 Secure MAC 및 Running-config 백업
[2] 연결 장비 종류와 정상 MAC 개수 확인
[3] Violation Mode 확인
[4] 교체 대상 Sticky MAC만 삭제
[5] 새 MAC 학습 확인
[6] Port Status 및 Violation Count 확인
[7] Running-config 저장
```

```cisco
show port-security interface FastEthernet0/1
show port-security address
show running-config interface FastEthernet0/1
copy running-config startup-config
```

---

> 📌 **핵심 요약**
> - Err-disabled이면 먼저 원인을 확인하고 나서 복구
> - PC 교체 시 기존 Sticky MAC만 정확히 삭제
> - Phone+PC는 최소 2개의 Secure MAC 고려
> - Sticky Aging은 플랫폼별 지원 여부 확인
> - BPDU Guard 복구 전 Rogue Switch/Bridge 제거
> - TCP Intercept ACL Hit Count와 통계를 함께 확인
> - TCP Intercept 지연은 Mode·Threshold·CPU 순서로 확인
> - NAT/ZBF/CBAC와 TCP Intercept 혼합 금지
> - 관련: 3-3-1. 🔒  스위치 & 방어 보안 - Port-security & TCP Intercept · 3-3-2. 🧩 스위치 & 방어 보안 통합 정리 · 6-1. 🔐 Port-security & Storm-control · 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard)
