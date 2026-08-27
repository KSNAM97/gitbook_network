# NTP & SSH & Telnet (원격 접속 & 시간 동기화)

> **Tag:** #NTP #SSH #Telnet #RemoteAccess #TimeSync #Cisco
> **핵심 요약:** NTP는 네트워크 장비 간 시간을 동기화(UDP 123)하고, SSH는 암호화된 원격 접속(TCP 22), Telnet은 평문 원격 접속(TCP 23) 프로토콜이다. 실무에서는 Telnet 대신 SSH를 사용하며, Privilege Level로 접근 권한을 세밀하게 제어한다.

---

## 1. 핵심 기술 개념 (Concept)

### Q1. NTP를 사용하는 이유와 Stratum 계층 구조는?
👉 **핵심 답변:** 장비 간 시간이 서로 다르면 Log 분석, 장애 추적, 인증서 검증, 보안 장비 이벤트 분석, AAA 인증 기록 확인 등이 정확하지 않기 때문이다. NTP는 UDP 123번 Port를 사용하며, Stratum 계층으로 정확도 단계를 표현한다.

- **꼬리 질문 대비 설명:**
  - Router나 Switch는 전원이 꺼지거나 배터리/RTC 정보가 정확하지 않으면 시간이 초기값으로 표시될 수 있다.
  - 최초 동기화로부터 3번은 45분 주기, 이후 8시간마다 동기화한다.
  - 시간 동기화 시 위/변조 방지를 위해 MD5 인증 기능을 지원한다.

| Stratum | 설명 |
|---------|------|
| 0 | 세슘 원자시계, GPS, 라디오 시계 (기준 시간 장치) |
| 1 | Stratum 0 장비에 직접 연결된 NTP Server |
| 2 | Stratum 1로부터 시간을 동기화받은 NTP Server |
| 3+ | 상위 Stratum으로부터 동기화받은 장비 (값 1씩 증가) |
| 16 | 동기화되지 않은 상태 |

### Q2. SSH 활성화에 필요한 3가지 필수 조건은?
👉 **핵심 답변:** Hostname 설정 → Domain Name 설정 → RSA Key 생성의 순서가 필수다. RSA Key는 Hostname + Domain Name 기반으로 이름이 생성되므로 앞의 두 설정이 선행되어야 한다.

- **꼬리 질문 대비 설명:**
  - SSH는 TCP 22번 Port를 사용하며, Telnet의 평문 전송 취약점을 보완하기 위해 사용한다.
  - 사용자 계정, Password, 명령어, 출력 결과 등이 암호화되어 전송된다.

| 방식 | 설명 |
|------|------|
| Password 인증 | 사용자 이름 + 패스워드 |
| Public Key 인증 | RSA/ECDSA 공개키 기반 |

### Q3. Telnet의 Privilege Level은 어떤 구조로 접근 권한을 제어하는가?
👉 **핵심 답변:** Cisco는 0~15까지 16단계의 Privilege Level을 제공한다. Level 1은 기본 User EXEC(Router>), Level 15는 최고 권한 Privileged EXEC(Router#)이며, 2~14는 커스텀 레벨로 특정 명령어를 특정 레벨에 할당해 세밀한 접근 제어가 가능하다.

- **꼬리 질문 대비 설명:**
  - Telnet은 TCP 23번 Port를 사용하며, 모든 데이터가 평문(Plain Text)으로 전송된다 → 보안 취약.
  - 실제 운영 환경에서는 SSH 사용을 권장한다.

| Level | 설명 |
|-------|------|
| 0 | 최소 권한 (disable, enable, exit, help, logout) |
| 1 | User EXEC (기본, prompt: Router>) |
| 2~14 | 커스텀 레벨 |
| 15 | 최고 권한 (Privileged EXEC, prompt: Router#) |

### Q4. NTP · SSH · Telnet 세 프로토콜의 핵심 차이는?

👉 **핵심 답변:** 목적, 전송 계층, 보안 수준이 모두 다르다. NTP는 시간 동기화(UDP), SSH는 암호화 원격 접속(TCP), Telnet은 평문 원격 접속(TCP)이다.

| 구분 | NTP | SSH | Telnet |
|------|-----|-----|--------|
| 목적 | 시간 동기화 | 암호화 원격 접속 | 원격 접속 |
| 프로토콜 | UDP | TCP | TCP |
| 포트 | 123 | 22 | 23 |
| 보안 | MD5 인증 지원 | 암호화 통신 | 평문 전송 |
| 실무 | 필수 | 권장 | 레거시/테스트용 |

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. NTP Server / Client 설정

```cisco
! === R1을 NTP Server로 설정 ===
R1# clock set 13:00:00 5 jun 2026

! NTP Server 설정 (R1)
ntp source loopback 211
ntp authenticate
ntp authentication-key 20260605 md5 SOL_NTP
ntp trusted-key 20260605
ntp master 2

! NTP Client 설정 (R2, R3)
ntp authenticate
ntp authentication-key 20260605 md5 SOL_NTP
ntp trusted-key 20260605
ntp server 211.241.228.2 key 20260605

! 한국 표준시 설정 (KST = UTC+9)
clock timezone KST +9
```

### 2-2. SSH 설정

```cisco
! === SSH 설정 (R3 예시) ===
username admin privilege 15 password cisco
!
ip domain-name soldesk.com
!
crypto key generate rsa
! The name for the keys will be: R3.soldesk.com
! RSA Key 크기 입력: 1024
! % Generating 1024 bit RSA keys, keys will be non-exportable...[OK]
!
line vty 0 4
 login local
 transport input ssh
!

! SSH 접속 방법
R1# ssh -v 2 -c aes128-cbc -m hmac-sha1-96 -l admin 3.3.3.3
! -v : SSH Version
! -c : 데이터 송/수신 시 사용할 암호화 알고리즘
! -m : 데이터 송/수신 시 사용할 인증 알고리즘
! -l : login 정보
```

### 2-3. Telnet 설정

```cisco
! === 기본 Telnet 설정 ===
line vty 0 4
 password ciscovty
 login
!

! === 사용자별 권한 레벨 설정 ===
username admin privilege 15 password cisco
!
line vty 0 4
 login local
!

! === Menu 설정 (Telnet 접속 후 메뉴) ===
menu SOL title
  (메뉴 제목 텍스트 입력)
menu SOL text 1 show interface serial
menu SOL command 1 show interfaces serial
!
line vty 0 4
 autocommand menu SOL
!
```

```cisco
! 다른 라우터로 Telnet 접속
R1# telnet 1.1.1.1
! 또는
R1# 1.1.1.1

! 세션 전환
R1# ctrl+shift+6, x   ! 백그라운드로 전환
R1# show sessions      ! 세션 목록 확인
```

### 2-4. Privilege Level 커스텀 권한 설정

```cisco
! EX1) guest 계정 → Global mode 접근 허용
R1(config)# privilege exec level 2 configure
R1(config)# privilege exec level 2 configure terminal

! EX2) guest 계정 → Interface mode 접근 허용
R1(config)# privilege configure level 2 interface

! EX3) guest 계정 → Interface에서 IP 주소 설정 허용
R1(config)# privilege interface level 2 ip
R1(config)# privilege interface level 2 ip address

! EX4) guest 계정 → RIPv2 라우팅 업데이트 허용
R1(config)# privilege configure level 2 router
R1(config)# privilege router level 2 version
R1(config)# privilege router level 2 network

! Menu 기반 접속 제한 (autocommand)
line vty 0 4
 login local
 autocommand menu SOL
!
menu SOL title
  (메뉴 제목 텍스트 입력)
menu SOL text 1 show interfaces
menu SOL command 1 show interfaces
menu SOL text 2 exit
menu SOL command 2 exit
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```cisco
! NTP
clock set HH:MM:SS DD MON YYYY          ! 시간 수동 설정
ntp master <stratum>                     ! NTP Server로 동작
ntp server <ip> [key <keyid>]            ! NTP Client 설정
ntp authenticate                         ! NTP 인증 활성화
ntp authentication-key <id> md5 <key>   ! 인증 키 설정
ntp trusted-key <id>                     ! 신뢰할 키 지정
clock timezone KST +9                    ! 타임존 설정
show clock                               ! 현재 시간 확인
show ntp status                          ! NTP 동기화 상태
show ntp associations                    ! NTP 피어 정보

! SSH
ip domain-name <domain>                  ! 도메인 설정
crypto key generate rsa                  ! RSA 키 생성
ip ssh version 2                         ! SSH v2 강제
show ip ssh                              ! SSH 상태 확인
show ssh                                 ! 현재 SSH 세션

! Telnet
line vty 0 4                             ! VTY 라인 설정
 password <pw>                           ! Telnet 패스워드
 login                                   ! 패스워드 인증
 login local                             ! 로컬 DB 인증
 transport input ssh                     ! SSH만 허용 (Telnet 차단)
show users                               ! 현재 접속 세션
show privilege                           ! 현재 Privilege Level 확인
```

### 3-2. 트러블슈팅 시나리오

| 증상 | 원인 | 해결 |
|------|------|------|
| show clock에 `*` 표시 | NTP 미동기화 상태 | `show ntp status`로 동기화 확인 |
| SSH 접속 불가 | RSA Key 미생성 | `crypto key generate rsa` 실행 |
| SSH 접속 불가 | Hostname/Domain 미설정 | `hostname`, `ip domain-name` 설정 |
| Telnet 접속 거부 | VTY password 미설정 | `line vty 0 4` → `password`, `login` |
| NTP 인증 실패 | Key 불일치 | `ntp authentication-key` 값 일치 확인 |

### 3-3. 실습 예시 & 콘솔 출력 해석

```text
R1# clock set 13:00:00 5 jun 2026
*Jun  5 13:00:00.000: %SYS-6-CLOCKUPDATE: System clock has been updated
  from 00:05:31 UTC Fri Mar 1 2002 to 13:00:00 UTC Fri Jun 5 2026,
  configured from console by console.

R1# show clock
13:00:12.199 UTC Fri Jun 5 2026

! NTP 동기화 후 KST 시간
R2# show clock
22:05:00.628 KST Fri Jun 5 2026
```

> NTP Server이다 → `ntp master <stratum>` 설정으로 해당 장비가 NTP Server로 동작
> NTP Client이다 → `ntp server <ip>` 설정으로 지정 Server로부터 시간 동기화

```cisco
R3(config)# ip domain-name soldesk.com
R3(config)# crypto key generate rsa
The name for the keys will be: R3.soldesk.com
Choose the size of the key modulus in the range of 360 to 2048 for your
  General Purpose Keys.
How many bits in the modulus [512]: 1024
% Generating 1024 bit RSA keys, keys will be non-exportable...[OK]

! RSA Key를 사용하면 SSH 활성화됨 → Hostname과 Domain 정보 기반으로 Key 이름 생성
```

```cisco
! Telnet 접속 시 인증
R1# telnet 1.1.1.1
Trying 1.1.1.1 ... Open

User Access Verification
Username: admin
Password:

! Privilege Level 확인
R3# show privilege
Current privilege level is 15

! 세션 백그라운드 전환
R1# Ctrl+Shift+6, x    ! 현재 세션 일시 중단

R1# show sessions
Conn Host          Address         Byte  Idle Conn Name
   1 1.1.1.1       1.1.1.1            0     0 1.1.1.1
*  2 2.2.2.2       2.2.2.2           10     0 2.2.2.2

! Invalid Input Detected 오류
! → 명령어 오타 또는 잘못된 모드에서 입력 시 발생
! → show run으로 현재 설정 확인
```

```cisco
! Level 2 미만 → 접근 불가 예시
R4# show startup-config    ! ❌ 권한 부족 (NVRAM 설정 확인 불가)
R4# conf t                 ! ❌ 권한 부족 (Global mode 전환 불가)

! Level 2 이상 → 접근 가능 예시
R1# conf t                 ! ✅ Global mode 진입
R1(config)# interface loopback 100                           ! ✅ Interface 생성 가능
R1(config-if)# ip address 100.100.100.100 255.255.255.0     ! ✅ IP 설정 가능
```

---

> 📌 **핵심 요약**
> - NTP: UDP 123, Stratum 계층으로 정확도 표현, `ntp master` (Server) / `ntp server` (Client), MD5 인증 지원
> - SSH: TCP 22, **Hostname + Domain Name + RSA Key** 3가지 필수 조건, 암호화 전송
> - Telnet: TCP 23, 평문 전송 → 실무 SSH 권장, Privilege Level 2~14로 세밀한 권한 제어 가능
> - 관련: 1. 🧮 IPv4 Addressing & Subnetting & VLSM & Summary & Wildcard Mask · 2. ⚙️ Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리
