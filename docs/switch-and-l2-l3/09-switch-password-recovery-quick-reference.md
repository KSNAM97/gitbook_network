# Switch Password 복구 퀵 레퍼런스

> **Tag:** #Network #QuickReference #Switch #PasswordRecovery #Flash #NVRAM #CheatSheet
> **핵심 요약:** Switch Password 복구 명령어를 빠르게 조회하기 위한 암기 카드. 절차는 **진입 → flash_init → rename → boot → 원복 → 설정 로딩 → 패스워드 변경 → 저장** 8단계다.

---

## 1. 전체 절차 요약 (8단계)

```ini
[1] 전원 분리
[2] Mode 버튼 누른 채 전원 연결 → switch: 프롬프트
[3] flash_init
[4] dir flash:  (config.text 확인)
[5] rename flash:config.text flash:old.text
[6] boot
[7] (부팅 후) rename flash:old.text flash:config.text
[8] copy flash:config.text running-config
    → enable secret <새_패스워드> 설정
    → copy running-config startup-config
```

---

## 2. switch: 프롬프트 명령어

```cisco
switch: flash_init                                      ! Flash 마운트 (필수)
switch: dir flash:                                      ! 파일 목록 확인
switch: rename flash:config.text flash:old.text         ! NVRAM 로딩 우회
switch: dir flash:                                      ! rename 확인
switch: boot                                            ! 재부팅
```

---

## 3. IOS 부팅 후 명령어

```cisco
Switch# rename flash:old.text flash:config.text         ! 파일명 원복
Switch# show flash:                                      ! 크기 0 이상 확인
Switch# copy flash:config.text running-config           ! 기존 설정 로딩
!                                                        ! hostname 복원 확인
Switch# configure terminal
Switch(config)# enable secret <새_패스워드>
Switch(config)# line console 0
Switch(config-line)# password <새_콘솔_패스워드>
Switch(config-line)# login
Switch(config-line)# exit
Switch(config)# end
Switch# copy running-config startup-config              ! 저장 (필수)
```

---

## 4. 검증 명령어

```cisco
show flash:                                ! Flash 파일 목록 및 크기
show running-config | include hostname     ! hostname 복원 확인
show running-config | include secret       ! Enable Secret 확인
show version                               ! IOS 버전, 장비 모델
```

---

## 5. Switch vs Router 방식 비교

| 항목 | **Switch** | **Router** |
|------|------------|------------|
| 진입 방법 | Mode 버튼 + 전원 재연결 | ROMMON (Break 키) |
| 프롬프트 | `switch:` | `rommon>` |
| 핵심 명령 | `flash_init` + `rename` + `boot` | `confreg 0x2142` + `reset` |
| 설정 우회 | `config.text` 파일명 변경 | 레지스터로 startup-config 무시 |
| 설정 복원 | `copy flash:config.text running-config` | `copy startup-config running-config` |

---

## 6. Flash 파일 구조

| 파일 | 역할 |
|------|------|
| `config.text` | startup-config (NVRAM 역할) |
| `private-config.text` | Encrypted 설정 일부 |
| `*.bin` | IOS 운영체제 이미지 |

---

## 7. 증상별 Quick Check

| 증상 | 먼저 확인 |
|------|-----------|
| `switch:` 미진입 | 전원 On과 동시에 Mode 버튼 누름 재시도 |
| `dir flash:` 오류 | `flash_init` 실행 여부 확인 |
| `config.text` 없음 | `dir flash:`로 실제 파일명 확인 후 해당 파일 rename |
| hostname 미복원 | `show flash:`로 크기 0 여부 확인 |
| 재부팅 후 패스워드 소멸 | `copy running-config startup-config` 누락 여부 |
| `password-recovery disabled` | 공장 초기화(Flash 삭제) 외 방법 없음 |

---

## 8. 핵심 주의사항

```properties
flash_init 없이 dir flash: 불가
rename은 실제 파일명과 정확히 일치해야 함 (오타 주의)
copy flash:config.text running-config = 기존 설정 전체 복원
copy running-config startup-config = 필수 (누락 시 재부팅 후 패스워드 소멸)
password-recovery disabled = 공장 초기화만 가능
Packet Tracer = Password Recovery 미지원
```

---

> 📌 **핵심 요약**
> - 진입: 전원 On + Mode 버튼 동시 → `switch:`
> - 핵심: `flash_init` → `rename` → `boot` → 원복 → `copy` → 패스워드 변경 → **저장**
> - 저장 누락 금지: `copy running-config startup-config`
> - 관련: 1-6. 🔑 Switch Password 복구 · 1-7. 🧩 Switch Password 복구 통합 정리 · 1-8. 🚑 Switch Password 복구 트러블슈팅 치트시트
