# Enterprise High-Availability Campus, IPSec over GRE & Protocol Testing Lab

A multi-router, multi-switch enterprise networking lab built in **GNS3 (`Cisco IOSv` & `IOSvL2`)** demonstrating high-availability campus switching and first-hop redundancy (`LACP`, `Rapid-PVST+`, `HSRP v2` with interface tracking), encrypted WAN tunneling (`IPSec over GRE` with `AES-256` & `SHA-256`), dynamic routing (`OSPF Area 0`), Layer 2 access security (`DHCP Snooping`, `Dynamic ARP Inspection`, `PortFast/BPDU Guard`), and TAC-style protocol fault-injection troubleshooting (`OSPF ExStart MTU mismatch` and `PMTUD DF-bit fragmentation`).

> **Project type:** Enterprise Network Redundancy, WAN Encryption & Protocol QA Lab · **Environment:** GNS3 (`Cisco IOSv` / `IOSvL2` / `VPCS`)

## At a Glance

- **Campus High Availability (`HSRP` + `LACP` + `RSTP`):** Configured `HSRP v2` (Group 10 VIP `10.10.10.1`) across `HQ-R1` (`Priority 110`, Preempt, WAN interface tracking) and `HQ-R2` (`Priority 100`, Standby), paired with an `802.3ad LACP` dual-link trunk (`Port-channel1`) and deterministic `Rapid-PVST+` Root Bridge election (`Priority 24576` on `HQ-SW1`, `28672` on `HQ-SW2`).
- **Encrypted Site-to-Site WAN (`IPSec over GRE` + `OSPF`):** Built a point-to-point `GRE` tunnel (`Tunnel0`, `10.255.0.8/30`) between `HQ-R1` and `BR-R1` across a simulated transit provider (`ISP-R1`), secured with `ISAKMP` + `IPSec ESP` (`esp-256-aes`, `esp-sha256-hmac`, Transport Mode) to carry dynamic `OSPF Area 0` routing and inter-site data traffic.
- **Layer 2 Security Hardening:** Enforced `DHCP Snooping` (Option 82 check, trusted uplinks), `Dynamic ARP Inspection (DAI)` with ARP ACL filtering (`STATIC_HOST_ARP`) and rate limiting (`15 pps`), plus `PortFast` and `BPDU Guard` on access ports.
- **TAC Root-Cause Analysis & Protocol QA:** Replicated and resolved an `OSPF ExStart` adjacency hang caused by tunnel MTU mismatch (`1460` vs `1400` against the `1434`-byte IPSec/GRE transport ceiling) and verified `ICMP Type 3 Code 4 (Fragmentation Needed)` Path MTU Discovery (`PMTUD`) and `TCP MSS clamping (1360 bytes)`.

## Topology & IP Addressing

![GNS3 Topology](screenshots/01-gns3-topology.png)

| Device | Role | Interface | IPv4 Address / Mask | Protocol / Configuration Highlights |
|---|---|---|---|---|
| **HQ-R1** | HQ Active Gateway | `Gi0/0` / `Gi0/1` / `Tu0` | `10.10.10.2/24` / `10.255.0.1/30` / `10.255.0.9/30` | `HSRP` Active (`VIP 10.10.10.1`, Pri `110`), `IPSec/GRE`, `OSPF RID 1.1.1.1`, `SNMPv3` |
| **HQ-R2** | HQ Standby Gateway | `Gi0/0` | `10.10.10.3/24` | `HSRP` Standby (`VIP 10.10.10.1`, Pri `100`, Preempt) |
| **HQ-SW1** | HQ Primary Switch | `VLAN 10` / `Po1` | `10.10.10.4/24` | `RSTP` Root Primary (`24586`), `LACP Po1(SU)`, `DHCP Snooping`, `DAI` |
| **HQ-SW2** | HQ Secondary Switch | `VLAN 10` / `Po1` | `10.10.10.5/24` | `RSTP` Root Secondary (`28682`), `LACP Po1(SU)` |
| **ISP-R1** | WAN Transit Router | `Gi0/0` / `Gi0/1` | `10.255.0.2/30` / `10.255.0.5/30` | Simulated ISP underlay between HQ and Branch |
| **BR-R1** | Branch Edge Router | `Gi0/0` / `Gi0/1` / `Tu0` | `10.20.10.1/24` / `10.255.0.6/30` / `10.255.0.10/30` | `IPSec/GRE`, `OSPF RID 2.2.2.2`, `ip mtu 1400`, `ip tcp adjust-mss 1360` |
| **HQ-Host** | HQ LAN Endpoint | `eth0` | `10.10.10.50/24` | Default Gateway: `10.10.10.1` (`HSRP` VIP) |
| **BR-Host** | Branch LAN Endpoint | `eth0` | `10.20.10.50/24` | Default Gateway: `10.20.10.1` (`BR-R1`) |

Full cabling and port mappings are documented in [`topology/ip-addressing-and-cabling.md`](topology/ip-addressing-and-cabling.md).

## Verification Summary

| Feature Under Test | Verification Command | Recorded Result |
|---|---|---|
| **LACP Link Aggregation** | `show etherchannel summary` (`HQ-SW1`) | `Po1(SU)` active using `LACP` with `Gi0/0(P)` and `Gi0/1(P)` bundled (`Cost 3`) |
| **Rapid-PVST+ Root Election** | `show spanning-tree vlan 10` (`HQ-SW1`) | `This bridge is the root` (`Priority 24586`, MAC `0c6f.f6bd.0000`), edge ports `P2p Edge FWD` |
| **L2 Security (Snooping & DAI)** | `show ip dhcp snooping` / `show ip arp inspection vlan 10` | Operational on `VLAN 10`; `STATIC_HOST_ARP` ACL active; access port rate-limited to `15 pps` |
| **HSRP Failover & Preemption** | `show standby brief` (`HQ-R1` & `HQ-R2`) | `HQ-R2` transitioned `Standby -> Active` on `HQ-R1 Gi0/0` down; `HQ-R1` preempted back in `74 ms` |
| **OSPF over IPSec/GRE** | `show ip ospf neighbor` / `show ip route ospf` | Neighbor `2.2.2.2` `FULL/ -` on `Tunnel0`; route `O 10.20.10.0/24 [110/1001] via 10.255.0.10` |
| **IPSec Encryption Counters** | `show crypto ipsec sa` (`HQ-R1`) | `ACTIVE(ACTIVE)` with `SPI 0x68343D32` / `0xBFB67F41` (`esp-256-aes esp-sha256-hmac`, `161 pkts encrypt`, `0 errors`) |
| **End-to-End Tunnel Traceroute** | `trace 10.20.10.50` (`HQ-Host`) | Hop 1: `10.10.10.2` (`HQ-R1`) -> Hop 2: `10.255.0.10` (`BR-R1 Tu0`) -> Hop 3: `10.20.10.50` (`ISP-R1` transparent) |

## TAC Root-Cause Analysis & Fault-Injection Scenarios

### Scenario 1: OSPF Neighbor Stuck in `EXSTART/ -` Due to Tunnel MTU Mismatch

- **Fault Injected:** Raised `Tunnel0` IP MTU on `BR-R1` to `1460` (`ip mtu 1460`) while `HQ-R1` remained at `1400`, then reset OSPF (`clear ip ospf process`).
- **Observed Symptom:** `BR-R1` logged `%TUN-4-MTUCONFIGEXCEEDSTRMTU_IPV4: Tunnel0 IPv4 MTU configured 1460 exceeds tunnel transport MTU 1434` and OSPF neighbor `1.1.1.1` wedged in `EXSTART/ -`.
- **Root Cause:** IPSec ESP transport + GRE encapsulation consumes `66 bytes` of the `1500-byte` physical WAN link (leaving a `1434-byte` transport ceiling). Additionally, OSPF routers compare interface MTUs in Database Description (`DBD`) packets during `ExStart` and reject Master/Slave negotiation when MTUs differ.
- **Resolution:** Standardized `ip mtu 1400` and `ip tcp adjust-mss 1360` on both `HQ-R1` and `BR-R1` `Tunnel0` interfaces, restoring `FULL/ -` adjacency immediately.

### Scenario 2: Path MTU Discovery (`PMTUD`) & `ICMP Type 3 Code 4` Validation

- **Test Execution:** Issued `ping 10.20.10.50 -l 1450 -D` (Don't Fragment bit set) from `HQ-Host` (`10.10.10.50`).
- **Observed Result:** `HQ-R1` (`10.10.10.2`) rejected the oversized frame at the tunnel boundary and returned `ICMP type:3, code:4, Fragmentation required, and DF flag set`. Subsequent `ping 10.20.10.50 -l 1350 -D` (`1378 bytes` on wire) succeeded with `0% loss` (`ttl=62`, `avg ~51.9 ms`).

## Repository Structure

- `README.md` - Lab overview, topology table, verification summary, and TAC RCA cases
- `topology/ip-addressing-and-cabling.md` - Full subnetting and GNS3 port cabling matrix
- `network-configs/` - Complete Cisco IOS running configurations (`HQ-R1`, `HQ-R2`, `HQ-SW1`, `HQ-SW2`, `ISP-R1`, `BR-R1`)
- `verification-logs/` - Recorded CLI verification outputs (`HSRP`, `LACP`, `RSTP`, `IPSec SA`, `OSPF ExStart`, `PMTUD`)
- `screenshots/` - Topology diagram and CLI verification screenshots

## Author

**Palukuru Harsha Vardan**  
Released under the MIT License.