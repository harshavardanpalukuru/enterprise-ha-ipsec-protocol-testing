# Topology, Cabling & IP Addressing Plan
## 1. Subnet & Addressing Table
| Node | Role | Interface | IPv4 Address / Mask | Protocol / Notes |
|---|---|---|---|---|
| **HQ-R1** | HQ Primary Gateway | `Gi0/0` (LAN) | `10.10.10.2/24` | HSRP Group 10 VIP: `10.10.10.1` (Priority 110, Preempt) |
| **HQ-R1** | HQ Primary Gateway | `Gi0/1` (WAN) | `10.255.0.1/30` | Transit to `ISP-R1` (`Gi0/0`) |
| **HQ-R1** | HQ Primary Gateway | `Tunnel0` | `10.255.0.9/30` | GRE over IPSec (`IKEv2/AES-256`), OSPF Area 0, `tcp adjust-mss 1360` |
| **HQ-R2** | HQ Standby Gateway | `Gi0/0` (LAN) | `10.10.10.3/24` | HSRP Group 10 VIP: `10.10.10.1` (Priority 100, Standby) |
| **HQ-SW1** | HQ Primary Dist/Access | `VLAN 10` (SVI) | `10.10.10.4/24` | RSTP Root Primary (`Priority 24576`), LACP `Po1`, DHCP Snooping, DAI |
| **HQ-SW2** | HQ Secondary Switch | `VLAN 10` (SVI) | `10.10.10.5/24` | RSTP Root Secondary (`Priority 28672`), LACP `Po1` |
| **ISP-R1** | Simulated WAN Transit | `Gi0/0` / `Gi0/1` | `10.255.0.2/30` / `10.255.0.5/30` | Simulates ISP transport between HQ and Branch |
| **BR-R1** | Branch Edge Router | `Gi0/1` (WAN) | `10.255.0.6/30` | Transit to `ISP-R1` (`Gi0/1`) |
| **BR-R1** | Branch Edge Router | `Tunnel0` | `10.255.0.10/30` | GRE over IPSec (`IKEv2/AES-256`), OSPF Area 0, `tcp adjust-mss 1360` |
| **BR-R1** | Branch Edge Router | `Gi0/0` (LAN) | `10.20.10.1/24` | Branch LAN Gateway + OSPF Area 0 |
| **HQ-Host** | RHEL 9 / Linux Endpoint | `eth0` / `ens3` | `10.10.10.50/24` | Default GW: `10.10.10.1` (HSRP VIP) |
| **BR-Host** | Linux / VPCS Endpoint | `eth0` | `10.20.10.50/24` | Default GW: `10.20.10.1` |
## 2. Exact GNS3 Cabling Matrix
| Source Device | Source Port | Destination Device | Destination Port | Link Purpose |
|---|---|---|---|---|
| `HQ-SW1` | `Gi0/0` | `HQ-SW2` | `Gi0/0` | LACP EtherChannel (`Port-channel1`) Link 1 |
| `HQ-SW1` | `Gi0/1` | `HQ-SW2` | `Gi0/1` | LACP EtherChannel (`Port-channel1`) Link 2 |
| `HQ-SW1` | `Gi0/2` | `HQ-R1` | `Gi0/0` | HQ Active Router LAN Uplink (`VLAN 10`) |
| `HQ-SW2` | `Gi0/2` | `HQ-R2` | `Gi0/0` | HQ Standby Router LAN Uplink (`VLAN 10`) |
| `HQ-SW1` | `Gi0/3` | `HQ-Host` (RHEL 9) | `eth0` / `ens3` | Access Port (`VLAN 10`, PortFast, BPDU Guard) |
| `HQ-R1` | `Gi0/1` | `ISP-R1` | `Gi0/0` | HQ-to-ISP Point-to-Point WAN (`10.255.0.0/30`) |
| `ISP-R1` | `Gi0/1` | `BR-R1` | `Gi0/1` | ISP-to-Branch Point-to-Point WAN (`10.255.0.4/30`) |
| `BR-R1` | `Gi0/0` | `BR-Host` | `eth0` | Branch LAN (`10.20.10.0/24`) |