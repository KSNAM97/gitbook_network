# ⚡ 스위치 & 방어 보안 명령어 퀵 레퍼런스

> **Tag:** #Network #QuickReference #PortSecurity #BPDUGuard #ErrDisable #TCPIntercept
> **핵심 요약:** Port-security·BPDU Guard·Err-disable·TCP Intercept 설정과 검증 명령어를 빠르게 조회하기 위한 암기 카드. Port 역할과 플랫폼 지원 여부를 확인한 후 사용한다.

---

## 1. 🔐 Port-security 기본 설정

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

---

## 2. 📍 Static Secure MAC

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport port-security maximum 1
 switchport port-security mac-address 00D0.BA58.EE9A
 switchport port-security violation shutdown
 switchport port-security
```

---

## 3. 📌 Sticky Secure MAC

Sticky 활성화:

```cisco
switchport port-security mac-address sticky
```

특정 Sticky MAC 지정:

```cisco
switchport port-security mac-address sticky 00D0.BA58.EE9A
```

특정 Sticky MAC 삭제:

```cisco
no switchport port-security mac-address sticky 00D0.BA58.EE9A
```

Sticky 비활성화:

```cisco
no switchport port-security mac-address sticky
```

Config 저장:

```cisco
copy running-config startup-config
```

---

## 4. 🚨 Violation Mode

```cisco
switchport port-security violation protect
switchport port-security violation restrict
switchport port-security violation shutdown
switchport port-security violation shutdown vlan
```

| Mode | 위반 프레임 | Port 상태 | 알림/Count |
|---|---|---|---|
| Protect | Drop | Up | 없음 또는 제한적 |
| Restrict | Drop | Up | 있음 |
| Shutdown | Drop | Err-disabled | 있음 |
| Shutdown VLAN | Drop | VLAN 제한 | 플랫폼 의존 |

기본값:

```properties
Maximum Secure MAC = 1
Violation Mode = Shutdown
Sticky Learning = Disabled
Port-security = Disabled
Aging = Disabled
```

---

## 5. ☎️ IP Phone + PC

```cisco
interface FastEthernet0/10
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport port-security
```

VLAN별 Maximum:

```cisco
switchport port-security maximum 1 vlan access
switchport port-security maximum 1 vlan voice
```

---

## 6. ⏱️ Secure MAC Aging

```cisco
switchport port-security aging time 60
switchport port-security aging type inactivity
```

Absolute:

```cisco
switchport port-security aging type absolute
```

Static MAC Aging 허용:

```cisco
switchport port-security aging static
```

> Sticky Secure MAC Aging 지원 여부는 플랫폼별로 확인합니다.

---

## 7. 🌳 PortFast + BPDU Guard

Interface별 설정:

```cisco
interface FastEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
```

전역 Edge Port 설정:

```cisco
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

> 전역 설정은 PortFast가 활성화된 Port에 BPDU Guard를 적용합니다. Uplink가 PortFast로 잘못 설정되지 않았는지 먼저 확인합니다.

---

## 8. ⛔ DTP 비활성화

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport nonegotiate
```

Static Trunk:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 10,20,30,100
```

---

## 9. ♻️ Err-disable Recovery

Port-security 자동 복구:

```cisco
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```

BPDU Guard 자동 복구:

```cisco
errdisable recovery cause bpduguard
```

자동 복구 해제:

```cisco
no errdisable recovery cause psecure-violation
no errdisable recovery cause bpduguard
```

수동 복구:

```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
```

> Root Cause를 제거한 후 복구합니다.

---

## 10. 🛡️ TCP Intercept ACL

전체 TCP 서비스:

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2
```

HTTP/HTTPS만:

```cisco
ip access-list extended TCP_PROTECT
 permit tcp any host 211.241.228.2 eq 80
 permit tcp any host 211.241.228.2 eq 443
```

활성화:

```cisco
ip tcp intercept list TCP_PROTECT
```

```properties
ACL permit = TCP Intercept 처리 대상
ACL deny/미매칭 = Intercept하지 않고 정상 전달
```

---

## 11. 👀 TCP Intercept Watch Mode

```cisco
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode watch
ip tcp intercept watch-timeout 20
```

```properties
SYN은 Server로 전달
Router가 연결 성립 여부 감시
Timeout 시 Server에 RST 전송
```

---

## 12. 🧱 TCP Intercept Intercept Mode

```cisco
ip tcp intercept list TCP_PROTECT
ip tcp intercept mode intercept
ip tcp intercept connection-timeout 30
```

```properties
Router가 Client와 먼저 Handshake
검증 성공 후 Server와 Handshake
두 연결을 결합
기본 Mode = Intercept
```

> `connection-timeout`은 관리 중인 연결의 Idle Timeout입니다.

---

## 13. 📈 TCP Intercept Threshold

최신 문법:

```cisco
ip tcp intercept max-incomplete low 900 high 1100
ip tcp intercept one-minute low 900 high 1100
```

구형 문법:

```cisco
ip tcp intercept max-incomplete low 900
ip tcp intercept max-incomplete high 1100
ip tcp intercept one-minute low 900
ip tcp intercept one-minute high 1100
```

Drop Mode:

```cisco
ip tcp intercept drop-mode oldest
ip tcp intercept drop-mode random
```

| Threshold | 의미 |
|---|---|
| `max-incomplete` | 현재 Half-open Connection 수 |
| `one-minute` | 최근 1분 Connection Request 수 |
| `high` | Aggressive Mode 진입 |
| `low` | Normal Mode 복귀 |

---

## 14. ❌ TCP Intercept 제거

```cisco
no ip tcp intercept list TCP_PROTECT
no ip tcp intercept mode watch
no ip tcp intercept watch-timeout 20
```

> Mode 명령 제거 시 플랫폼 기본값인 Intercept Mode로 돌아갈 수 있으므로 전체 기능 제거는 `no ip tcp intercept list`를 우선 사용합니다.

---

## 15. 🔍 Port-security 검증

```cisco
show port-security
show port-security interface FastEthernet0/1
show port-security address
show port-security interface FastEthernet0/1 address
show running-config interface FastEthernet0/1
show mac address-table interface FastEthernet0/1
```

확인 항목:

```properties
Port Security
Port Status
Violation Mode
Maximum MAC Addresses
Total MAC Addresses
Configured MAC Addresses
Sticky MAC Addresses
Last Source Address
Security Violation Count
```

---

## 16. 🔍 Err-disable/BPDU Guard 검증

```cisco
show interfaces status err-disabled
show interfaces FastEthernet0/1 status
show errdisable recovery
show errdisable detect
show logging | include ERR_DISABLE
show logging | include PSECURE
show logging | include BPDU
show spanning-tree interface FastEthernet0/1 detail
```

---

## 17. 🔍 TCP Intercept 검증

```cisco
show tcp intercept connections
show tcp intercept statistics
show ip access-lists TCP_PROTECT
show running-config | include ip tcp intercept
show processes cpu sorted
show processes memory sorted
```

Debug:

```cisco
debug ip tcp intercept
undebug all
```

> 운영망에서는 CPU와 출력량에 주의합니다.

---

## 18. 🧭 포트 유형별 권장 프로파일

| 포트 유형 | Port-security | Maximum | BPDU Guard | 비고 |
|---|---|---:|---|---|
| 일반 PC | Sticky 가능 | 1 | 적용 | 고정 단말 |
| Phone+PC | Sticky 가능 | 2 이상 | 적용 | Voice VLAN |
| 고정 서버 | Static/Sticky | 실제 MAC 수 | 적용 가능 | Teaming 확인 |
| AP | 별도 설계 | 다수 가능 | 환경별 | 여러 Client MAC |
| Hypervisor | 일반 설정 부적합 | 다수 | 환경별 | VM MAC 통과 |
| 802.1X Port | 호환성 확인 | NAC 정책 | 적용 | AAA/MAB 고려 |
| Uplink/Trunk | 일반적으로 미적용 | 해당 없음 | 미적용 | Trunk 정책 사용 |
| SPAN Destination | 미지원 가능 | 해당 없음 | 환경별 | 플랫폼 확인 |
| EtherChannel | 미지원 가능 | 해당 없음 | 환경별 | 플랫폼 확인 |

---

## 19. ⚠️ 핵심 함정표

| 함정 | 결과 | 조치 |
|---|---|---|
| PC 교체 전 Sticky 미삭제 | 새 MAC 차단 | 기존 MAC 삭제 |
| Phone+PC에 Maximum 1 | Violation | Maximum 2 이상 |
| Sticky 후 Config 미저장 | 재부팅 후 소실 | Startup-config 저장 |
| Sticky+Aging 무조건 기대 | Aging 미동작 | 플랫폼 확인 |
| BPDU Guard 원인 제거 없이 복구 | 재차 Err-disable | Rogue 장비 제거 |
| 모든 Access Port에 동일 정책 | AP/Phone/VM 장애 | 역할별 프로파일 |
| TCP ACL Permit을 통과 의미로 오해 | 정책 오판 | 처리 대상 의미로 이해 |
| Threshold를 임의로 낮춤 | 정상 연결 Drop | Traffic Baseline 측정 |
| TCP Intercept+NAT/ZBF | 비정상 동작 가능 | 설계 분리 |
| Intercept를 대규모 DDoS 방어로 사용 | Router 과부하 | 전용 방어 도입 |

---

## 20. 🚑 즉시 복구 순서

Port-security:

```properties
show interfaces status err-disabled
        ↓
show port-security interface <IF>
        ↓
show logging
        ↓
비인가 장비/기존 Sticky MAC 제거
        ↓
shutdown / no shutdown
        ↓
새 MAC 학습 및 저장
```

TCP Intercept:

```properties
show ip access-lists TCP_PROTECT
        ↓
show tcp intercept statistics
        ↓
show processes cpu sorted
        ↓
Mode/Threshold 확인
        ↓
Watch 전환 또는 기능 제거
        ↓
ZBF/NGFW/DDoS 설계 검토
```

---

> 📌 **핵심 요약**
> - Sticky: `switchport port-security mac-address sticky`
> - Violation: `protect | restrict | shutdown`
> - Err-disable 복구: 원인 제거 후 `shutdown` → `no shutdown`
> - BPDU Guard: End Host Port에만 적용
> - TCP Intercept Watch: 통과 후 감시
> - TCP Intercept Intercept: Router Proxy
> - TCP Intercept ACL Permit: 검사 대상 선택
> - 관련: 3-3-1. 🔒  스위치 & 방어 보안 - Port-security & TCP Intercept · 3-3-2. 🧩 스위치 & 방어 보안 통합 정리 · 3-3-3. 🚑 스위치 & 방어 보안 트러블슈팅 치트시트 · 6-1. 🔐 Port-security & Storm-control · 3-3. 🛡️ STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard)
