# Switch Password 복구 (Password Recovery)

> **Tag:** #Network #Cisco #Switch #PasswordRecovery #NVRAM
> 핵심 요약: Enable Secret 또는 Console Password를 분실한 경우, 물리적 접근(Mode 버튼)으로 Switch를 부팅 모드로 진입한 뒤 NVRAM 설정 파일을 우회하여 새 Password를 설정한다. 반드시 **Console 또는 물리적 접근**이 전제된다.

---

## Q1. Switch Password 복구가 뭔가요?

- **핵심 답변:** Enable Secret / Console Password 분실 시, 장비를 정상 부팅 없이 **Flash 메모리 직접 접근 모드(switch: 프롬프트)** 로 진입하여 NVRAM 설정 파일 적용을 건너뛰고 새 패스워드를 설정하는 절차입니다.
- **전제 조건:** 물리적 Console 케이블 접근 필수. 원격(Telnet/SSH)으로는 불가합니다.
- **원리:** 장비 부팅 시 `config.text`(NVRAM 저장 설정)를 로딩하는데, 파일명을 변경해 로딩을 건너뛰면 **초기화 상태로 부팅**되고, 이후 해당 파일을 수동으로 로딩하여 패스워드만 변경할 수 있습니다.

---

## 동작 원리 & Flash/NVRAM 구조

```text
부팅 순서:
  전원 ON → POST → Bootstrap → IOS 로딩(Flash) → startup-config 로딩(NVRAM/Flash의 config.text)
                                                             ↑
                                              이 파일명을 변경 → 로딩 건너뜀
```

| 파일 | 위치 | 역할 |
|------|------|------|
| `config.text` | Flash 메모리 | startup-config (NVRAM 역할) |
| `private-config.text` | Flash 메모리 | Encrypted 설정 일부 |
| IOS 이미지(.bin) | Flash 메모리 | 운영체제 이미지 |

---

## 표준 설정 — Password 복구 단계별 절차

### Step 1. 진입 준비 (Mode 버튼)

1. 전원 케이블을 뽑습니다.
2. **Mode 버튼을 누른 채로** 전원 케이블을 다시 꽂습니다.
3. `switch:` 프롬프트가 나타날 때까지 Mode 버튼을 유지합니다.

```text
The password-recovery mechanism is enabled.
The system has been interrupted prior to initializing the flash filesystem.
    flash_init
    boot
switch:
```

---

### Step 2. Flash 초기화 및 확인

```cisco
switch: flash_init          ! Flash 파일시스템 초기화 및 마운트
switch: dir flash:          ! Flash 파일 목록 확인
```

**출력 예시:**
```text
Directory of flash:/
 2    -rwx  2110     <date>   config.text          ← NVRAM 설정 파일
 4    -rwx  4086912  <date>   c3550-i5q3l2-mz.121-14.EA1a.bin
 3    -rwx  5        <date>   private-config.text
11907584 bytes available (4091392 bytes used)
```

---

### Step 3. config.text 파일명 변경 (로딩 우회)

```cisco
switch: rename flash:config.text flash:old.text   ! 파일명 변경으로 자동 로딩 방지
switch: dir flash:                                 ! 변경 확인
```

---

### Step 4. 재부팅

```cisco
switch: boot                ! IOS 재부팅 (config.text 없으므로 기본 설정으로 기동)
```

부팅 완료 후:
```text
Would you like to enter the initial configuration dialog? [yes/no]: n
Press RETURN to get started!
Switch>
Switch# enable              ! 패스워드 없이 enable 진입 가능
```

---

### Step 5. 원본 설정 파일 복원 및 패스워드 변경

```cisco
Switch# rename flash:old.text flash:config.text    ! 파일명 원복
Switch# show flash:                                ! config.text 복원 확인
Switch# copy flash:config.text running-config      ! 기존 설정을 DRAM으로 로딩
Destination filename [running-config]?             ! Enter
```

로딩 완료 후 기존 hostname이 복원됩니다:

```cisco
Global_IT# configure terminal
!
Global_IT(config)# enable secret <새_패스워드>     ! Enable Secret 재설정
!
Global_IT(config)# line console 0
Global_IT(config-line)# password <새_콘솔_패스워드>
Global_IT(config-line)# login
Global_IT(config-line)# exit
!
Global_IT(config)# end
!
Global_IT# copy running-config startup-config      ! NVRAM 저장
```

---

## 검증 명령어 & 출력 판독

```cisco
show flash:                         ! Flash 파일 목록 및 config.text 존재 확인
show running-config | include secret ! Enable Secret 해시 확인
show version                        ! 장비 모델, IOS 버전, NVRAM 정보
```

**정상 확인 포인트:**
- `show flash:`에서 `config.text`가 존재하고 크기가 0 이상인지 확인
- `copy flash:config.text running-config` 후 hostname이 원래 장비명으로 복원되는지 확인
- 새 패스워드로 Console/Enable 진입이 정상 동작하는지 테스트

---

## 대표 장애 & 트러블슈팅

| 증상 | 원인 | 조치 |
|------|------|------|
| `switch:` 프롬프트 미진입 | Mode 버튼 타이밍 오류 | 전원 On과 동시에 Mode 버튼 누름 유지 |
| `flash_init` 후 config.text 없음 | 이전에 설정 미저장 또는 파일명이 다름 | `dir flash:`로 실제 파일명 확인 후 rename |
| `copy flash:config.text running-config` 후 hostname 미변경 | 파일 손상 또는 경로 오류 | `show flash:`로 파일 크기 확인, 재시도 |
| 패스워드 변경 후 저장 누락 | `copy run start` 미실행 | 반드시 `copy running-config startup-config` 실행 |

---

## 플랫폼별 주의사항

- **Catalyst 3550/3560/3750:** 위 절차가 적용됩니다. Mode 버튼 위치는 모델마다 다름.
- **Packet Tracer:** Password Recovery 시뮬레이션은 지원하지 않습니다. 실장비 또는 GNS3에서 실습하십시오.
- **Router Password 복구:** 라우터는 `confreg 0x2142`(ROMMON 모드)를 사용하는 별도 절차로, 스위치와 다릅니다.
- `password-recovery mechanism is disabled` 메시지가 나타나면 관리자가 복구 기능을 비활성화한 것으로, 공장 초기화(Flash 전체 삭제) 외에는 복구가 불가능합니다.

---

## 운영망 변경 위험 & Rollback

> ⚠️ Password Recovery는 **물리적 접근이 필요한 작업**입니다. 데이터센터 환경에서는 반드시 변경 승인 후 수행하십시오.

- `copy flash:config.text running-config` 전에 `show flash:`로 **파일 크기가 0이 아닌지** 반드시 확인
- `copy running-config startup-config`를 누락하면 장비 재부팅 시 패스워드가 다시 사라집니다.

---

## 시험 & 면접 포인트

- Password Recovery의 핵심은 **config.text 파일명 변경 → 로딩 우회 → 수동 로딩 → 패스워드 재설정** 4단계
- `flash_init`은 Flash 파일시스템을 마운트하는 명령. 이 없이는 `dir flash:`가 동작하지 않음
- Router는 ROMMON `confreg 0x2142`, Switch는 **Mode 버튼 + flash_init** 방식으로 절차가 다름
- `private-config.text`는 encrypted 설정 일부를 별도 저장하는 파일. 일반적으로 함께 rename하거나 삭제하지 않아도 됨

> 관련: 스위치 동작원리 & VLAN 기초 · Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리
> 통합 정리: 1-7. 🧩 Switch Password 복구 통합 정리 · 1-8. 🚑 Switch Password 복구 트러블슈팅 치트시트 · 1-9. ⚡ Switch Password 복구 퀵 레퍼런스

```text

0) SW1 Fa0/20 = Blocking

   ---- 장애 발생 ----

1) SW1 Fa0/20 = Listening 	[LIS   : 15초]
1) SW1 Fa0/20 = Listening 	[LIS   : 15초]
2) SW1 Fa0/20 = Learning 	[LRN  : 15초]
3) SW1 Fa0/20 = Forwarding 	[FWD : 총 30초 소요]

  # SW1 or SW2

SW1 Fa0/24 = Listening (15초)

SW1 Fa0/24 = RP

SW1 Fa0/20 = AP (blocking)

SW1 Fa0/24 = Learning (15초)

SW1 Fa0/24 = Forwarding

0) SW1 Fa0/20 = Blocking

   ---- 장애 발생 ----

1) SW1 Fa0/20 = Blocking 	[BLK  : Blocking 상태가 20초간 유지된다.]
1) SW1 Fa0/20 = Blocking 	[BLK  : Blocking 상태가 20초간 유지된다.]
2) SW1 Fa0/20 = Listening 	[LIS   : 15초]
3) SW1 Fa0/20 = Learning 	[LRN  : 15초]
4) SW1 Fa0/20 = Forwarding 	[FWD : 총 50초 소요]

0) SW1 Fa0/20 = Blocking

   ---- 장애 발생 ----
```

> `Switch Password 복구.txt`

```cisco
 : Switchport가 동작하지 않는 상태  (Switchport가 "shutdown"된 상태)
 # BPDU 수신	 : X
 # BPDU 송신	 : X
 # Mac-address 학습	 : X
 # 데이터 전송	 : X

 : Switch가 Loop를 방지하기위해서 Switchport를 논리적으로 차단한 상태
 # BPDU 수신	 : O
 # BPDU 송신	 : X
 # Mac-address 학습	 : X
 # 데이터 전송	 : X

   Loop가 감지되지 않으면 통신을 위해서 Learning단계로 전환된다.
 # BPDU 수신	 : O
 # Mac-address 학습	 : X
 # 데이터 전송	 : X

   통신을 위해서 Mac-address를 학습하는 단계이며 15초간 진행된다.
 # BPDU 수신	 : O
 # Mac-address 학습	 : O
 # 데이터 전송	 : X

 # BPDU 송신	 : O
 # BPDU 수신	 : O (해당 Port가 DP인 경우)
 # Mac-address 학습	 : O
 # 데이터 전송	 : O

  # SW1 or SW2
interface fastethernet 0/24
 shutdown
!

SW1# debug spanning-tree events
  # SW1 or SW2
interface fastethernet 0/24
 no shutdown
!
```

> `Switch Password 복구.txt`

```text
Initializing Flash...
flashfs[0]: 3 files, 1 directories
flashfs[0]: 0 orphaned files, 0 orphaned directories
flashfs[0]: Total bytes: 15998976
flashfs[0]: Bytes used: 4091392
flashfs[0]: Bytes available: 11907584
flashfs[0]: Bytes available: 11907584
flashfs[0]: flashfs fsck took 15 seconds.
...done Initializing Flash.
Boot Sector Filesystem (bs:) installed, fsid: 3
switch: 

```

---

> 관련: 1-1. 🔌 스위치 동작원리 & VLAN 기초 · 1-2. 🚚 Switchport & Trunk & Inter-VLAN · 1-3. 🧩 스위치 기본 & VLAN·Trunk 통합 정리 · 1-4. 🚑 스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트 · 1-5. ⚡ 스위치 기본 & VLAN·Trunk 퀵 레퍼런스 · 1-7. 🧩 Switch Password 복구 통합 정리 · 1-8. 🚑 Switch Password 복구 트러블슈팅 치트시트 · 1-9. ⚡ Switch Password 복구 퀵 레퍼런스