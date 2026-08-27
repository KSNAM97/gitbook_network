# GRE Dynamic (라우팅 프로토콜 연동 GRE)

> **Tag:** #network #vpn #gre #eigrp #nat #cisco
> **핵심 요약:** GRE 터널 위에 동적 라우팅(EIGRP)을 실어 사설망 경로를 자동 교환. NAT와 터널의 공존이 핵심

---

## 1. 핵심 기술 개념 (Concept)

### Static Route 대신 GRE 위에 라우팅 프로토콜을 올리는 이유

사설망이 많아지면 Static Route를 일일이 관리하기 어렵고, 경로 변경/장애 시 자동 대응이 안 된다. GRE 터널 인터페이스를 라우팅 프로토콜(EIGRP 등)에 포함시키면, 터널을 통해 양쪽 사설망 경로가 자동으로 교환·갱신되어 관리가 쉽고 장애에 유연하다.

라우팅 프로토콜은 터널 IP를 넥스트홉으로 인접(neighbor)을 맺는다. 즉 공중망이 아니라 터널 위에서 EIGRP Hello가 오간다. `passive-interface default` 상태에서 터널 인터페이스만 `no passive-interface`로 열어줘야 인접이 맺힌다.

### NAT와 GRE 터널이 한 라우터에 같이 있을 때의 공존 관계

NAT와 GRE 터널은 충돌하지 않는다. NAT는 인터넷으로 나가는 일반 트래픽을 공인 IP로 변환(PAT)하고, GRE 터널은 본사↔지사 사설 트래픽을 캡슐화하는 별개의 경로다. 설계상 "지사 간 트래픽은 터널로, 인터넷 트래픽은 NAT로" 분리되어 공존한다.

터널 source를 Loopback으로 잡으면, 물리 회선이 여러 개여도 안정적인 터널 종단점을 유지할 수 있다.

---

## 2. 표준 설정 템플릿 (Configuration)

### GRE 터널 (Loopback을 source로)

```cisco
! === SOL-A ===
SOL-A(config)# interface tunnel 12
SOL-A(config-if)# ip address 172.16.100.1 255.255.255.0
SOL-A(config-if)# tunnel source 100.100.1.1        ! Loopback 0 IP (안정적 종단점)
SOL-A(config-if)# tunnel destination 100.100.2.2

! === SOL-B ===
SOL-B(config)# interface tunnel 12
SOL-B(config-if)# ip address 172.16.100.2 255.255.255.0
SOL-B(config-if)# tunnel source 100.100.2.2
SOL-B(config-if)# tunnel destination 100.100.1.1

### GRE 위에 EIGRP 연동

```cisco
! === 양쪽 공통 : 터널 인터페이스를 EIGRP에 포함 ===
SOL-A(config)# router eigrp 100
SOL-A(config-router)# no passive-interface tunnel 12       ! 터널로 EIGRP Hello 허용
SOL-A(config-router)# network 172.16.100.0 0.0.0.255       ! 터널 대역 포함
```

### 인터넷 트래픽용 NAT (PAT) - 터널과 공존

```cisco
! === SOL-A : 사설망 → 공인(WAN) PAT ===
SOL-A(config)# access-list 10 permit 192.168.10.0 0.0.0.255
SOL-A(config)# ip nat pool SOL-A 121.160.10.1 121.160.10.1 netmask 255.255.255.0
SOL-A(config)# ip nat inside source list 10 pool SOL-A overload
!
SOL-A(config)# interface fastethernet 0/0
SOL-A(config-if)# ip nat inside
SOL-A(config)# interface serial 1/0.10
SOL-A(config-if)# ip nat outside
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 필수 검증 명령어

```cisco
Router# show ip eigrp neighbor          ! 터널 위 EIGRP 인접 확인
Router# show ip route eigrp             ! 학습된 상대 사설망 경로(D) 확인
Router# show ip nat translations        ! NAT 변환 매핑 확인
Router# traceroute [목적지]              ! 실제 경로가 터널/NAT 중 어디로 가는지
```

### 장애 시나리오 1: 터널은 up인데 EIGRP 인접이 안 맺힘

- **원인 파악:** `passive-interface default`는 걸었는데 **터널을 `no passive-interface`로 열지 않음** (가장 흔함). 또는 양쪽 터널 대역 network 문 누락.
- **해결:** 양쪽 라우터에서 터널 인터페이스를 EIGRP에 포함하고 passive 해제 → `show ip eigrp neighbor` 확인.

### 장애 시나리오 2: 사설망끼리는 되는데 인터넷이 안 됨 (또는 반대)

- **원인 파악:** NAT의 inside/outside 방향 지정 누락, 또는 ACL이 사설 대역을 커버 못 함.
- **해결:** `show ip nat translations`로 변환 발생 여부 확인. 터널 트래픽과 인터넷 트래픽 경로가 각각 올바른지 traceroute로 점검.

> 💡 **면접 한 줄 요약:** GRE Dynamic은 터널 위에 라우팅 프로토콜을 올려 사설망 경로를 자동화하는 방식이며, NAT(인터넷용)와 터널(사설망용)은 트래픽 성격이 달라 한 장비에서 문제없이 공존한다.

---

> 관련: 5-3-1-1. 🚇 GRE Static · 5-3-1-3. 🌐 mGRE & NHRP