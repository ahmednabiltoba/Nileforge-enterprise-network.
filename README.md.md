# NileForge Secure Enterprise Network (Cisco Packet Tracer Lab)

A multi-site enterprise network that I designed, configured, audited and validated in Cisco Packet Tracer. It includes a three-tier headquarters campus with redundant Layer 3 cores, an ASA perimeter with a DMZ, a data center, a security and monitoring segment, and three branch offices (Alexandria, Cairo, Giza) joined by a GRE full mesh.

> **Status:** Version 1.2, October 2026.
> **Lab disclaimer:** This is a Packet Tracer simulation built as a portfolio project. It is not a production network and was not deployed in a real organization. Every result marked PASS is backed by captured output. Anything I could not prove is marked INCONCLUSIVE, NOT TESTED or UNSUPPORTED in [`VALIDATION_TESTS.md`](VALIDATION_TESTS.md) and explained in [`KNOWN_ISSUES.md`](KNOWN_ISSUES.md).

---

## Contents

- [Scenario](#scenario)
- [Objectives](#objectives)
- [Architecture and topology](#architecture-and-topology)
- [Addressing and segmentation](#addressing-and-segmentation)
- [Routing and switching design](#routing-and-switching-design)
- [Security controls](#security-controls)
- [Testing and validation](#testing-and-validation)
- [Key design decisions](#key-design-decisions)
- [Known limitations](#known-limitations)
- [Skills demonstrated](#skills-demonstrated)
- [How to explore the project](#how-to-explore-the-project)
- [Repository structure](#repository-structure)
- [Future improvements](#future-improvements)
- [Version history](#version-history)
- [Author](#author)

---

## Scenario

NileForge is a fictional company with a headquarters and three branch offices in Egypt. The headquarters needs department separation, a redundant gateway for users, a controlled path to the Internet, public-facing servers kept apart from the internal LAN, a central data center, and a place to collect logs and time. The branches need local Internet access and connectivity to each other.

## Objectives

1. Build a three-tier campus (core, distribution, access) with a redundant first-hop gateway for every user VLAN.
2. Segment departments into VLANs and restrict traffic from the higher-risk HR and Guest VLANs at the gateway.
3. Put a firewall at the perimeter with NAT for users and a DMZ for public servers.
4. Connect three branches to the Internet and to each other.
5. Add time synchronization and central logging.
6. Harden the switching layer, then audit every HQ switch and record what remains open.
7. Prove each claim with captured output, and state clearly what could not be proven.

---

## Architecture and topology

![HQ topology: data center, perimeter and DMZ, core, distribution and access layers](screenshots/01-topology-hq.png)
*Headquarters: data center (top left), security and monitoring segment (bottom left), ASA perimeter and DMZ (top center), core pair, distribution pair and the ten access switches (right).*

![Branch topology: Alexandria, Cairo and Giza](screenshots/02-topology-branches.png)
*Branches: each site has a router, an ASA, a Layer 3 switch and access switches. Cairo is the largest site, with two multilayer switches.*

| Zone | Devices | Role |
|---|---|---|
| Core | CORE-SW01, CORE-SW02 (Catalyst 3650) | VLAN SVIs, HSRP gateways, OSPF area 0, inbound ACLs |
| Distribution | DIST-SW01, DIST-SW02 (Catalyst 3560) | Layer 2 aggregation, LACP Po2 between the pair, root guard toward the access layer |
| Access | ACC-SW-IT, -HR, -FINANCE, -SALES, -MARKETING, -MANAGEMENT, -VOICE, -GUEST, -PRINTERS-IOT, -MGMT (Catalyst 2960) | One department each, dual-homed to both distribution switches |
| Perimeter and DMZ | HQ-FW-01 (ASA), ISP1, ISP2, INTERNET-SIM, DMZ-SW01, WEB01, REVPROXY01, DNS-DMZ01, VPNGW01, MAILGW01 | PAT for the HQ LAN and static NAT for DMZ servers. HQ-FW-02 is a spare and is not in service |
| Data Center | DC-CORE-SW01, DC01, DC02, DNS01, DHCP01, FILE01, APP01, DB01, WEB02, BACKUP01, WSUS01 | Central servers, routed to the core over OSPF |
| Security and Monitoring | SEC-MON-SW01, SIEM01, NIDS01, MON01, SYSLOG01 | NTP source (MON01) and syslog collection |
| Branches | Router, ASA, Layer 3 switch and access switches at Alexandria, Cairo and Giza | Local LAN, PAT to the Internet, GRE tunnels between sites |

### HQ physical links (confirmed with CDP)

| From | To | Type |
|---|---|---|
| CORE-SW01 Gi1/0/1, Gi1/0/2 | DIST-SW01 Gi0/1, DIST-SW02 Gi0/1 | 802.1Q trunks |
| CORE-SW02 Gi1/0/1, Gi1/0/2 | DIST-SW01 Gi0/2, DIST-SW02 Gi0/2 | 802.1Q trunks |
| CORE-SW01 Gi1/0/3-4 | CORE-SW02 Gi1/0/3-4 | LACP Po1 (trunk) |
| CORE-SW01 Gi1/0/6 | CORE-SW02 Gi1/0/6 | Routed, 10.10.253.0/30 |
| CORE-SW01 Gi1/0/5 | DC-CORE-SW01 Gi0/1 | Routed, 10.10.254.0/30 |
| CORE-SW01 Gi1/0/7 | HQ-FW-01 | Routed, 10.10.255.0/30 |
| DIST-SW01 Fa0/23-24 | DIST-SW02 Fa0/23-24 | LACP Po2 (trunk) |
| DIST-SW01 Fa0/1-10, DIST-SW02 Fa0/1-10 | Each access switch Fa0/24 and Fa0/23 | 802.1Q trunks |

---

## Addressing and segmentation

### HQ VLANs

The gateway for each VLAN is the HSRP virtual IP (.1). CORE-SW01 owns .2 and CORE-SW02 owns .3. HQ endpoints are statically addressed; no DHCP relay is configured on the cores.

| VLAN | Name | Subnet |
|---|---|---|
| 10 | IT | 10.10.10.0/24 |
| 20 | HR | 10.10.20.0/24 |
| 30 | Finance | 10.10.30.0/24 |
| 40 | Sales | 10.10.40.0/24 |
| 50 | Marketing | 10.10.50.0/24 |
| 60 | Management | 10.10.60.0/24 |
| 70 | Voice | 10.10.70.0/24 |
| 80 | Guest | 10.10.80.0/24 |
| 90 | Printers/IoT | 10.10.90.0/24 |
| 99 | MGMT | 10.10.99.0/24 |
| 999 | PARKING | No subnet. Holds shut-down unused ports |

### Routed links and server segments

| Link or segment | Subnet |
|---|---|
| CORE-SW01 to CORE-SW02 | 10.10.253.0/30 (.1 and .2) |
| CORE-SW01 to DC-CORE-SW01 | 10.10.254.0/30 (.1 and .2) |
| CORE-SW01 to HQ-FW-01 | 10.10.255.0/30 (CORE-SW01 .2, HQ-FW-01 .1) |
| DMZ | 172.16.10.0/24 (WEB01 .10, MAILGW01 .14) |
| Data Center | 172.16.20.0/24 |
| Security and Monitoring | 172.16.40.0/24 (SIEM01 .10, MON01 .12, SYSLOG01 .13) |
| HQ to ISP1 / ISP2 | 203.0.113.0/30 and 203.0.113.4/30 |

### Branches

| Site | LAN | Router WAN | Tunnel interfaces |
|---|---|---|---|
| Alexandria | 172.20.0.0/16 | 198.51.100.2/30 | Tunnel1 10.255.0.1 (to Cairo), Tunnel2 10.255.0.5 (to Giza) |
| Cairo | 172.21.0.0/16 | 198.51.100.6/30 | Tunnel1 10.255.0.2, Tunnel3 10.255.0.9 |
| Giza | 172.22.0.0/16 | 198.51.100.9/30 | Tunnel2 10.255.0.6, Tunnel3 10.255.0.10 |

Tunnel subnets: Alexandria-Cairo 10.255.0.0/30, Alexandria-Giza 10.255.0.4/30, Cairo-Giza 10.255.0.8/30.

---

## Routing and switching design

- **Gateway redundancy.** HSRP on all ten VLANs. CORE-SW01 is Active (priority 150), CORE-SW02 is Standby (priority 100), preempt enabled on both.
- **Spanning tree.** Rapid-PVST on the core and distribution layers. CORE-SW01 is root for the ten data VLANs (priority 4096) and CORE-SW02 is secondary (priority 8192). The distribution switches run root guard on their access-facing ports. Four access switches run Rapid-PVST and six still run PVST (finding F5).
- **Dual-homed access layer.** Each access switch has one uplink to each distribution switch. Spanning tree forwards on the uplink to DIST-SW01 and blocks the uplink to DIST-SW02. The access trunks carry only the VLANs each switch needs (for example, ACC-SW-SALES carries VLAN 40 only); ACC-SW-MGMT is the exception and allows 10-99.
- **Link aggregation.** LACP Po1 between the cores and Po2 between the distribution switches.
- **Routing.** OSPF area 0 between CORE-SW01, CORE-SW02 and DC-CORE-SW01, with `passive-interface default` so adjacencies form only on the routed links. CORE-SW01 has static routes to the data center and security segments and a default route to HQ-FW-01. CORE-SW02 has no direct link to the data center or the firewall: it learns those segments through CORE-SW01 and its default route points to CORE-SW01 (10.10.253.1).
- **Perimeter NAT (ASA Auto NAT).** Dynamic PAT for the HQ LAN to the OUTSIDE-ISP1 interface, dynamic PAT for the DMZ network, and static NAT for WEB01 (198.51.100.18) and MAILGW01 (198.51.100.19).
- **Branches.** Each branch router performs PAT for its LAN (one NAT rule, `list 101`) and runs GRE tunnels to the other two sites.
- **Scope.** HQ and the branches are separate networks on purpose. Branches reach each other and the Internet, not the HQ LAN.

---

## Security controls

| Control | Where | Status |
|---|---|---|
| `HR-IN` ACL: HR may reach IT and the Data Center; blocked from other internal VLANs, the DMZ and the Security segment | Inbound on Vlan20, both cores | **Verified on CORE-SW01.** Allowed and blocked pings, with every counter matching the pings sent |
| `GUEST-IN` ACL: Guest may reach the Internet and one internal DNS host only | Inbound on Vlan80, both cores | **Verified on CORE-SW01** for the blocks and Internet access; the DNS exception was not tested |
| Same ACLs on the standby core | CORE-SW02 | Configuration identical; enforcement after failover **inconclusive** |
| HQ dynamic PAT | HQ-FW-01 | **Verified** with `show xlate` |
| Static NAT for DMZ servers, DMZ dynamic PAT | HQ-FW-01 | Configured, 0 hits; **not tested** |
| Unused ports in VLAN 999 and shut down | All 14 HQ switches | **Verified.** Uplink-module ports (Gi1/1/1-4 on the cores, Gi0/1-2 on the access switches) remain in VLAN 1 |
| SSH v2 only on vty 0-15, local user stored as a secret, password encryption, banner, exec-timeout | HQ switches | **Verified** on 13 of 14 switches. ACC-SW-SALES runs SSH 1.5 (finding F4). Only the cores have a management IP, so SSH is reachable on the cores only |
| PortFast and BPDU Guard on access ports | Access switches | PortFast **verified**; BPDU Guard configured, violation not tested |
| Root guard on distribution downlinks | DIST-SW01, DIST-SW02 | Configured, not tested |
| Port security (maximum 2 MACs, restrict) | ACC-SW-IT, -HR, -FINANCE, -VOICE, -MGMT | **Verified** (MAC learned, 0 violations). Missing on five switches (finding F14) |
| NTP from MON01 | Both cores, HQ-FW-01 | **Verified** (stratum 2, reference 172.16.40.12) |
| Syslog to SYSLOG01 and SIEM01 | Both cores | **Verified** on SYSLOG01; SIEM01 not checked |
| SNMP read-only community | Cores | Configured; cannot be verified or restricted in the simulator |
| HSRP authentication, VTY `access-class` | Cores | **Not configured** (findings F11, F12) |
| IPsec protection of branch GRE tunnels | Branch routers | **Attempted and rolled back**; tunnels are plain GRE (accepted risk) |

The configuration-based vulnerability assessment, with 14 findings, their severity, remediation and status, is in [`VA_REPORT.md`](VA_REPORT.md).

---

## Testing and validation

The full matrix, with commands, expected and actual results and the evidence source for each test, is in [`VALIDATION_TESTS.md`](VALIDATION_TESTS.md).

| Area | Tests | PASS | FAIL | INCONCLUSIVE | NOT TESTED | UNSUPPORTED |
|---|---|---|---|---|---|---|
| A. Switching and routing | 11 | 8 | 1 | 1 | 1 | 0 |
| B. ACL policy | 14 | 13 | 0 | 0 | 1 | 0 |
| C. NAT and Internet access | 7 | 4 | 0 | 0 | 3 | 0 |
| D. Branch tunnels | 9 | 7 | 1 | 0 | 1 | 0 |
| E. Monitoring | 5 | 2 | 0 | 0 | 1 | 2 |
| H. Hardening | 10 | 8 | 1 | 0 | 1 | 0 |
| **Total** | **56** | **42** | **3** | **1** | **8** | **2** |

The three failures are documented: six access switches still on PVST, the IPsec attempt, and SSH 1.5 on ACC-SW-SALES.

### How ACL enforcement was proven

The running configuration contains `ip access-group HR-IN in` and `ip access-group GUEST-IN in`, but `show ip interface Vlan20` and `Vlan80` report `Inbound access list is not set` on both cores. Because the two views disagree, enforcement was proven with traffic. From an HR PC:

| Ping | Result | ACL line that matched |
|---|---|---|
| 10.10.10.1 (IT) ×4 | 4/4 replies | permit to 10.10.10.0/24: 4 matches |
| 172.16.20.10 (Data Center) ×4 | 3/4 replies (first lost to ARP) | permit to 172.16.20.0/24: 4 matches |
| 10.10.30.1 (Finance gateway) ×4 | Unreachable from 10.10.20.2 | deny to 10.10.0.0/16: 8 matches in total |
| 10.10.30.11 (Finance PC) ×4 | Unreachable from 10.10.20.2 | (same line) |
| 172.16.10.10 (DMZ) ×4 | Unreachable from 10.10.20.2 | deny to 172.16.0.0/12: 4 matches |

```
CORE-SW01#show ip access-lists HR-IN
Extended IP access list HR-IN
    permit udp any eq bootpc any eq bootps (2 match(es))
    permit ip 10.10.20.0 0.0.0.255 10.10.20.0 0.0.0.255
    permit ip 10.10.20.0 0.0.0.255 10.10.10.0 0.0.0.255 (4 match(es))
    permit ip 10.10.20.0 0.0.0.255 172.16.20.0 0.0.0.255 (4 match(es))
    deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (8 match(es))
    deny ip 10.10.20.0 0.0.0.255 172.16.0.0 0.15.255.255 (4 match(es))
    permit ip 10.10.20.0 0.0.0.255 any (886 match(es))
```

The large count on the final permit is mostly control-plane traffic, such as the peer core's HSRP hellos entering the SVI, not user traffic. The Guest tests worked the same way: Guest to IT and to the Data Center were refused by 10.10.80.2, with 4 matches on each deny line, and Guest reached 8.8.8.8.

### HSRP failover

With Vlan20 shut down on CORE-SW01, CORE-SW02 took over group 20 (`Vl20 20 100 P Active local`). After `no shutdown`, CORE-SW01 preempted back to Active on all ten groups. During the failover, HR traffic to Finance was still blocked, but the unreachable came from CORE-SW01's routed link and CORE-SW02's deny counters did not move. ACL enforcement after failover is therefore recorded as **inconclusive**.

### Sample evidence

HSRP roles on CORE-SW01 (excerpt; all ten groups match):

```
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           10.10.10.3      10.10.10.1
Vl20        20   150 P Active   local           10.10.20.3      10.10.20.1
Vl99        99   150 P Active   local           10.10.99.3      10.10.99.1
```

OSPF adjacencies on CORE-SW01:

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3           1   FULL/DR         00:00:36    10.10.254.2     GigabitEthernet1/0/5
2.2.2.2           1   FULL/DR         00:00:36    10.10.253.2     GigabitEthernet1/0/6
```

ASA translations on HQ-FW-01 after an HQ host pinged 8.8.8.8:

```
ICMP PAT from INSIDE:10.10.10.11/3 to OUTSIDE-ISP1:203.0.113.2/54267 flags i idle 00:00:23, timeout 0:00:30
NAT from DMZ:172.16.10.14/32 to OUTSIDE-ISP1:198.51.100.19/32 flags s idle 00:06:10, timeout 0:00:00
NAT from DMZ:172.16.10.10/32 to OUTSIDE-ISP1:198.51.100.18/32 flags s idle 00:06:10, timeout 0:00:00
```

Branch LAN to branch LAN across the GRE tunnel (Alexandria to Cairo, extended ping sourced from the LAN interface):

```
Sending 5, 100-byte ICMP Echos to 172.21.99.254, timeout is 2 seconds:
Packet sent with a source address of 172.20.99.254
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/2/10 ms
```

NTP on CORE-SW01:

```
Clock is synchronized, stratum 2, reference is 172.16.40.12
```

### Screenshots

| File | Shows |
|---|---|
| [`01-topology-hq.png`](screenshots/01-topology-hq.png) | HQ topology: data center, security segment, perimeter, DMZ, core, distribution, access |
| [`02-topology-branches.png`](screenshots/02-topology-branches.png) | Alexandria, Cairo and Giza branch topologies |
| [`03-hsrp-ospf-cores.png`](screenshots/03-hsrp-ospf-cores.png) | `show standby brief` and `show ip ospf neighbor` on both cores |
| [`04-asa-nat-xlate.png`](screenshots/04-asa-nat-xlate.png) | `show nat` and `show xlate` on HQ-FW-01 |
| [`05-ntp-internet-ssh.png`](screenshots/05-ntp-internet-ssh.png) | NTP status on CORE-SW01, PC0 ping to 8.8.8.8, SSH login from PC0 to CORE-SW01 |
| [`06-acl-hr-tests.png`](screenshots/06-acl-hr-tests.png) | HR PC pings and the `HR-IN` counters on CORE-SW01 |
| [`07-gre-alexandria.png`](screenshots/07-gre-alexandria.png) | Tunnel and LAN-to-LAN pings from BR-RTR-Alex |
| [`08-gre-cairo-giza.png`](screenshots/08-gre-cairo-giza.png) | Cairo to Giza and Giza to Cairo tunnel pings |
| [`09-syslog01-events.png`](screenshots/09-syslog01-events.png) | Syslog messages from both cores received on SYSLOG01 |

---

## Key design decisions

- **Dual cores with HSRP and preempt**, so the user gateway survives the loss of an SVI on CORE-SW01, verified by a failover and preempt test. The redundancy covers the first hop only: CORE-SW01 holds the only links to the data center and the firewall.
- **Policy at the gateway.** Inter-VLAN filtering is applied inbound on the SVIs closest to the source, and the same ACLs exist on both cores so the policy follows the active gateway.
- **HR and Guest treated as the restricted VLANs.** HR is limited to IT, the Data Center and the Internet; Guest is limited to the Internet and one internal DNS host.
- **Layer 2 protection at the edge.** PortFast and BPDU Guard on access ports, root guard on the distribution downlinks, port security on five access switches, and unused ports parked in a dead VLAN.
- **Trunks carry only what they need.** Each access trunk is pruned to its department VLAN (plus voice and management where used).
- **DMZ behind the ASA**, with public servers mapped by static NAT and kept off the internal LAN.
- **Honest rollback of IPsec.** IPsec over GRE was unstable in the simulator (one peer `QM_IDLE`, the other `MM_NO_STATE`, tunnel pings 0/5). I removed the crypto map from the interfaces, kept plain GRE, and recorded unencrypted inter-branch traffic as an accepted risk.
- **Accepted risk on the native VLAN.** Native VLAN 1 remains on all trunks because changing it safely requires a matched change on both ends of every trunk across 14 switches.

---

## Known limitations

Full details and workarounds: [`KNOWN_ISSUES.md`](KNOWN_ISSUES.md).

**Simulator limits**
- `show ip interface` reports no ACL on the SVIs even though the running configuration applies them.
- ACL counters reset when the file is reopened.
- ASA `logging` and `snmp-server` commands are rejected; SNMP cannot be restricted or verified.
- IPsec over GRE did not establish reliably.

**Design limitations**
- CORE-SW01 is the only path from HQ to the data center, the security segment and the firewall. A full CORE-SW01 failure would isolate the HQ VLANs from those segments and the Internet.
- Only the two cores have a management IP. The distribution and access switches are managed from the console.
- No HSRP authentication and no VTY `access-class`.
- The Management department PC sits in VLAN 99 (MGMT) rather than VLAN 60.

**Configured but not verified**
- Static NAT for the DMZ servers from outside, and the DMZ dynamic PAT.
- ACL enforcement while CORE-SW02 is Active.
- Guest DNS to the permitted host, syslog receipt on SIEM01, SNMP polling.
- Root guard, BPDU Guard and port-security violation behavior.
- PAT from endpoints at Cairo and Giza; Cairo LAN to Giza LAN.

**Out of scope**
- No path between the HQ LAN and the branches.
- VPNGW01 and remote-access VPN are conceptual only.
- HQ-FW-02 is a spare. The branch ASAs and switches were not audited.
- No hardening on the ASA or the branch devices; no AAA `new-model`.

---

## Skills demonstrated

- Hierarchical campus design (core, distribution, access) and IP addressing plans
- VLANs, 802.1Q trunking and trunk pruning, Rapid-PVST root placement, root guard
- LACP EtherChannel
- First-hop redundancy with HSRP, including failover and preempt testing
- OSPF single-area routing with passive interfaces, static and default routes
- Extended named ACLs for inter-VLAN segmentation, verified with traffic and hit counters
- Layer 2 edge security: port security, PortFast, BPDU Guard, unused-port parking
- Cisco ASA Auto NAT: dynamic PAT and static NAT for a DMZ
- GRE site-to-site tunnels and branch PAT
- Device hardening: SSH v2, encrypted secrets, login banners, exec-timeout
- NTP and centralized syslog
- Configuration auditing across 14 switches, with severity and risk acceptance
- Structured troubleshooting and documentation of failures and limitations

---

## How to explore the project

1. Install Cisco Packet Tracer 8.x (available through Cisco Networking Academy).
2. Open `Enterprise_Network_Final.pkt`.
3. Suggested checks:
   - CORE-SW01: `show standby brief`, `show ip ospf neighbor`, `show ip access-lists`, `show ntp status`
   - DIST-SW01: `show etherchannel summary`, `show spanning-tree vlan 10`
   - Any access switch: `show port-security`, `show interfaces status`
   - HQ-FW-01: `show nat`, `show xlate`
   - BR-RTR-Alex: `show ip interface brief`, `ping 10.255.0.2`
   - From an HR PC: `ping 10.10.10.1` (allowed) and `ping 10.10.30.1` (blocked), then `show ip access-lists HR-IN` on CORE-SW01
   - SYSLOG01: Services > SYSLOG

The lab file uses lab-only credentials that are not used anywhere else.

---

## Repository structure

```
.
├── README.md                     project overview (this file)
├── VALIDATION_TESTS.md           56 tests with command, expected and actual result, status and evidence source
├── KNOWN_ISSUES.md               simulator limits, unverified items, design limitations, accepted risks
├── VA_REPORT.md                  configuration-based vulnerability assessment (14 findings)
├── Enterprise_Network_Final.pkt  Packet Tracer project file
├── configs/                      sanitized per-device configuration
├── screenshots/                  evidence captures listed above
└── LICENSE
```

---

## Future improvements

- Give CORE-SW02 its own upstream links (to DC-CORE-SW01 and the firewall), so the core pair is redundant end to end.
- Add HSRP authentication and a VTY `access-class` limited to the MGMT and IT subnets.
- Give every switch a management SVI in VLAN 99 with a default gateway, so SSH works across the layer.
- Finish the open items: Rapid-PVST on the remaining access switches, SSH v2 on ACC-SW-SALES, port security on the remaining five switches, the Management department in VLAN 60.
- Prove ACL enforcement while CORE-SW02 is Active, and test static NAT from outside.
- Retry IPsec protection of the GRE tunnels, or rebuild the branch links in a tool with fuller IPsec support (for example GNS3 or EVE-NG).
- Run a scanner-based assessment against an intentionally vulnerable target in a separate lab.

---

## Version history

- **v1.0:** base build (core, distribution, access, ASA, DMZ, data center, security segment, three branches, GRE, ACLs, NAT).
- **v1.1:** NTP, syslog and SNMP; first vulnerability assessment; unused-port parking, SSH-only management and encrypted secrets; HR ACL tightened to block the DMZ and Security segments.
- **v1.2:** read-only audit of all 14 HQ switches; fixes for a re-enabled parked port and inconsistent VTY lines; traffic tests with matching ACL counters, a failover test, and five new findings; 56-test matrix.

---

## Author

**Ahmed Nabil Toba**, cybersecurity graduate focused on network security and infrastructure.

- GitHub: [github.com/ahmednabiltoba](https://github.com/ahmednabiltoba)
- LinkedIn: [linkedin.com/in/ahmed-nabil-502aa7252](https://www.linkedin.com/in/ahmed-nabil-502aa7252)
- Email: `<your email>`

Built as a hands-on portfolio project. Feedback and questions are welcome through GitHub issues or LinkedIn.
