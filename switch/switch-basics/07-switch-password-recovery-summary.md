# 🧩 Switch Password 복구 통합 정리 — 절차·원리·플랫폼 한눈에

> **Tag:** #Network #Cisco #Switch #PasswordRecovery #NVRAM #Flash #Summary
> **핵심 요약:** Switch Password 복구는 물리적 접근(Mode 버튼)으로 `switch:` 프롬프트에 진입한 뒤 `config.text` 파일명을 변경해 NVRAM 로딩을 우회하고, 재부팅 후 기존 설정을 수동으로 불러와 패스워드만 재설정하는 4단계 절차다. Router의 ROMMON `confreg 0x2142` 방식과 구분해야 한다.

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Q1. Password 복구의 핵심 원리는?

👉 **핵심 답변:** Cisco Switch는 부팅 시 Flash의 `config.text` 파일을 startup-config로 로딩합니다. 이 파일명을 변경해두면 부팅 중 로딩을 건너뛰어 **패스워드 없는 초기 상태**로 부팅됩니다. 이후 파일명을 원복하고 수동으로 로딩하면 기존 설정 전체를 유지하면서 패스워드만 변경할 수 있습니다.

```text
부팅 순서:
  전원 ON → POST → Bootstrap → IOS 로딩(Flash)
                            → config.text 로딩(NVRAM/Flash)
                                         ↑
                              파일명 변경 → 로딩 건너뜀 → 초기 상태 부팅
```

---

### Q2. Switch vs Router Password 복구 차이는?

| 항목 | **Switch** | **Router** |
|------|------------|------------|
| 진입 방법 | Mode 버튼 + 전원 재연결 | ROMMON (Break 키) |
| 프롬프트 | `switch:` | `rommon>` |
| 핵심 명령 | `flash_init`, `rename`, `boot` | `confreg 0x2142`, `reset` |
| 설정 우회 | `config.text` 파일명 변경 | `0x2142` 레지스터로 startup-config 무시 |

---

### Q3. Flash/NVRAM 구조는?

| 파일 | 위치 | 역할 |
|------|------|------|
| `config.text` | Flash 메모리 | startup-config (NVRAM 역할) |
| `private-config.text` | Flash 메모리 | Encrypted 설정 일부 |
| IOS 이미지 (`.bin`) | Flash 메모리 | 운영체제 이미지 |

> Switch에서 NVRAM은 물리적으로 별도 존재하지 않고 Flash에 `config.text`로 저장됩니다.

---

## 2. 🛠️ 표준 절차 (4단계)

### Step 1. `switch:` 프롬프트 진입

1. 전원 케이블 분리
2. **Mode 버튼을 누른 채** 전원 케이블 재연결
3. `switch:` 프롬프트 표시될 때까지 Mode 버튼 유지

```text
The system has been interrupted prior to initializing the flash filesystem.
    flash_init
    boot
switch:
```

---

### Step 2. Flash 초기화 및 파일 확인

```cisco
switch: flash_init          ! Flash 파일시스템 마운트 (필수)
switch: dir flash:          ! 파일 목록 확인 → config.text 존재 확인
```

---

### Step 3. config.text 파일명 변경 → 재부팅

```cisco
switch: rename flash:config.text flash:old.text
switch: boot
```

부팅 완료 후:

```text
Would you like to enter the initial configuration dialog? [yes/no]: n
Switch> enable               ! 패스워드 없이 진입 가능
```

---

### Step 4. 설정 복원 및 패스워드 변경 → 저장

```cisco
Switch# rename flash:old.text flash:config.text    ! 파일명 원복
Switch# copy flash:config.text running-config      ! 기존 설정 DRAM 로딩
!
! 기존 hostname이 복원됨
Global_IT(config)# enable secret <새_패스워드>
Global_IT(config)# line console 0
Global_IT(config-line)# password <새_콘솔_패스워드>
Global_IT(config-line)# login
Global_IT(config-line)# exit
Global_IT# copy running-config startup-config      ! NVRAM 저장 (필수)
```

---

## 3. 🔍 검증 명령어

```cisco
show flash:                               ! Flash 파일 목록 및 config.text 확인
show running-config | include secret      ! Enable Secret 해시 확인
show version                              ! 장비 모델, IOS 버전
```

**정상 확인 포인트:**
- `show flash:`에서 `config.text`가 존재하고 크기가 0 이상
- `copy flash:config.text running-config` 후 hostname 복원 확인
- 새 패스워드로 Console/Enable 정상 진입 테스트

---

## 4. ⚠️ 대표 함정

| 함정 | 결과 | 조치 |
|------|------|------|
| Mode 버튼 타이밍 실패 | 정상 부팅됨 | 전원 분리 후 재시도. 전원 On과 동시에 버튼 누름 |
| `flash_init` 전에 `dir flash:` | 파일시스템 미마운트 오류 | `flash_init` 먼저 실행 |
| `config.text`가 `dir flash:`에 없음 | 설정 미저장 또는 파일명 상이 | `dir flash:`로 실제 파일명 확인 후 rename |
| `copy flash:config.text running-config` 후 hostname 미변경 | 파일 손상 또는 크기 0 | `show flash:`로 크기 확인 |
| `copy running-config startup-config` 누락 | 재부팅 시 새 패스워드 사라짐 | 반드시 저장 |
| `password-recovery mechanism is disabled` | 관리자가 복구 기능 비활성화 | 공장 초기화(Flash 전체 삭제) 외 방법 없음 |

---

## 5. 🏗️ 플랫폼별 주의사항

- **Catalyst 3550/3560/3750:** 위 절차 적용. Mode 버튼 위치는 모델마다 상이
- **Packet Tracer:** Password Recovery 시뮬레이션 미지원. GNS3 또는 실장비 사용
- **Router:** ROMMON `confreg 0x2142` 방식으로 Switch와 절차 다름
- `private-config.text`: 일반적으로 함께 rename하지 않아도 됨

---

## 6. 📌 핵심 요약

> - 핵심 원리: **`config.text` 파일명 변경 → 로딩 우회 → 수동 복원 → 패스워드 재설정**
> - `flash_init` 없이 `dir flash:` 불가
> - Router는 ROMMON `confreg 0x2142`, Switch는 **Mode 버튼 + flash_init**
> - `copy running-config startup-config` 누락 시 재부팅 후 패스워드 초기화
> - 전제 조건: **반드시 물리적 Console 케이블 접근**
> - 관련: 1-6. 🔑 Switch Password 복구 · 1-8. 🚑 Switch Password 복구 트러블슈팅 치트시트 · 1-9. ⚡ Switch Password 복구 퀵 레퍼런스 · 스위치 동작원리 & VLAN 기초 · Cisco 라우터 기초 - CDP, Mode, Interface, Password 종합 정리
