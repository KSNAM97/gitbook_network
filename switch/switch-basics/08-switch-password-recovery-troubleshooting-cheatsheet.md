# Switch Password 복구 트러블슈팅 치트시트

> **Tag:** #Network #Troubleshooting #Switch #PasswordRecovery #Flash #NVRAM #CheatSheet
> **핵심 요약:** Password 복구 실패는 주로 **Mode 버튼 타이밍 오류**, **`flash_init` 누락**, **`config.text` 파일명 상이**, **저장 누락** 4가지 원인에서 발생한다. `switch:` 프롬프트 진입 여부가 모든 절차의 전제 조건이다.

---

## 1. 핵심 진단 흐름

### Q1. Password 복구 중 막혔을 때 어떤 순서로 확인하는가?

👉 **핵심 답변:** 아래 순서로 확인합니다.

```properties
[1] switch: 프롬프트 진입 성공 여부
        ↓
[2] flash_init 실행 후 dir flash: 확인
        ↓
[3] config.text 파일명 및 크기 확인
        ↓
[4] rename 후 boot (초기 상태 부팅 확인)
        ↓
[5] rename 원복 후 copy flash:config.text running-config
        ↓
[6] hostname 복원 확인
        ↓
[7] 패스워드 변경 후 copy running-config startup-config
        ↓
[8] 재부팅 후 새 패스워드로 진입 테스트
```

---

## 2. 증상별 즉시 대응표

| 증상 | 원인 | 조치 |
|------|------|------|
| `switch:` 프롬프트 미진입 (정상 부팅됨) | Mode 버튼 타이밍 실패 | 전원 분리 후 재시도. 전원 On과 **동시에** Mode 버튼 누름 유지 |
| `dir flash:` 실행 오류 (파일시스템 미마운트) | `flash_init` 미실행 | `flash_init` 먼저 실행 후 `dir flash:` |
| `dir flash:`에 `config.text` 없음 | 설정 미저장 또는 파일명 상이 | `dir flash:`로 실제 파일명 확인 후 해당 파일 rename |
| `rename` 후 `boot`인데 패스워드 여전히 요구 | rename 오타 또는 다른 파일명 | `dir flash:`로 `old.text` 생성 여부 확인 |
| `copy flash:config.text running-config` 후 hostname 미변경 | 파일 손상 또는 크기 0 | `show flash:`로 파일 크기 확인. 0이면 설정이 없었던 것 |
| 새 패스워드 저장했는데 재부팅 후 사라짐 | `copy running-config startup-config` 누락 | 반드시 저장 후 재부팅 테스트 |
| `password-recovery mechanism is disabled` 메시지 | 관리자가 복구 기능 비활성화 | Flash 전체 삭제(공장 초기화) 외 방법 없음 |
| `switch: flash_init` 이후 `dir flash:` 반응 없음 | Flash 하드웨어 오류 가능성 | 장비 교체 검토 또는 하드웨어 점검 |

---

## 3. 필수 진단 명령어

### switch: 프롬프트에서

```cisco
switch: flash_init           ! Flash 파일시스템 마운트
switch: dir flash:           ! 파일 목록 및 config.text 존재·크기 확인
switch: rename flash:config.text flash:old.text
switch: dir flash:           ! 변경 확인 (old.text 존재)
switch: boot
```

### 부팅 후 (IOS에서)

```cisco
Switch# rename flash:old.text flash:config.text
Switch# show flash:                              ! 크기·존재 재확인
Switch# copy flash:config.text running-config
Switch# show running-config | include hostname   ! hostname 복원 확인
Switch# show running-config | include secret     ! 기존 Enable Secret 확인
```

---

## 4. 트러블슈팅 시나리오

### 시나리오 1. `switch:` 프롬프트 진입 실패

#### 증상

Mode 버튼을 눌렀는데 정상 IOS 부팅이 됨.

#### 원인

Mode 버튼 누름 시점이 너무 느림. IOS 로딩이 시작된 후에는 Mode 버튼이 무시됨.

#### 조치

```text
1. 전원 케이블 분리
2. Mode 버튼을 먼저 누른 상태로 유지
3. 그 상태로 전원 케이블 연결
4. 아래 메시지 나타날 때까지 버튼 유지:
   "The system has been interrupted prior to initializing the flash filesystem."
5. switch: 프롬프트 확인
```

---

### 시나리오 2. `dir flash:` 에서 `config.text`가 다른 이름

#### 증상

```text
switch: dir flash:
2  -rwx  2110   config.text        ← 이 장비는 정상
3  -rwx   512   startup.cfg        ← 일부 모델에서 다른 파일명 사용
```

#### 조치

실제 파일명을 확인 후 해당 파일명으로 rename:

```cisco
switch: rename flash:startup.cfg flash:startup.cfg.bak
switch: boot
```

---

### 시나리오 3. `copy flash:config.text running-config` 후 hostname 복원 안 됨

#### 원인 및 확인

```cisco
Switch# show flash:
```

**출력 예시 (비정상):**

```text
2  -rwx  0      config.text        ← 크기 0 = 빈 파일
```

크기가 0이면 설정이 저장된 적 없거나 파일이 손상된 것.

#### 조치

- 기존 설정 없음으로 판단하고 새로 전체 설정 진행
- 백업 설정 파일이 다른 위치에 있다면 TFTP 또는 복사 활용

---

### 시나리오 4. 재부팅 후 새 패스워드로 진입 실패

#### 원인

`copy running-config startup-config` 누락으로 새 패스워드가 Flash에 저장되지 않음.

#### 확인

```cisco
show startup-config | include secret   ! 저장된 패스워드 해시 확인
show running-config | include secret   ! 현재 메모리의 패스워드 확인
```

#### 조치

패스워드를 다시 설정하고 반드시 저장:

```cisco
configure terminal
 enable secret <새_패스워드>
end
copy running-config startup-config
```

---

### 시나리오 5. `password-recovery mechanism is disabled`

#### 증상

```text
The password-recovery mechanism is disabled.
Proceed with system initialization? [yes/no]:
```

#### 원인

관리자가 `no service password-recovery` 명령으로 복구 기능을 비활성화한 것.

#### 해결

- `yes`를 선택하면 Flash 전체 삭제 후 기본 설정으로 초기화됨
- 기존 설정 및 데이터가 **모두 삭제됨**
- 복구 기능 비활성화 정책을 사전에 확인하고 백업을 유지해야 함

---

## 5. 현장 Quick Check

```cisco
! switch: 프롬프트에서
switch: flash_init
switch: dir flash:
switch: rename flash:config.text flash:old.text
switch: boot

! IOS 부팅 후
Switch# rename flash:old.text flash:config.text
Switch# show flash:
Switch# copy flash:config.text running-config
Switch(config)# enable secret <NEW_PASSWORD>
Switch# copy running-config startup-config
```

---

> 📌 **핵심 요약**
> - `switch:` 프롬프트 = 전원 On과 **동시에** Mode 버튼 누름
> - `flash_init` 없이 `dir flash:` 불가
> - `copy flash:config.text running-config` 후 hostname으로 복원 확인
> - `copy running-config startup-config` 누락 = 재부팅 후 패스워드 소멸
> - `password-recovery disabled` = 공장 초기화만 가능
> - 관련: 1-6. 🔑 Switch Password 복구 · 1-7. 🧩 Switch Password 복구 통합 정리 · 1-9. ⚡ Switch Password 복구 퀵 레퍼런스
