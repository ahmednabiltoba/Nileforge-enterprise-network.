# Validation Tests

**Version 1.2, October 2026.** 56 tests across the HQ, the perimeter, the branches and the monitoring segment.

## Status values

| Status | Meaning |
|---|---|
| **PASS** | Output captured and it matches the expected result |
| **FAIL** | Output captured and it does not match |
| **INCONCLUSIVE** | Output captured, but it does not prove the claim either way |
| **NOT TESTED** | No operational or traffic evidence. "Configured" means the configuration exists but its behavior was not exercised |
| **UNSUPPORTED** | Packet Tracer rejects or cannot display what the test needs |

A test is never marked PASS because a configuration line exists. It needs operational output or traffic.

> **ID change in v1.2:** the hardening tests are now H1 to H10. In v1.1 they were F1 to F8, which collided with the finding IDs in [`VA_REPORT.md`](VA_REPORT.md).

## Evidence sources

| Code | Source | When |
|---|---|---|
| **S** | Original captures from the build (screenshots `01` to `09`) and the v1.1 test records | September to October 2026 |
| **B1** | Audit batch 1: read-only commands on CORE-SW01 and CORE-SW02 | 21 October 2026 |
| **B2** | Audit batch 2: distribution and all access switches, plus core prechecks | 21 October 2026 |
| **FR** | Final round: verification after fixes, traffic tests, HSRP failover | 21 October 2026 |

## Summary

| Area | Tests | PASS | FAIL | INCONCLUSIVE | NOT TESTED | UNSUPPORTED |
|---|---|---|---|---|---|---|
| A. Switching and routing (HQ) | 11 | 8 | 1 | 1 | 1 | 0 |
| B. ACL policy | 14 | 13 | 0 | 0 | 1 | 0 |
| C. NAT and Internet access | 7 | 4 | 0 | 0 | 3 | 0 |
| D. Branch tunnels | 9 | 7 | 1 | 0 | 1 | 0 |
| E. Monitoring | 5 | 2 | 0 | 0 | 1 | 2 |
| H. Hardening | 10 | 8 | 1 | 0 | 1 | 0 |
| **Total** | **56** | **42** | **3** | **1** | **8** | **2** |

All three failures are known and documented: the access-layer STP mode (A9), the IPsec attempt (D8), and SSH on ACC-SW-SALES (H4).

---

## A. Switching and routing (HQ)

| ID | Test | Command / action | Expected | Actual | Status | Evidence |
|---|---|---|---|---|---|---|
| A1 | HSRP roles | `show standby brief` on both cores | SW01 Active (150), SW02 Standby (100), preempt, groups 10-99 | Matches on all 10 groups | PASS | B1, FR, `03` |
| A2 | HSRP failover and preempt | `shutdown` on Vlan20 of CORE-SW01, then `no shutdown` | SW02 becomes Active for group 20; SW01 takes it back | SW02: `Vl20 20 100 P Active local unknown`. After restore, SW01 Active on all 10 groups | PASS | FR |
| A3 | ACL enforcement while CORE-SW02 is Active | HR PC `ping 10.10.30.1` during A2, then `show ip access-lists HR-IN` on CORE-SW02 | Blocked by CORE-SW02 (unreachable from 10.10.20.3); SW02 deny counter increases | The ping was blocked, but the unreachable came from 10.10.253.1 (CORE-SW01's routed link) and SW02's deny lines stayed at 0. The block cannot be attributed to CORE-SW02's ACL | INCONCLUSIVE | FR |
| A4 | OSPF adjacencies | `show ip ospf neighbor` | FULL | SW01: 3.3.3.3 FULL/DR on Gi1/0/5, 2.2.2.2 FULL/DR on Gi1/0/6. SW02: 1.1.1.1 FULL/BDR. `passive-interface default` keeps the SVIs passive | PASS | B1, B2 |
| A5 | Core EtherChannel | `show etherchannel summary` | Po1 up, members bundled | Po1(SU) LACP, Gi1/0/3(P) Gi1/0/4(P) on both cores | PASS | B1 |
| A6 | Distribution EtherChannel | `show etherchannel summary` | Po2 up, members bundled | Po2(SU) LACP, Fa0/23(P) Fa0/24(P) on both | PASS | B2, FR |
| A7 | Core and distribution trunks | `show interfaces trunk` | VLANs 10-99 allowed | Cores: Po1, Gi1/0/1-2. Distribution: Po2, Gi0/1-2, Fa0/1-10. All allow 10,20,...,99; native VLAN 1 | PASS | B1, B2 |
| A8 | STP root placement | `show spanning-tree vlan 10`, `vlan 20` | CORE-SW01 root | CORE-SW01 root for all 10 VLANs (priority 4096); CORE-SW02 secondary (8192); distribution root port Gi0/1 | PASS | B1, B2 |
| A9 | Access-layer STP mode | `show spanning-tree summary` on all access switches | rapid-pvst | Rapid-PVST on IT, HR, SALES, MANAGEMENT. PVST on MARKETING, FINANCE, VOICE, GUEST, PRINTERS-IOT, MGMT | FAIL (finding F5) | B2, FR |
| A10 | Root guard on distribution downlinks | `show running-config \| include spanning-tree` | Root guard on access-facing ports | 10 `spanning-tree guard root` lines per distribution switch; behavior not exercised | NOT TESTED (configured) | B2 |
| A11 | Access uplink redundancy | `show interfaces trunk`, `show cdp neighbors` on access switches | Two uplinks, one forwarding, one blocking | Fa0/24 to DIST-SW01 forwarding, Fa0/23 to DIST-SW02 blocking, on the nine switches whose trunk output was captured | PASS | B2 |

## B. ACL policy

All "after" counters are from CORE-SW01 in the final round, read straight after the pings. Each counter matches the number of pings sent.

| ID | Test | From | Command | Expected | Actual | Status | Evidence |
|---|---|---|---|---|---|---|---|
| B1 | HR to IT | HR PC | `ping 10.10.10.1` | Replies | 4/4. HR-IN `permit … 10.10.10.0` = 4 matches | PASS | FR |
| B2 | HR to Finance gateway | HR PC | `ping 10.10.30.1` | Blocked | `Destination host unreachable` from 10.10.20.2 | PASS | FR |
| B3 | HR to Sales gateway | HR PC | `ping 10.10.40.1` | Blocked | `Destination host unreachable` from 10.10.20.2 | PASS | S (not re-run) |
| B4 | Guest to IT | Guest laptop | `ping 10.10.10.1` | Blocked | Unreachable from 10.10.80.2. GUEST-IN `deny … 10.0.0.0` = 4 matches | PASS | FR |
| B5 | Guest to Finance | Guest laptop | `ping 10.10.30.1` | Blocked | Unreachable from 10.10.80.2 | PASS | S (not re-run) |
| B6 | HR deny counter | CORE-SW01 | `show ip access-lists HR-IN` | Deny line increments | `deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (8 match(es))`: 4 pings to 10.10.30.1 plus 4 to 10.10.30.11. An earlier session recorded 28 | PASS | FR, `06` |
| B7 | Same ACLs on both cores | Both cores | `show ip access-lists` | Identical | Identical line for line. The 172.16.0.0/12 deny sits after the Data Center permit | PASS | B1 |
| B8 | HR to DMZ | HR PC | `ping 172.16.10.10` | Blocked | Unreachable from 10.10.20.2. `deny … 172.16.0.0 0.15.255.255` = 4 matches | PASS | FR |
| B9 | HR to Data Center | HR PC | `ping 172.16.20.10` | Replies | 3/4 (first lost to ARP). `permit … 172.16.20.0` = 4 matches | PASS | FR |
| B10 | Guest to Internet | Guest laptop | `ping 8.8.8.8` | Replies | 1/4 on the first run (3 timeouts while ARP and NAT warmed up) | PASS | FR |
| B11 | HR host to Finance host | HR PC | `ping 10.10.30.11` | Blocked | Unreachable from 10.10.20.2 | PASS | FR |
| B12 | Guest to Data Center | Guest laptop | `ping 172.16.20.10` | Blocked | Unreachable from 10.10.80.2. GUEST-IN `deny … 172.16.0.0` = 4 matches | PASS | FR |
| B13 | ACL bindings in configuration | Both cores | `show running-config \| include access-group`, `\| begin interface Vlan10` | HR-IN on Vlan20, GUEST-IN on Vlan80 | Both lines present on both cores. On CORE-SW02 they sit under `interface Vlan20` and `interface Vlan80`. `show ip interface` still reports no ACL (known issue 1) | PASS | B1, B2 |
| B14 | Guest DNS to the permitted host | Guest laptop | DNS query to 172.16.20.12 | Allowed | Not captured | NOT TESTED | — |

## C. NAT and Internet access

| ID | Test | Device | Expected | Actual | Status | Evidence |
|---|---|---|---|---|---|---|
| C1 | HQ dynamic PAT | HQ-FW-01 `show xlate` after PC0 ping to 8.8.8.8 | HQ host translated | `ICMP PAT from INSIDE:10.10.10.11/3 to OUTSIDE-ISP1:203.0.113.2/54267` | PASS | S, `04` |
| C2 | Static NAT for DMZ servers from outside | HQ-FW-01 | Inbound traffic translated | Static entries present (172.16.10.10 → 198.51.100.18, 172.16.10.14 → 198.51.100.19), translate_hits 0 | NOT TESTED (configured) | S, `04` |
| C3 | Branch PAT from an endpoint | BR-RTR-Alex `show ip nat translations` | Endpoint translated | `198.51.100.2:2 <- 172.20.1.11:2` to 8.8.8.8, hits 3 | PASS | S |
| C4 | Single NAT rule per branch router | Branch routers `show running-config \| include nat` | Only `list 101` | Only `list 101` on all three | PASS | S |
| C5 | Branch routers reach the Internet | `ping 8.8.8.8` | Replies | 5/5 on all three | PASS | S |
| C6 | PAT from endpoints at Cairo and Giza | Branch PCs | Endpoint translated | Not captured | NOT TESTED | — |
| C7 | DMZ-NET dynamic PAT | HQ-FW-01 `show nat` | Translations | Policy 1 present, translate_hits 0 | NOT TESTED (configured) | S, `04` |

## D. Branch tunnels (GRE, unencrypted)

| ID | Test | From | Command | Result | Status | Evidence |
|---|---|---|---|---|---|---|
| D1 | Alexandria to Cairo tunnel | BR-RTR-Alex | `ping 10.255.0.2` | First run 2/5 (warm-up), second run 5/5 | PASS | S, `07` |
| D2 | Alexandria to Giza tunnel | BR-RTR-Alex | `ping 10.255.0.6` | First run 1/5 (warm-up), second run 5/5 | PASS | S, `07` |
| D3 | Alexandria LAN to Cairo LAN | BR-RTR-Alex | Extended ping to 172.21.99.254, source Gi0/1 | 5/5 | PASS | S, `07` |
| D4 | Alexandria LAN to Giza LAN | BR-RTR-Alex | Extended ping to 172.22.99.254, source Gi0/1 | 5/5 | PASS | S |
| D5 | Cairo to Giza tunnel | BR-RTR-Cairo | `ping 10.255.0.10` | First run 3/5 (warm-up), second run 5/5 | PASS | S, `08` |
| D6 | Giza to Cairo tunnel | BR-RTR-Giza | `ping 10.255.0.9` | 5/5 | PASS | S, `08` |
| D7 | Cairo LAN to Giza LAN | BR-RTR-Cairo | Extended ping, source Gi0/1 | Not captured | NOT TESTED | — |
| D8 | IPsec over GRE | BR-RTR-Alex | `show crypto isakmp sa`, tunnel pings | Cairo peer QM_IDLE with 4 encaps and 4 decaps; Giza peer MM_NO_STATE; tunnel pings 0/5 | FAIL, rolled back | S |
| D9 | Crypto map not applied after rollback | BR-RTR-Cairo, Giza | `show running-config \| include crypto map` | Map entries defined, no interface binding | PASS (tunnels stay unencrypted) | S |

## E. Monitoring

| ID | Test | Device | Expected | Actual | Status | Evidence |
|---|---|---|---|---|---|---|
| E1 | NTP synchronization | Both cores, HQ-FW-01 | Synchronized | Both cores: `Clock is synchronized, stratum 2, reference is 172.16.40.12` (B1 and FR). CORE-SW02 dropped to `.INIT.` once during B2 and resynchronized without changes. HQ-FW-01 from the original capture only | PASS | B1, FR, S, `05` |
| E2 | Syslog to SYSLOG01 | SYSLOG01 Services > SYSLOG | Messages from the cores | Entries from 10.10.254.1 (CORE-SW01) and 10.10.253.2 (CORE-SW02) | PASS | S, `09` |
| E3 | Syslog to SIEM01 | SIEM01 | Messages received | `logging 172.16.40.10` on both cores; receipt not checked | NOT TESTED (configured) | B1 |
| E4 | SNMP | Cores | Poll or trap observed | Read-only community configured; `show snmp` returns nothing | UNSUPPORTED | S |
| E5 | ASA logging | HQ-FW-01 | Logging to 172.16.40.13 | `logging` commands rejected | UNSUPPORTED | S |

The cores send syslog from the outgoing interface address (10.10.254.1 and 10.10.253.2), not from a loopback.

## H. Hardening

| ID | Test | Devices | Expected | Actual | Status | Evidence |
|---|---|---|---|---|---|---|
| H1 | Unused ports parked on the cores | CORE-SW01, CORE-SW02 | Disabled in VLAN 999 | SW01: Gi1/0/8-24. SW02: Gi1/0/5, Gi1/0/7-24. Gi1/0/12 was left enabled on both and was shut down in the final round. Residual: Gi1/1/1-4 are in VLAN 1 and not shut down | PASS | B1, FR |
| H2 | Unused ports parked on distribution and access | 2 distribution, 10 access | Disabled in VLAN 999 | Distribution: Fa0/11-22. All 10 access switches: Fa0/2-21. Residual: access Gi0/1-2 in VLAN 1 and not shut down; ACC-SW-SALES Fa0/22 in VLAN 1 | PASS | B2, FR |
| H3 | SSH version 2 | 13 switches | `SSH Enabled - version 2.0` | CORE-SW01, CORE-SW02, DIST-SW01, DIST-SW02, ACC-SW-IT, -MARKETING, -MANAGEMENT (audit); ACC-SW-HR, -FINANCE, -VOICE, -GUEST, -PRINTERS-IOT, -MGMT (original record) | PASS | B1, B2, FR, S |
| H4 | SSH version 2 on ACC-SW-SALES | ACC-SW-SALES | `SSH Enabled - version 2.0` | B2: `SSH Disabled`. After the final-round fix: `SSH Enabled - version 1.5` (RSA key under 768 bits, so v2 cannot start) | FAIL | B2, FR |
| H5 | Password encryption and hashed local user | Cores, DIST-SW02, IT, MARKETING (audit); seven access switches (original) | `service password-encryption`, user with `secret 5` | Confirmed. Not captured: the local user on ACC-SW-SALES after the fix, and password encryption on DIST-SW01 | PASS | B1, B2, S |
| H6 | VTY 0-15 restricted to SSH | Cores, DIST-SW01, ACC-SW-SALES (final round); IT, MARKETING (B2) | `transport input ssh` on vty 0 4 and 5 15 | Two `transport input ssh` lines on each. CORE-SW02's vty 5 15 had only `login` before the fix. DIST-SW02 not re-captured after the fix | PASS | B2, FR |
| H7 | SSH login from a PC | PC0 to `ssh -l admin 10.10.10.2` | Prompt and banner | `Password:`, then `Authorized access only`, then the CORE-SW01 prompt | PASS | S, `05` |
| H8 | PortFast on access ports | IT, SALES, MARKETING (operational); nine switches (config) | Edge ports in PortFast | `The port is in the portfast mode` on Fa0/1; 22 `spanning-tree portfast` lines per switch | PASS | B2 |
| H9 | BPDU Guard on access ports | Nine access switches | Port err-disables on a BPDU | 22 `spanning-tree bpduguard enable` lines per switch; violation not tested | NOT TESTED (configured) | B2 |
| H10 | Port security | ACC-SW-IT, -HR, -FINANCE, -VOICE, -MGMT | Secure ports active | Fa0/1-20, maximum 2 MACs, violation `restrict`; 1 MAC learned on each connected port, 0 violations. Not configured on SALES, MARKETING, GUEST, PRINTERS-IOT, MANAGEMENT (finding F14) | PASS (on the five switches) | B2, FR |

---

## G. Not covered

- Connectivity between the HQ LAN and the branches (separate networks by design).
- Remote-access VPN (VPNGW01 is a conceptual device only).
- Branch ASAs (ASA4, ASA5, ASA6) and branch switches: present in the topology, configuration not audited.
- DC-CORE-SW01, DMZ-SW01, SEC-MON-SW01 and the servers: reachability shown indirectly (B9, E1, E2), not audited device by device.
- Second ISP (ISP2): present, role not tested.
- Port-security and BPDU Guard violation behavior.
