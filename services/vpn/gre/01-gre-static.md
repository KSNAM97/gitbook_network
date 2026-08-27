# GRE Static (정적 GRE 터널)

> **Tag:** #network #vpn #gre #tunnel #cisco
> **핵심 요약:** 공중망(ISP)을 가로질러 두 사설망을 잇는 논리적 P2P 터널. Static Route와 결합해 사설망 통신 구현

---

## 1. 핵심 기술 개념 (Concept)

### GRE 터널을 쓰는 이유와 사설 IP의 공중망 통과 방식

사설 IP는 ISP 라우터에서 라우팅되지 않으므로 그대로는 본사↔지사 통신이 불가능하다. GRE는 원래 사설 IP 패킷을 공인 IP(New IP) + GRE 헤더로 한 번 더 캡슐화(encapsulation)해서, ISP 입장에선 그냥 공인 IP 트래픽으로 보이게 만들어 터널 반대편까지 전달한다.

캡슐화 구조는 `[New IP(공인 SA/DA)] [GRE] [원본 IP(사설)] [Data]`이며, ISP는 바깥 공인 IP만 보고 라우팅한다. 터널의 `source`/`destination`은 양 끝단 라우터의 공인 IP이며, 이 목적지 공인 IP가 라우팅 테이블에 있어야만 터널 인터페이스가 up 된다.

### 터널 생성 후 사설망끼리 통신이 안 되는 이유

GRE 터널이 up 되어 터널 IP(예: 172.16.10.x)끼리는 통신되어도, 각 라우터가 상대편 사설망(192.168.x.0) 경로를 모르기 때문에 사설망 간 통신은 되지 않는다. 터널을 넥스트홉으로 하는 Static Route(또는 라우팅 프로토콜)를 추가해야 사설망 간 통신이 된다.

ISP의 라우팅 테이블에는 사설 대역(172.16.x, 192.168.x)이 절대 나타나지 않는다(터널 안에 숨겨져 있음). 터널 활성화 조건은 destination 공인 IP가 라우팅 가능해야 한다는 것이며, source만 설정하면 여전히 down 상태다.

---

## 2. 표준 설정 템플릿 (Configuration)

### GRE Static Tunnel 생성

```cisco
! === SOL-A (본사) ===
SOL-A(config)# interface tunnel 12
SOL-A(config-if)# ip address 172.16.10.1 255.255.255.0   ! 터널 논리 IP
SOL-A(config-if)# tunnel source 121.160.10.1              ! 자신의 공인 IP
SOL-A(config-if)# tunnel destination 121.160.20.2         ! 상대 공인 IP

! === SOL-B (지사) : source/destination 반대로 ===
SOL-B(config)# interface tunnel 12
SOL-B(config-if)# ip address 172.16.10.2 255.255.255.0
SOL-B(config-if)# tunnel source 121.160.20.2
SOL-B(config-if)# tunnel destination 121.160.10.1
```

### 사설망 연결용 Static Route

```cisco
! === SOL-A : 지사 사설망으로 가는 경로를 터널로 지정 ===
SOL-A(config)# ip route 192.168.20.0 255.255.255.0 172.16.10.2   ! 넥스트홉=상대 터널IP
!                또는
SOL-A(config)# ip route 192.168.20.0 255.255.255.0 tunnel 12     ! 터널 인터페이스 지정

! === SOL-B : 본사 사설망으로 가는 경로 ===
SOL-B(config)# ip route 192.168.10.0 255.255.255.0 172.16.10.1
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Router# show ip interface brief          ! 터널 인터페이스 up/down 확인
Router# show interface tunnel 12          ! 터널 상세 상태
Router# show ip route                     ! 터널 경로(C) + 사설망 경로(S) 확인
Router# ping 172.16.10.2                   ! 터널 IP 소통 테스트
```

### 장애 시나리오 1: 터널 인터페이스가 Down

- **원인 파악:** 터널 **destination 공인 IP가 라우팅 테이블에 없음.** GRE는 목적지 공인 IP로 도달 가능해야만 활성화됨.
- **해결:** 공중망 라우팅(OSPF/EIGRP 등)으로 상대 공인 IP가 학습됐는지 `show ip route`로 확인. source만 설정하면 여전히 down임을 유의.

### 장애 시나리오 2: 터널 IP는 되는데 사설망끼리 안 됨

- **원인 파악:** 상대 사설망으로 가는 **Static Route(또는 라우팅) 누락.** `show ip route`에 상대 사설 대역이 없음.
- **해결:** `ip route [상대사설망] [마스크] tunnel X` 추가 후 PC에서 재확인.

> 💡 **면접 한 줄 요약:** GRE Static은 "공인 IP로 한 번 더 감싸 사설 패킷을 공중망에 태우는 P2P 터널"이며, 터널 up 조건(destination 도달성)과 사설망 라우팅 추가가 두 가지 필수 체크포인트다.


---

]