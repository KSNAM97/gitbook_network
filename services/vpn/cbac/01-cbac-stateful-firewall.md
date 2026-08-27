# CBAC (Context-Based Access Control)

> **Tag:** #network #security #cbac #stateful-firewall #ids #ips #dos #cisco
> **핵심 요약:** 세션 상태를 추적해 "내부→외부 허용, 외부→내부 차단"을 자동 구현하는 IOS 상태 기반 방화벽. L7 검사·DoS 방어·URL 필터까지 지원

---

## 1. 핵심 기술 개념 (Concept)

### 일반 ACL, Reflexive ACL, CBAC의 차이와 방화벽/IDS/IPS의 관계

일반 ACL은 상태를 모르는 정적 필터, Reflexive ACL은 L3/L4까지 나간 세션의 역방향을 미러링, CBAC은 L7(애플리케이션 계층)까지 검사하고 감사·경고·DoS 방어까지 하는 IOS 상태 기반 방화벽이다.

방화벽은 IP/Port 기반 접근 통제를 하고, IDS는 공격을 탐지·경고만 한다(경로 옆에서 복사 분석, 차단 안 함). IPS는 탐지 후 직접 차단한다(경로 중간 인라인 배치, 패킷 폐기·IP 차단·세션 강제종료). CBAC의 동작 원리는 내부→외부 트래픽을 `ip inspect`가 검사해 State Table에 세션을 기록하고, 돌아오는 응답만 동적으로 허용하며, 매칭 안 되는 트래픽은 인바운드 ACL을 적용해 차단하는 방식이다.

### CBAC이 DoS 공격(특히 SYN Flood)을 막는 방식

CBAC은 Half-open 세션(3-way handshake 미완성) 개수를 모니터링해, 과도하게 쌓이면 SYN Flood로 판단하고 오래된 half-open 세션을 정리하거나 해당 호스트를 일정 시간 차단한다. `max-incomplete`(총 개수), `one-minute`(분당 발생률), `tcp max-incomplete host`(호스트별) 임계치로 제어한다.

Half-open이 비정상적으로 많다는 것은 응답을 받지 못하는 연결 시도가 폭주한다는 뜻으로, 전형적인 SYN Flood 신호다. UDP는 handshake가 없어 half-open 개념이 없다.

### CBAC 세션 타이머 (기본값)

| 프로토콜 | 유휴 타임아웃 | 비고 |
|----------|:-------------:|------|
| **TCP idle** | 3600초 | FIN 수신 후 5초 뒤 세션 제거 |
| **UDP idle** | 30초 | 무응답 시 세션 제거 |
| **DNS** | 5초 | DNS Spoofing/DoS 방지 (응답 즉시 세션 제거) |
| **ICMP** | 10초 | 무응답 시 entry 제거 |
| **TCP synwait** | 30초 | SYN 후 미응답 시 삭제 |

### 방화벽 계열 비교

| 구분 | 역할 | 배치 | 차단 여부 |
|------|------|------|:---------:|
| **Firewall** | IP/Port 접근통제 | 인라인 | O |
| **IDS** | 공격 탐지·경고 | 경로 옆(미러링) | X |
| **IPS** | 탐지 + 차단 | 인라인 | O |

---

## 2. 표준 설정 템플릿 (Configuration)

### CBAC 기본 구성 (내부외부 허용, 외부내부 차단)

```cisco
! === [1] 외부→내부 인바운드 ACL : 기본 차단 + 필수 트래픽만 허용 ===
ip access-list extended ACL_IN
 permit ospf host 121.160.10.11 host 224.0.0.5   ! 라우팅(OSPF) 필수 허용
 deny ip any any log-input                        ! 나머지 외부발 차단 + 로그(입력IF 기록)
!
! === [2] 검사(inspect) 규칙 : 내부→외부 나가는 프로토콜 추적 ===
ip inspect name CBAC tcp audit-trail on           ! TCP 추적 + 감사로그
ip inspect name CBAC udp                          ! UDP 추적
ip inspect name CBAC icmp audit-trail on          ! ICMP 추적 + 감사로그
!
! === [3] 인터페이스 적용 ===
interface serial 1/0.10
 ip access-group ACL_IN in       ! 외부→내부는 ACL로 차단
 ip inspect CBAC out             ! 내부→외부는 검사(상태기록)
```

### Half-open 세션 기반 DoS 방어

```cisco
Copy! 전체 Half-open TCP 세션 임계치
ip inspect max-incomplete high 500        ! 500개 초과 시 오래된 세션 정리 시작
ip inspect max-incomplete low 400         ! 400개 이하로 떨어지면 정리 중단
!
! 분당 세션 생성률 임계치
ip inspect one-minute high 1000           ! 분당 1000개 초과 시 방어 동작
ip inspect one-minute low 800
!
! 호스트별 Half-open 임계치 (특정 대상 SYN Flood 방어)
ip inspect tcp max-incomplete host 50 block-time 2   ! 호스트당 50개 초과 시 2분 차단
```

### 세션 타이머 조정

```cisco
Copyip inspect tcp synwait-time 30            ! SYN 후 대기(기본 30초)
ip inspect tcp finwait-time 5             ! FIN 후 대기(기본 5초)
ip inspect tcp idle-time 3600             ! TCP 유휴(기본 3600초)
ip inspect udp idle-time 30               ! UDP 유휴(기본 30초)
ip inspect dns idle-time 5                ! DNS(기본 5초)
```

### URL 필터링 (특정 사이트 허용/차단)

```cisco
Copyip inspect name CBAC http urlfilter
ip urlfilter exclusive-domain permit www.naver.com     ! 허용 도메인
ip urlfilter exclusive-domain deny www.google.co.kr    ! 차단 도메인
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
CopyRouter# show ip inspect session          ! 추적 중 세션 (SIS_OPEN)
Router# show ip inspect config           ! inspect 규칙/타이머 확인
Router# show ip inspect interfaces        ! 인터페이스별 적용 방향 확인
Router# show ip inspect all               ! 전체 CBAC 상태
Router# show ip access-lists              ! ACL 매칭 카운터
```

### `show ip inspect session` 판독

```text
CopySession (192.168.1.1:23234)=>(200.20.2.2:23) tcp SIS_OPEN
```

내부(192.168.1.1)가 외부(200.20.2.2:23)로 연 TCP 세션 → 이 세션의 응답만 자동 허용.

### Case 1: 내부외부는 되는데 응답이 안 돌아옴

- **원인 파악:** `ip inspect`를 **안 걸었거나 방향이 틀림.** 나가는 방향(out)에 inspect가 있어야 상태 기록 + 반환 구멍 생성.
- **해결:** 나가는 방향에 `ip inspect [규칙] out` 확인. `show ip inspect session`에 세션이 잡히는지 점검.

### Case 2: 라우팅(OSPF) 인접이 끊김

- **원인 파악:** 인바운드 ACL이 **OSPF(224.0.0.5)를 차단.** CBAC은 라우팅을 자동 허용하지 않음.
- **해결:** 인바운드 ACL 최상단에 `permit ospf ... host 224.0.0.5` 추가.

### Case 3 (핵심): NAT + IPsec + CBAC 통합 시 VPN이 안 됨

이 셋을 한 라우터에 함께 쓸 때 **순서·예외 처리**가 틀리면 VPN이 깨집니다. 세 가지를 모두 챙겨야 합니다:

1. **NAT 예외 처리** — VPN 대상 트래픽은 NAT되면 안 됨.
    
    ```cisco
    Copyaccess-list 110 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255   ! VPN은 NAT 제외
    access-list 110 permit ip 192.168.1.0 0.0.0.255 any                    ! 나머지만 PAT
    ip nat inside source list 110 pool NAT overload
    ```
    
2. **사설망 경로(Static Route)** — 상대 사설망을 알아야 crypto ACL이 매칭됨.
    
    ```cisco
    Copyip route 192.168.2.0 255.255.255.0 121.160.10.11
    ```
    
3. **CBAC 인바운드에서 IPsec 트래픽 허용** — CBAC이 ESP/ISAKMP를 막으면 VPN 불가.
    
    ```cisco
    Copyip access-list extended ACL_IN
     permit ospf host 121.160.10.11 host 224.0.0.5
     12 permit udp host 100.100.2.2 eq 500 host 100.100.100.1 eq 500   ! ISAKMP(IKE) 허용
     13 permit esp host 100.100.2.2 host 100.100.100.1                  ! ESP 허용
     deny ip any any log-input
    ```
    

- **검증:** `show crypto engine connections active`에서 encrypt/decrypt 카운터가 증가하면 성공.

---

> 🔗 **연계:** 5-3-2. 🔒 IPsec VPN (암호화 · AH&ESP · IKE) — CBAC 환경에서 VPN 구성 시 위 3단계 예외 처리 필수.

```cisco

EX1) 아래의 조건에 맞게 CBAC을 구성하시오
 # 내부에서 외부로 향하는 모든 TCP 트래픽에 대해서는 허용되어져야하며
  외부에서 내부로 향하는 모든것은 트래픽은 차단되어야 한다.

EX2) 아래의 조건에 맞게 CBAC을 구성하시오
 # 내부에서 외부로 향하는 모든 TCP , UDP , ICMP 트래픽에 대해서는 허용되어져야하며
  외부에서 내부로 향하는 모든것은 트래픽은 차단되어야 한다.
 # 단 TCP 트래픽에 대해서는 Log-message가 출력되어야한다.

EX3-1) 아래의 조건에 맞게 CBAC을 구성하시오
 # Rx는 인터넷(http , https)과 DNS만 허용되어야 한다.

   # Rx
ip access-list extended ACL

EX3-2) 아래의 조건에 맞게 CBAC을 구성하시오
 # Rx는 인터넷(http , https)과 DNS만 허용되어야 한다.
 # DNS에 대해서는 Syslog message를 출력해야한다.

   # Rx

EX4) 아래의 조건에 맞게 CBAC을 구성하시오
 # Rx는 내부에서 외부로 통신하는 http 정보중 NAVER로의 통신은 허용해야한다.
 # Rx는 내부에서 외부로 통신하는 http 정보중 Google로의 통신은 차단해야한다.

   # Rx

EX1) 아래의 조건에 맞게 CBAC을 구성하시오
 # R1은 외부에서 내부로 향하는 모든 트래픽의 접근을 차단해야한다.
 # R1은 내부에서 외부로 향하는 TCP트래픽에 대해서 허용해야한다.
 # R1은 내부에서 외부로 향하는 UDP트래픽에 대해서 허용해야한다.
 # R1은 내부에서 외부로 향하는 IP트래픽에 대해서 허용해야한다.

EX2) 아래의 조건에 맞게 CBAC을 구성하시오
 # R3은 외부에서 내부로 향하는 모든 트래픽의 접근을 차단해야한다.
 # R3은 내부에서 외부로 향하는 TCP트래픽에 대해서 허용해야한다. (TCP 트래픽 출력시 Log-message를 출력해야한다.)
```

---
