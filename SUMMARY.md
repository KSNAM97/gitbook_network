# Summary

* [소개](README.md)

## Network 기본

* [IPv4 Addressing & Subnetting & VLSM](network-basics/01-ipv4-addressing-subnetting-vlsm.md)
* [Cisco 라우터 기초 (CDP·Mode·Interface·Password)](network-basics/02-cisco-router-basics-cdp-mode-interface-password.md)
* [NTP & SSH & Telnet](network-basics/03-ntp-ssh-telnet.md)

## Routing — Router Basic

* [라우팅 기초 개념](routing/router-basics/01-routing-concept.md)
* [Static Route](routing/router-basics/02-static-route.md)
* [라우터의 스위칭 방식 (CEF)](routing/router-basics/03-router-switching-cef.md)
* [라우팅 기초 통합 정리](routing/router-basics/04-routing-basics-summary.md)
* [라우팅 기초 트러블슈팅 치트시트](routing/router-basics/05-routing-basics-troubleshooting-cheatsheet.md)
* [라우팅 기초 퀵 레퍼런스](routing/router-basics/06-routing-basics-quick-reference.md)
* [Frame-relay (WAN L2 Protocol)](routing/router-basics/07-frame-relay.md)
* [Frame-relay 통합 정리](routing/router-basics/08-frame-relay-summary.md)
* [Frame-relay 트러블슈팅 치트시트](routing/router-basics/09-frame-relay-troubleshooting-cheatsheet.md)
* [Frame-relay 퀵 레퍼런스](routing/router-basics/10-frame-relay-quick-reference.md)

## Routing — Dynamic Routing Protocol

### RIP

* [RIP v1 & v2](routing/dynamic-routing/rip/01-rip.md)
* [RIP 통합 정리](routing/dynamic-routing/rip/02-rip-summary.md)
* [RIP 트러블슈팅 치트시트](routing/dynamic-routing/rip/03-rip-troubleshooting-cheatsheet.md)
* [RIP 퀵 레퍼런스](routing/dynamic-routing/rip/04-rip-quick-reference.md)

### EIGRP

* [EIGRP](routing/dynamic-routing/eigrp/01-eigrp.md)
* [EIGRP 통합 정리](routing/dynamic-routing/eigrp/02-eigrp-summary.md)
* [EIGRP 트러블슈팅 치트시트](routing/dynamic-routing/eigrp/03-eigrp-troubleshooting-cheatsheet.md)
* [EIGRP 퀵 레퍼런스](routing/dynamic-routing/eigrp/04-eigrp-quick-reference.md)

### OSPF

* [OSPF](routing/dynamic-routing/ospf/01-ospf.md)
* [OSPF 통합 정리](routing/dynamic-routing/ospf/02-ospf-summary.md)
* [OSPF 트러블슈팅 치트시트](routing/dynamic-routing/ospf/03-ospf-troubleshooting-cheatsheet.md)
* [OSPF 퀵 레퍼런스](routing/dynamic-routing/ospf/04-ospf-quick-reference.md)

### BGP

* [BGP 이론 (BGPv4)](routing/dynamic-routing/bgp/01-bgp-theory.md)
* [BGP Peer 연결 (Physical · Loopback · Multihop)](routing/dynamic-routing/bgp/02-bgp-peer-connection.md)
* [BGP Split-horizon 문제 해결 (Full-mesh · Route-reflector · Confederation)](routing/dynamic-routing/bgp/03-bgp-split-horizon.md)
* [BGP Next-hop 문제 해결 (Static · Redistribute · Next-hop-self)](routing/dynamic-routing/bgp/04-bgp-next-hop.md)
* [BGP 통합 정리 — AS·Peer·Split-horizon·Next-hop 한눈에](routing/dynamic-routing/bgp/05-bgp-summary.md)
* [BGP 트러블슈팅 치트시트](routing/dynamic-routing/bgp/06-bgp-troubleshooting-cheatsheet.md)
* [BGP 명령어 퀵 레퍼런스](routing/dynamic-routing/bgp/07-bgp-quick-reference.md)

## Routing — Redistribution

* [Route Redistribution with Route-Map Filtering](routing/redistribution/01-route-redistribution.md)
* [라우팅 재분배 통합 정리](routing/redistribution/02-redistribution-summary.md)
* [라우팅 재분배 트러블슈팅 치트시트](routing/redistribution/03-redistribution-troubleshooting-cheatsheet.md)
* [라우팅 재분배 퀵 레퍼런스](routing/redistribution/04-redistribution-quick-reference.md)

## ACL — ACL 기초 & 응용

* [Standard ACL (Numbered & Named)](acl/acl-basics/01-standard-acl.md)
* [Extended ACL & 라우팅 프로토콜 예외 처리](acl/acl-basics/02-extended-acl-routing-protocol-exceptions.md)
* [고급 ACL - Reflective & Dynamic & Time-range](acl/acl-basics/03-advanced-acl-reflective-dynamic-time-range.md)
* [ACL 통합 정리](acl/acl-basics/04-acl-summary.md)
* [ACL 트러블슈팅 치트시트](acl/acl-basics/05-acl-troubleshooting-cheatsheet.md)
* [ACL 명령어 퀵 레퍼런스](acl/acl-basics/06-acl-quick-reference.md)

## ACL — 라우팅 필터링

* [Prefix-list 기초 & Bogon Filter](acl/route-filtering/01-prefix-list-bogon-filter.md)
* [Distribute-list를 활용한 라우팅 필터링](acl/route-filtering/02-distribute-list.md)
* [Route-map을 활용한 정책 기반 필터링 & Tagging](acl/route-filtering/03-route-map-policy-filtering-tagging.md)
* [Offset-list를 활용한 경로 조작](acl/route-filtering/04-offset-list.md)
* [라우팅 필터링 & 정책 제어 통합 정리](acl/route-filtering/05-route-filtering-summary.md)
* [라우팅 필터링 & 정책 제어 트러블슈팅 치트시트](acl/route-filtering/06-route-filtering-troubleshooting-cheatsheet.md)
* [라우팅 필터링 & 정책 제어 퀵 레퍼런스](acl/route-filtering/07-route-filtering-quick-reference.md)

## ACL — 방어 보안

* [스위치 & 방어 보안 - Port-security & TCP Intercept](acl/defensive-security/01-port-security-tcp-intercept.md)
* [스위치 & 방어 보안 통합 정리](acl/defensive-security/02-defensive-security-summary.md)
* [스위치 & 방어 보안 트러블슈팅 치트시트](acl/defensive-security/03-defensive-security-troubleshooting-cheatsheet.md)
* [스위치 & 방어 보안 명령어 퀵 레퍼런스](acl/defensive-security/04-defensive-security-quick-reference.md)

## Switch & L2/L3 — 스위치 기본

* [스위치 동작원리 & VLAN 기초](switch/switch-basics/01-switch-operation-vlan-basics.md)
* [Switchport & Trunk & Inter-VLAN](switch/switch-basics/02-switchport-trunk-inter-vlan.md)
* [스위치 기본 & VLAN·Trunk 통합 정리](switch/switch-basics/03-switch-vlan-trunk-summary.md)
* [스위치 기본 & VLAN·Trunk 트러블슈팅 치트시트](switch/switch-basics/04-switch-vlan-trunk-troubleshooting-cheatsheet.md)
* [스위치 기본 & VLAN·Trunk 퀵 레퍼런스](switch/switch-basics/05-switch-vlan-trunk-quick-reference.md)
* [Switch Password 복구](switch/switch-basics/06-switch-password-recovery.md)
* [Switch Password 복구 통합 정리](switch/switch-basics/07-switch-password-recovery-summary.md)
* [Switch Password 복구 트러블슈팅 치트시트](switch/switch-basics/08-switch-password-recovery-troubleshooting-cheatsheet.md)
* [Switch Password 복구 퀵 레퍼런스](switch/switch-basics/09-switch-password-recovery-quick-reference.md)

## Switch & L2/L3 — VTP

* [VTP 개념 & Mode & 동기화](switch/vtp/01-vtp.md)
* [VTP 통합 정리](switch/vtp/02-vtp-summary.md)
* [VTP 트러블슈팅 치트시트](switch/vtp/03-vtp-troubleshooting-cheatsheet.md)
* [VTP 명령어 퀵 레퍼런스](switch/vtp/04-vtp-quick-reference.md)

## Switch & L2/L3 — Spanning-tree (STP)

* [STP 기본 원리 & Port 상태 & Convergence](switch/stp/01-stp-basics-port-states-convergence.md)
* [Fast 계열 (PortFast, UplinkFast, BackboneFast)](switch/stp/02-portfast-uplinkfast-backbonefast.md)
* [STP 보호 기술 (BPDUguard & filter, LoopGuard, RootGuard)](switch/stp/03-stp-protection-bpduguard-loopguard-rootguard.md)
* [RSTP & MSTP](switch/stp/04-rstp-mstp.md)
* [STP·RSTP·MSTP 통합 정리](switch/stp/05-stp-rstp-mstp-summary.md)
* [STP·RSTP·MSTP 트러블슈팅 치트시트](switch/stp/06-stp-rstp-mstp-troubleshooting-cheatsheet.md)
* [STP·RSTP·MSTP 퀵 레퍼런스](switch/stp/07-stp-rstp-mstp-quick-reference.md)

## Switch & L2/L3 — Multi-Layer Switch (L3 Switch)

* [Routed Interface & SVI & Inter-VLAN Routing](switch/l3-switch/01-routed-interface-svi-inter-vlan-routing.md)
* [L3 Switch & Inter-VLAN Routing 통합 정리](switch/l3-switch/02-l3-switch-summary.md)
* [L3 Switch & Inter-VLAN Routing 트러블슈팅 치트시트](switch/l3-switch/03-l3-switch-troubleshooting-cheatsheet.md)
* [L3 Switch & Inter-VLAN Routing 퀵 레퍼런스](switch/l3-switch/04-l3-switch-quick-reference.md)

## Switch & L2/L3 — EtherChannel

* [EtherChannel 핵심 (PAgP & LACP & L2·L3)](switch/etherchannel/01-etherchannel-pagp-lacp.md)
* [통합 실습 부가기술 (DHCP & NTP & Anti-Frag ACL)](switch/etherchannel/02-integration-lab-dhcp-ntp-anti-frag-acl.md)
* [EtherChannel 통합 정리](switch/etherchannel/03-etherchannel-summary.md)
* [EtherChannel 트러블슈팅 치트시트](switch/etherchannel/04-etherchannel-troubleshooting-cheatsheet.md)
* [EtherChannel 명령어 퀵 레퍼런스](switch/etherchannel/05-etherchannel-quick-reference.md)

## Switch & L2/L3 — Switch 보안

* [Port-security & Storm-control](switch/switch-security/01-port-security-storm-control.md)
* [SPAN (Port Mirroring)](switch/switch-security/02-span-port-mirroring.md)
* [Switch 보안 통합 정리](switch/switch-security/03-switch-security-summary.md)
* [Switch 보안 트러블슈팅 치트시트](switch/switch-security/04-switch-security-troubleshooting-cheatsheet.md)
* [Switch 보안 명령어 퀵 레퍼런스](switch/switch-security/05-switch-security-quick-reference.md)

## Switch & L2/L3 — Gateway 이중화 (FHRP)

* [HSRP & VRRP (Gateway Redundancy)](switch/fhrp/01-hsrp-vrrp.md)
* [HSRP & VRRP 통합 정리](switch/fhrp/02-hsrp-vrrp-summary.md)
* [HSRP & VRRP 트러블슈팅 치트시트](switch/fhrp/03-hsrp-vrrp-troubleshooting-cheatsheet.md)
* [HSRP & VRRP 명령어 퀵 레퍼런스](switch/fhrp/04-hsrp-vrrp-quick-reference.md)

## IP 서비스 & 보안 — DHCP

* [DHCP (동적 IP 할당)](services/dhcp/01-dhcp.md)
* [DHCP 통합 정리](services/dhcp/02-dhcp-summary.md)
* [DHCP 트러블슈팅 치트시트](services/dhcp/03-dhcp-troubleshooting-cheatsheet.md)
* [DHCP 명령어 퀵 레퍼런스](services/dhcp/04-dhcp-quick-reference.md)

## IP 서비스 & 보안 — NAT

* [NAT & PAT (주소 변환)](services/nat/01-nat-pat.md)
* [NAT & PAT 통합 정리](services/nat/02-nat-pat-summary.md)
* [NAT & PAT 트러블슈팅 치트시트](services/nat/03-nat-pat-troubleshooting-cheatsheet.md)
* [NAT & PAT 명령어 퀵 레퍼런스](services/nat/04-nat-pat-quick-reference.md)

## IP 서비스 & 보안 — VPN

* [GRE Static](services/vpn/gre/01-gre-static.md)
* [GRE Dynamic](services/vpn/gre/02-gre-dynamic.md)
* [mGRE & NHRP](services/vpn/gre/03-mgre-nhrp.md)
* [IPsec VPN (암호화 · AH&ESP · IKE)](services/vpn/ipsec/01-ipsec-vpn.md)
* [CBAC (Stateful Firewall)](services/vpn/cbac/01-cbac-stateful-firewall.md)
* [GRE-Over-IPsec (라우팅 암호화)](services/vpn/gre-over-ipsec/01-gre-over-ipsec.md)
* [DMVPN (동적 멀티포인트 VPN)](services/vpn/dmvpn/01-dmvpn.md)
* [IPsec 이중화 (HSRP 연동)](services/vpn/ipsec-redundancy/01-ipsec-redundancy-hsrp.md)
* [VPN 통합 정리 (GRE · IPsec · DMVPN)](services/vpn/07-vpn-summary.md)
* [VPN 트러블슈팅 치트시트](services/vpn/08-vpn-troubleshooting-cheatsheet.md)
* [VPN 명령어 퀵 레퍼런스](services/vpn/09-vpn-quick-reference.md)

## 연습문제

* [Network 종합 연습문제](exercises/01-network-comprehensive-exercises.md)
