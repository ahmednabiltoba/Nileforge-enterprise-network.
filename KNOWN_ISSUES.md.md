# Known Issues

**Version 1.2, October 2026.** Grouped by cause, so a reader can tell a simulator limit from an unproven item from a design choice. Test IDs refer to [`VALIDATION_TESTS.md`](VALIDATION_TESTS.md); finding IDs refer to [`VA_REPORT.md`](VA_REPORT.md).

## 1. Packet Tracer limitations (simulator behavior, not configuration errors)

| # | Issue | Evidence | Workaround used |
|---|---|---|---|
| 1 | `show ip interface Vlan20` and `Vlan80` report `Inbound access list is not set` on both cores, although the running configuration contains `ip access-group HR-IN in` and `ip access-group GUEST-IN in` and traffic is filtered | B13; traffic tests B1-B12 | Prove enforcement with blocked and allowed pings plus match counters |
| 2 | ACL match counters reset when the `.pkt` file is reopened. The final `permit … any` lines also count control-plane traffic such as the peer core's HSRP hellos, so those counts are not evidence of user traffic | CORE-SW02 (standby) showed 246 to 963 matches on its permit-any lines | Read counters immediately after a known number of pings. In the final round every counter matched the pings sent |
| 3 | `snmp-server community … RO <acl>` is rejected, and `show snmp` returns no output | Invalid input marker on the ACL name | Community configured without an ACL (finding F6) |
| 4 | ASA `logging` and `snmp-server` commands, `show logging`, `show running-config logging` and `ping INSIDE …` are rejected. Only `ntp server` was accepted | Invalid input markers | NTP only on HQ-FW-01 |
| 5 | `crypto key generate rsa modulus 2048` is not accepted | Invalid input marker on `modulus` | Use `crypto key generate rsa` and answer the size prompt with 1024 or more. A smaller key leaves SSH at version 1.5, which is what happened on ACC-SW-SALES (H4) |
| 6 | IPsec over GRE was unstable: one peer reached `QM_IDLE` with encaps and decaps, the other stayed at `MM_NO_STATE`, and tunnel pings failed with the crypto map applied | D8 | Crypto map removed from the interfaces; tunnels left as plain GRE |
| 7 | During the HSRP failover test, HR traffic to Finance was blocked, but the unreachable came from 10.10.253.1 (CORE-SW01's routed link) and CORE-SW02's deny counters stayed at 0. Earlier, `show standby brief` on the other core showed inconsistent states for a while after a single-SVI shutdown | A2, A3 | Failover itself proven by the role change and preempt. Post-failover ACL enforcement recorded as INCONCLUSIVE |
| 8 | The syslog server shows the sender IP in the HostName column, and entries logged before NTP sync carry an old date | `09` screenshot | Recorded. Sender IPs are 10.10.254.1 and 10.10.253.2 |
| 9 | The first ping across a multi-hop path after idle time loses packets while ARP and the tunnel warm up: Alexandria to Cairo 2/5, Alexandria to Giza 1/5, Cairo to Giza 3/5, HR to the Data Center 3/4, Guest to 8.8.8.8 1/4 | D1, D2, D5, B9, B10 | Repeat the ping |
| 10 | `show nat` on the ASA skips a policy number (1, 2, 3, 5) | `04` screenshot | Cosmetic |
| 11 | CORE-SW02's NTP association dropped to `.INIT.` (stratum 16, reach 0) once during the audit and resynchronized without any change | B2 vs FR | Synchronization confirmed again in the final round |
| 12 | Commands rejected on the 3650: `show running-config interface X`, `show ip ospf interface <SVI>`, `show standby \| include …` | B1, B2 | `show interfaces X switchport`, `show running-config \| include passive-interface`, full `show standby` |
| 13 | `show version` prints "test uptime" and the same serial number on both 3650s | B1 | Template data; identity taken from the prompt and CDP |
| 14 | For VLANs 10 and 50 only, CORE-SW02 Gi1/0/2 is Alternate/Blocking and DIST-SW02 Gi0/2 is Designated, which does not follow the path-cost election. Both ends agree and the topology is loop-free | B1, B2 | No action; no claim made about it |

## 2. Configured but not verified

- Static NAT for WEB01 (198.51.100.18) and MAILGW01 (198.51.100.19) from outside, and the DMZ-NET dynamic PAT (C2, C7). Translate hits are 0. INTERNET-SIM must also route 198.51.100.18-19 toward HQ-FW-01, which was not checked.
- ACL enforcement on CORE-SW02 while it is the active gateway (A3).
- Guest DNS to the permitted host 172.16.20.12 (B14).
- Syslog receipt on SIEM01 (E3); SNMP polling (E4).
- Root guard (A10), BPDU Guard (H9) and port-security violation behavior.
- PAT from endpoints at Cairo and Giza (C6) and Cairo LAN to Giza LAN (D7).
- NTP on HQ-FW-01 was not re-captured during the audit.
- After the final-round fixes, not re-captured: the local user on ACC-SW-SALES and the vty 5 15 lines on DIST-SW02.

## 3. Design choices, limitations and accepted risks

- **CORE-SW01 is the only upstream path** (finding F10). CORE-SW02 has no link to DC-CORE-SW01 or HQ-FW-01; its default route points to CORE-SW01 (10.10.253.1) and it learns the Data Center and Security segments through it. HSRP protects the user gateway, but a full CORE-SW01 failure would cut the HQ VLANs off from the Data Center, the Security segment and the Internet.
- **No management IP on the distribution and access switches.** SSH is configured on them, but it is reachable only on the two cores; the rest are managed from the console.
- **HQ endpoints are statically addressed.** No `ip helper-address` is configured on either core, so DHCP01 does not serve the HQ VLANs.
- **The Management department PC is in VLAN 99 (MGMT)**, not VLAN 60 (finding F13). VLAN 60 has no access port on any switch.
- **HQ and branches are separate.** No path between the HQ LAN and the branch LANs. Intentional scoping.
- **GRE tunnels are unencrypted** (F9). Crypto map entries remain defined on the branch routers but are not applied.
- **Native VLAN 1** on every trunk (F1). Changing it needs a matched change on both ends of every trunk across 14 switches.
- **Six access switches still run PVST** (F5).
- **No HSRP authentication** (F11) and **no `access-class` on the VTY lines** (F12).
- **One shared local admin credential** across the HQ switches (part of F8).
- **No AAA `new-model`.** Management uses local accounts through `login local`.
- **No hardening on the ASA or the branch devices.** Hardening covers the HQ switching layer only.
- **HQ-FW-02 is a spare.** It is not cabled or configured.
- **VPNGW01 and the remote user are conceptual.** Packet Tracer does not negotiate a client VPN.
- **Published files carry placeholders.** Credentials and the SNMP community string are replaced with `<REDACTED>` in every published file.
