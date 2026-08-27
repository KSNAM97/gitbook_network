# 🔀 NAT (Network Address Translation)

> **Tag:** #network #nat #pat #overload #private-ip #cisco
> **핵심 요약:** 사설 IP를 공인 IP로 변환해 외부 통신을 가능케 하는 기술. Static/Dynamic/PAT 구분과 inside/outside가 핵심

---

## 1. 🎯 핵심 기술 개념 (Concept)

### NAT가 필요한 이유와 사설 IP의 한계

사설 IP(RFC1918)는 인터넷에서 라우팅되지 않도록 예약된 대역이라, ISP 라우터는 사설 대역 트래픽을 즉시 폐기한다. 그래서 내부는 사설 IP로 구성하되, 외부로 나갈 때 NAT가 공인 IP로 변환해줘야 인터넷 통신이 된다. 부가적으로 IPv4 주소 고갈 완화와 내부 주소 은닉(보안) 효과도 있다.

사설 대역은 A=`10.0.0.0/8`, B=`172.16.0.0~172.31.255.255`, C=`192.168.0.0/16`이며, 라우팅 프로토콜(RIP/EIGRP/OSPF)로도 인터넷에 광고되지 않는다. NAT 방향 기준은 `ip nat inside`(내부망 쪽 인터페이스), `ip nat outside`(외부/ISP 쪽 인터페이스)이며, 이 지정이 없으면 NAT는 동작하지 않는다.

### Static NAT, Dynamic NAT, PAT(Overload)의 결정적 차이

Static은 사설↔공인 1:1 고정 매핑으로 외부→내부 역방향 접속이 가능(서버 공개용)하다. Dynamic은 다수 사설을 공인 풀에서 임시 배정하며, PAT(Overload)는 포트 번호를 이용해 다수 사설을 공인 IP 1개로 몰아 쓰는 방식으로 실무에서 가장 널리 쓰인다.

역방향(외부→내부) 접속은 Static NAT만 가능하다. PAT는 외부에서 온 트래픽에 매핑 엔트리가 없어 라우터까지만 도달하고 내부로 못 들어간다(포트포워딩 설정 시 예외). PAT는 세션마다 출발지 포트 번호로 구분하므로 공인 IP 1개로 수천 세션을 동시에 처리할 수 있다.

### 📊 NAT 유형 비교

| 유형 | 매핑 | 외부→내부 | 주요 용도 |
|------|------|:---------:|-----------|
| **Static NAT** | 사설↔공인 1:1 고정 | ✅ 가능 | 내부 서버 외부 공개 |
| **Dynamic NAT** | 다수 사설 ↔ 공인 풀 | ❌ | 공인 IP 여유 있을 때 |
| **PAT(Overload)** | 다수 사설 → 공인 1개 (포트 구분) | ❌ | 일반 인터넷 공유(대부분) |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 📌 Static NAT (1:1, 서버 공개용)

```cisco
! === 방향 지정 (필수) ===
GIT(config)# interface fastethernet 0/1
GIT(config-if)# ip nat inside                              ! 내부망 쪽
GIT(config)# interface serial 1/0
GIT(config-if)# ip nat outside                             ! 외부(ISP) 쪽
!
! === 1:1 고정 매핑 ===
GIT(config)# ip nat inside source static 192.168.10.1 121.160.23.2
!                                        (사설IP)        (공인IP)
```

### 📌 Dynamic NAT / PAT (Overload) - 4단계 정석

```cisco
! === 1단계: 방향 지정 ===
GIT(config)# interface fastethernet 0/1
GIT(config-if)# ip nat inside
GIT(config)# interface serial 1/0
GIT(config-if)# ip nat outside
!
! === 2단계: ACL로 변환 대상(사설 대역) 정의 ===
GIT(config)# access-list 1 permit 192.168.10.0 0.0.0.255
!
! === 3단계: 변환 후 사용할 공인 IP 풀 정의 ===
GIT(config)# ip nat pool S_NET 121.160.23.2 121.160.23.2 netmask 255.255.255.0
!
! === 4단계: ACL + 풀 연결하여 PAT 활성화 ===
GIT(config)# ip nat inside source list 1 pool S_NET overload    ! overload = PAT
```

> 💡 **overload 키워드가 핵심.** 붙이면 포트 기반 PAT(공인 1개 공유), 빼면 순수 Dynamic NAT(1:1 임시 배정)가 됩니다.

### 📌 인터페이스 직접 PAT (풀 없이, 소규모 표준)

```cisco
! 공인 IP가 아웃바운드 인터페이스에 이미 있으면 풀 없이 바로 PAT 가능
GIT(config)# ip nat inside source list 1 interface serial 1/0 overload
```

### 📌 사설 대역 외부 유출 차단 ACL (ISP 방향 필터)

```cisco
! ISP로 사설 대역이 새어나가지 않도록 아웃바운드 차단
ISP(config)# ip access-list standard NAT
ISP(config-std-nacl)# deny 10.0.0.0 0.255.255.255
ISP(config-std-nacl)# deny 172.16.0.0 0.15.255.255
ISP(config-std-nacl)# deny 192.168.0.0 0.0.255.255
ISP(config-std-nacl)# permit any
!
ISP(config)# interface serial 1/1
ISP(config-if)# ip access-group NAT out
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### ✅ 필수 검증 명령어

```cisco
GIT# show ip nat translations       ! 현재 NAT 변환 매핑 테이블 (가장 중요)
GIT# show ip nat statistics         ! NAT 히트 수/풀 사용량
GIT# debug ip nat                   ! 실시간 변환 과정 추적 (s=..-> 형태)
GIT# clear ip nat translation *     ! NAT 테이블 초기화 (설정 변경 전 필수)
```

### 📊 `show ip nat translations` 필드 판독

- **Inside local**: 내부 실제 사설 IP (예: 192.168.10.1)
- **Inside global**: 변환된 공인 IP (예: 121.160.23.2)
- **Outside local/global**: 목적지(외부) IP
- **Pro + :포트**: PAT일 때 포트 번호로 세션 구분됨 (`icmp 121.160.23.2:8`)

### 🚨 장애 시나리오 1: 내부 → 외부 통신이 안 됨 (NAT 미동작)

- **원인 파악 (순서대로):**
    1. **inside/outside 방향 지정 누락** — 가장 흔한 원인. 두 인터페이스에 각각 지정돼야 함.
    2. ACL이 변환 대상 사설 대역을 permit하지 않음.
    3. `overload` 키워드 누락으로 공인 IP 1개인데 Dynamic으로 잡혀 주소 고갈.
    4. 외부→내부 경로의 리턴 라우팅 부재.
- **해결:** `show ip nat translations`에 매핑이 생기는지 확인 → 없으면 방향/ACL 점검, 있는데 실패면 리턴 경로 점검.

### 🚨 장애 시나리오 2: 외부에서 내부 서버로 접속이 안 됨

- **증상:** 인터넷에서 내부 웹서버로 접속 시 라우터까지만 가고 서버에 도달 못 함.
- **원인 파악:** **PAT/Dynamic NAT는 역방향을 지원하지 않음.** 외부에서 먼저 시작된 세션은 매핑 엔트리가 없어 내부로 역변환되지 않음.
- **해결:** 해당 서버에 **Static NAT(1:1)** 또는 **포트포워딩**(`ip nat inside source static tcp 192.168.10.1 80 121.160.23.2 80`) 설정.

### 🚨 장애 시나리오 3: 설정을 바꿨는데 이전 NAT가 계속 적용됨

- **원인 파악:** NAT 테이블에 **기존 변환 엔트리가 캐시**되어 있어 새 설정이 반영 안 됨.
- **해결:** `clear ip nat translation *`로 테이블을 비운 뒤 재확인.

> 💡 **면접 한 줄 요약:** NAT의 핵심은 ① inside/outside 방향 지정, ② PAT는 overload로 공인 IP 1개를 포트로 나눠 공유, ③ 외부→내부 공개는 반드시 Static NAT라는 세 가지이며, 설정 변경 시엔 항상 clear로 테이블을 초기화한다.

---

> 관련: 5-2-1. 🧩 NAT & PAT 통합 정리 · 5-2-2. 🚑 NAT & PAT 트러블슈팅 치트시트 · 5-2-3. ⚡ NAT & PAT 명령어 퀵 레퍼런스