# Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리

> **Tag:** #Network #Cisco #Routing #CDP #Interface
> **핵심 요약:** **📌 문서 목적:** Cisco 라우터의 기본 운용에 필요한 CDP, 운영 모드, 인터페이스 설정, 접근 보안(Password)까지 실무 관점에서 압축 정리한 표준 레퍼런스 문서.
>**📌 문서 목적:** Cisco 라우터의 기본 운용에 필요한 CDP, 운영 모드, 인터페이스 설정, 접근 보안(Password)까지 실무 관점에서 압축 정리한 표준 레퍼런스 문서.

---

## 1. 핵심 기술 개념 (Concept)

### CDP (Cisco Discovery Protocol)

> **Q. CDP는 편리한 토폴로지 확인 도구인데, 실무 운영 장비에서는 왜 `no cdp run`으로 비활성화하는 것이 보안 Best Practice로 간주되나요?**

- 👉 **핵심 답변:** CDP는 인접 장비의 **Hostname, IOS 버전, IP, 플랫폼(모델명), 포트 정보**를 평문(Plain Text)으로 광고(Multicast)하기 때문에, 공격자가 내부망에 침투했을 경우 정찰(Reconnaissance) 단계에서 **네트워크 전체 지도를 그리는 결정적 힌트**를 제공하는 정보 유출 취약점이 되기 때문입니다.
- **꼬리 질문 대비 설명:**
  - CDP는 **Layer 2 프로토콜**이므로 IP 주소 설정 없이도 동작하는 것이 장점이자, IP를 몰라도 옆 장비 정보를 캐낼 수 있다는 단점이 됨.
  - 실무 원칙: **내부 코어/디스트리뷰션 구간은 유지**(운영/장애 파악 유리), **외부망 연결 인터페이스(ISP 구간, DMZ) 및 사용자 액세스 포트는 `no cdp enable`로 인터페이스별 차단**하는 것이 표준.

### 운영 모드(User / Privilege / Global) & 저장 구조(RAM / NVRAM)

> **Q. `configure terminal`에서 설정한 내용이 왜 장비를 재부팅(reload)하면 사라질 수 있나요?**

- 👉 **핵심 답변:** Cisco 라우터는 실제 동작 설정을 **휘발성 메모리인 RAM(running-config)** 에서 처리하고, 부팅 시 **비휘발성 메모리인 NVRAM(startup-config)** 의 설정을 읽어오는 이원화 구조이기 때문에, `copy running-config startup-config` (또는 `wr`)로 명시적으로 저장하지 않으면 재부팅 시 유실됩니다.
- **꼬리 질문 대비 설명:**
  - **RAM = 실시간 반영 / NVRAM = 부팅 시 로드** 라는 이원 구조는 "장애 시 즉시 롤백" 이라는 실무 장점으로도 활용됨 (설정 실수 시 저장하지 않고 `reload`).
  - 반대로 원격 작업 중에는 `reload in 5` (5분 뒤 리로드 예약) 명령으로 안전장치를 걸어두는 것이 실무 팁.

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. CDP 운영 표준 (외부 인터페이스 차단)

```cisco
! ============================================
! CDP Global 운영 (기본 유지, 외부 구간만 차단)
! ============================================
enable
configure terminal
!
! CDP Hello 주기 및 Holdtime 조정 (기본값 60/180초, 필요 시 튜닝)
cdp timer 60                      ! CDP 메시지 전송 주기 (초)
cdp holdtime 180                  ! 인접 정보 유지 시간 (미수신 시 삭제까지)
!
! 외부망(ISP 연결) 인터페이스에서는 CDP 비활성화 → 정보 유출 방지
interface Serial1/0
 no cdp enable                    ! 해당 인터페이스에서만 CDP OFF
!
! 만약 전체 장비에서 완전히 끄고 싶다면 (보안 매우 강조 환경)
! no cdp run
!
end
write memory                      ! running-config → startup-config 저장```
```

### 2-2. 접근 보안 (Console / VTY / AUX / Enable)
```cisco
! ============================================
! 기본 호스트명 및 관리자 접근 보안 표준 템플릿
! ============================================
enable
configure terminal
!
hostname R1                       ! 장비 식별용 호스트명 (CDP에도 노출됨)
!
! --- Privilege Mode 진입 비밀번호 (MD5 해시 저장, 필수) ---
enable secret Cisc0!P@ss13Char    ! 13자 이상 대/소문자+숫자+특수문자 권장
!                                 ! (enable password는 평문 저장이라 사용 지양)
!
! --- Console 접근 (물리 접근용) ---
line console 0
 password C0n!Password13char      ! Console 접속 시 요구할 비밀번호
 login                            ! 로그인 시 password 검증 활성화
 exec-timeout 5 0                 ! 5분 무입력 시 자동 로그아웃 (보안)
 logging synchronous              ! 로그 출력으로 입력 끊김 방지 (실무 필수)
!
! --- VTY 접근 (Telnet/SSH 원격) ---
line vty 0 4                      ! 라우터 기본 5개 세션 (0~4)
 password Vty!Password13char
 login
 exec-timeout 5 0
 transport input ssh              ! Telnet 차단, SSH만 허용 (보안 표준)
!
! --- AUX 접근 (Modem, 미사용 시 무조건 차단) ---
line aux 0
 no password
 login
 no exec                          ! AUX 포트 완전 비활성화 권장
!
! --- 모든 평문 패스워드 암호화 (Type 7, 약하지만 최소 방어) ---
service password-encryption
!
end
write memory
```
### 2-3. 인터페이스 표준 (Ethernet / Serial / Loopback)
```cisco
! ============================================
! Ethernet Interface (LAN, DTE 자동 감지)
! ============================================
interface FastEthernet0/0
 description ## LAN_SEGMENT_192.168.1.0/24 ##   ! 실무 필수: 용도 명시
 ip address 192.168.1.254 255.255.255.0         ! Layer 3 주소 (G/W 역할)
 no shutdown                                    ! Layer 1 활성화 (필수)
!
! ============================================
! Serial Interface (WAN 구간, DCE/DTE 구분 중요)
! ============================================
interface Serial1/0
 description ## WAN_TO_R2_HDLC ##
 encapsulation hdlc                             ! L2 프로토콜 (기본값이나 명시 권장)
 bandwidth 64                                   ! 라우팅 메트릭 계산용 (Kbps)
 ip address 192.168.12.1 255.255.255.0
 no shutdown
! ★ DCE 쪽 장비에서는 추가 필수:
! clock rate 64000                              ! DCE만 설정 (단위: bps)
!
! ============================================
! Loopback Interface (관리/OSPF Router-ID 용도)
! ============================================
interface Loopback0
 description ## MGMT_LOOPBACK ##
 ip address 1.1.1.1 255.255.255.255             ! /32 관례 (주소 낭비 방지)
!                                               ! Loopback은 shutdown 되지 않아 안정적
end
write memory
```
## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

|목적|명령어|확인 포인트|
|---|---|---|
|인접 장비 확인|`show cdp neighbors detail`|Hostname, IP, Platform, Local/Remote 포트|
|인터페이스 상태 요약|`show ip interface brief`|Status / Protocol (up/up 여부)|
|라우팅 테이블|`show ip route`|Connected(C) 경로 등록 여부|
|Serial 상세|`show controllers serial 1/0`|**DCE/DTE 판별 및 Clock rate 확인**|
|현재 설정 확인|`show running-config`|RAM 상 실시간 반영 상태|
|저장된 설정 확인|`show startup-config`|NVRAM 저장 상태 (재부팅 시 로드)|
|ARP 매핑|`show arp`|L3-L2 매핑 정상 여부|

---

### 3-2. 대표 장애 시나리오 & 트러블슈팅

#### **[시나리오 1] Serial 인터페이스가 `up / down` 상태로 통신 불가**

`show ip interface brief` 결과 Status는 `up`인데 Protocol이 `down` 인 경우.

**단계별 트러블슈팅:**

1. **1단계 (물리/L1 확인):** Status가 `up`이므로 케이블/포트 물리 계층은 정상. → L2 문제로 좁혀짐.
2. **2단계 (DCE/DTE 판별):** `show controllers serial 1/0` 실행 → 어느 쪽이 **DCE**인지 확인.
	```cisco
DCE V.35, clock rate 64000   ← DCE 쪽
DTE V.35 TX and RX clocks detected  ← DTE 쪽
	```
3. **3단계 (Clock Rate 확인):** DCE 장비에 `clock rate` 명령이 누락되었는지 확인. 누락 시:
    
    ```cisco
    interface Serial1/0
     clock rate 64000
    ```
    
4. **4단계 (Encapsulation 일치 확인):** 양쪽 라우터의 `encapsulation` 프로토콜이 다르면 L2 다운. 양단 모두 `hdlc` 또는 `ppp`로 통일. → **Cisco 기본값은 HDLC**, 이기종 장비 연동 시 반드시 `ppp` 권장.

---

> 관련: 1. 🧮 IPv4 Addressing & Subnetting & VLSM & Summary & Wildcard Mask · 3. 🕐 NTP & SSH & Telnet (원격 접속 & 시간 동기화)