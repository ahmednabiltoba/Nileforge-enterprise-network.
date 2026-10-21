# Vulnerability Assessment Report (configuration-based)

**Scope:** HQ switching layer and policy devices of the NileForge lab. **Version 1.2, 21 October 2026.**

## Method

Packet Tracer has no vulnerability scanner, so this is a configuration review, similar in spirit to a CIS benchmark check. Each finding comes from `show` output on a device in the lab and carries a severity, the evidence, a remediation and a status.

1. **Baseline (v1.1):** CORE-SW01, DIST-SW01 and ACC-SW-SALES, before any hardening.
2. **Full re-assessment (v1.2):** all 14 HQ switches (2 cores, 2 distribution, 10 access), read-only, followed by a short fix window and a verification round. Details and evidence codes are in [`VALIDATION_TESTS.md`](VALIDATION_TESTS.md).

Main commands:

```
show interfaces status
show interfaces trunk
show vlan brief
show spanning-tree summary
show ip ssh
show port-security
show ip access-lists
show standby
show ip route
show running-config | include <keyword>
```

Running-configuration output was filtered by keyword so that password hashes and keys were not displayed.

## Findings

| ID | Finding | Severity | Evidence | Remediation | Status |
|---|---|---|---|---|---|
| F1 | Native VLAN 1 on every trunk | Low | `show interfaces trunk` on cores, distribution and access switches: native VLAN 1 | Move to a dedicated unused VLAN on both ends of every trunk | **Accepted risk.** Needs a matched change across 14 switches |
| F2 | Unused ports left in VLAN 1 and enabled | Low | Baseline: CORE-SW01 Gi1/0/8+, DIST-SW01 Fa0/12-22, ACC-SW-SALES Fa0/2-21 notconnect in VLAN 1 | Park in VLAN 999 and shut down | **Remediated, residual open.** Parked on all 14 switches (H1, H2). Gi1/0/12 on both cores was found re-enabled and shut down again. Residual: Gi1/1/1-4 on the cores and Gi0/1-2 on the access switches remain in VLAN 1 and enabled; ACC-SW-SALES Fa0/22 in VLAN 1 |
| F3 | Passwords stored in readable form | Low | `no service password-encryption` on the baseline devices | `service password-encryption` and `secret` for local users | **Remediated.** Encryption on and the user stored as `secret 5` on every device checked (H5) |
| F4 | Management lines not limited to SSH | Medium | No `transport input` under the VTY lines on the baseline devices | RSA keys, SSH v2, local user, `transport input ssh` on vty 0-15, `exec-timeout`, banner | **Remediated, one exception.** SSH v2 on 13 of 14 switches (H3). ACC-SW-SALES runs SSH 1.5 because its RSA key is under 768 bits (H4). vty 5 15 lacked SSH-only settings on the cores until the fix window (H6) |
| F5 | Access layer runs PVST, not Rapid-PVST | Low | `show spanning-tree summary`: `pvst` | `spanning-tree mode rapid-pvst` on each access switch | **Partially remediated.** Rapid-PVST on IT, HR, SALES, MANAGEMENT; PVST on six switches (A9) |
| F6 | SNMP community in clear text and not restricted | Low | `snmp-server community <REDACTED> RO` without an ACL on the cores | Bind the community to the monitoring host and use SNMPv3 | **Open.** The simulator rejects the ACL binding |
| F7 | `HR-IN` ended with a broad permit, so HR reached the DMZ and Security segments | Medium | Baseline `show ip access-lists`: the final line permitted HR to any destination | `deny ip 10.10.20.0 0.0.0.255 172.16.0.0 0.15.255.255` after the Data Center permit | **Remediated and verified.** Same position on both cores (B7). HR to 172.16.10.10 blocked with 4 matches on the new line (B8) |
| F8 | Lab credentials reused and present in build documents | Medium | One value used for a local account, an enable secret and an IPsec key in earlier command files. The same admin hash appears on every HQ switch | Unique secrets per purpose; `<REDACTED>` in published files | **Open** until the repository is scrubbed |
| F9 | Inter-branch GRE traffic is not encrypted | Medium | IPsec attempt: one peer `QM_IDLE`, one `MM_NO_STATE`, tunnel pings failed (D8) | IPsec protection of the GRE tunnels | **Accepted risk.** Rolled back; crypto map not applied |
| F10 | CORE-SW01 is the only upstream path for the HQ | Medium (availability) | CORE-SW02: Gi1/0/5 and Gi1/0/7 parked, default route `via 10.10.253.1`, Data Center routes learned through CORE-SW01 | Link CORE-SW02 to DC-CORE-SW01 and the firewall (or a second firewall interface), and route both cores upstream | **Open.** Documented design limitation |
| F11 | HSRP runs without authentication | Medium | `show running-config \| include standby`: no authentication on either core | `standby <group> authentication md5 key-string <key>` if the simulator accepts it | **Open** |
| F12 | VTY lines accept SSH from any internal source | Low | No `access-class` on either core | Standard ACL for the MGMT and IT subnets, applied with `access-class <name> in` on vty 0-15 | **Open** |
| F13 | Management department PC sits in VLAN 99 (MGMT) instead of VLAN 60 | Low | ACC-SW-MANAGEMENT Fa0/1 in VLAN 99; no switch has an access port in VLAN 60 | Move the port to VLAN 60 and readdress the PC, or document VLAN 99 use as intended | **Open.** Intent to be confirmed |
| F14 | Port security missing on five access switches | Low | `show port-security` empty on SALES, MARKETING, GUEST, PRINTERS-IOT and MANAGEMENT | Same template as the other five (maximum 2, restrict) | **Open** |

## Summary

| Status | Count | Findings |
|---|---|---|
| Remediated | 2 | F3, F7 |
| Remediated with residual items | 2 | F2, F4 |
| Partially remediated | 1 | F5 |
| Accepted risk | 2 | F1, F9 |
| Open | 7 | F6, F8, F10, F11, F12, F13, F14 |
| **Total** | **14** | |

## What a re-assessment still needs

1. ACC-SW-SALES: regenerate the RSA key at 1024 bits or more and confirm `SSH Enabled - version 2.0`.
2. `spanning-tree mode rapid-pvst` on the six remaining access switches.
3. Decide on F13 (VLAN 60 or documented intent) and F14 (port security coverage).
4. HSRP authentication and a VTY `access-class`, if the simulator accepts them.
5. A scrub of published configuration files and screenshots for credentials and the SNMP community.

## Limits of this assessment

It reads configuration and observes traffic, so it cannot find software vulnerabilities, weak protocols in the simulated servers, or exposed services. The branch devices, branch ASAs, DMZ switch and data center switch were not audited. A scanner-based assessment (for example OpenVAS against an intentionally vulnerable target) belongs in a separate lab.
