# IPFire Network Segmentation Lab

A four-zone firewall deployment using **IPFire 2.29 (Core Update 191)** on VMware Workstation. The lab segments a network into WAN, LAN, wireless, and DMZ zones, hosts a web server in the DMZ, and verifies the firewall policy with connectivity tests between every zone.

> **My role:** I carried out the full project end to end: planning and prioritization (MoSCoW), IPFire installation, zone and DHCP configuration, firewall rules, DMZ web server, security services, and all testing and documentation.

## Table of Contents
- [Overview](#overview)
- [Network Design](#network-design)
- [Requirements (MoSCoW)](#requirements-moscow)
- [Implementation](#implementation)
- [Firewall Rules](#firewall-rules)
- [Key Takeaways](#key-takeaways)
- [Future Work](#future-work)
- [References](#references)

## Overview

IPFire is an open-source Linux firewall distribution used in home networks, businesses, and data centers. It separates a network into color-coded zones, each with its own security policy. This project builds a four-interface IPFire firewall and uses it to control traffic between the zones.

| Zone | Role | Subnet / Address |
|------|------|------------------|
| **RED** | WAN / Internet | DHCP from upstream network |
| **GREEN** | Trusted LAN | 192.168.133.0/24 (firewall: .4) |
| **BLUE** | Wireless (WLAN) | 192.168.92.0/24 (firewall: .2) |
| **ORANGE** | DMZ (web server) | 192.168.220.0/24 (firewall: .2) |

## Network Design

```
                        Internet / upstream
                               │
                          RED (WAN)
                               │
   BLUE (WLAN) ───────── [ IPFire ] ───────── GREEN (LAN)
 192.168.92.0/24             │               192.168.133.0/24
                             │
                       ORANGE (DMZ)
                     192.168.220.0/24
                  Apache2 web server (.5)
```

Each zone is bound to its own virtual NIC in the zone configuration (NIC assignment: one native NIC per color).

## Requirements (MoSCoW)

Features were prioritized with the MoSCoW method before implementation.

| Priority | Features |
|----------|----------|
| **Must have** | Firewall rules and packet filtering, IDS/IPS, SSH and web administration interfaces |
| **Should have** | VPN (OpenVPN), web proxy with content filtering, Quality of Service (QoS) |
| **Could have** | Logging and monitoring with email alerts, NAT configuration |
| **Won't have (this phase)** | Load balancing and failover, IPv6 configuration |

## Implementation

### 1. Installation and zone configuration
- Installed IPFire from the ISO in VMware Workstation with four virtual NICs
- Assigned one NIC to each zone (RED, GREEN, BLUE, ORANGE) in **Network → Zone Configuration**
- Verified the interface addresses from the IPFire console with `ip a`

### 2. DHCP
- **GREEN:** pool 192.168.133.0 – 192.168.133.20, primary DNS 192.168.133.4
- **BLUE:** pool 192.168.92.100 – 192.168.92.200, primary DNS 192.168.92.2, secondary DNS 8.8.8.8
- Lease time 2280 minutes, domain suffix `ipfiregroup`
- Fixed (static) leases created for the test VMs so their addresses stay constant

### 3. BLUE access control
IPFire blocks BLUE traffic by default. Devices must be explicitly allowed under **Wireless Configuration**. The test VM on BLUE (192.168.92.128) was added from the DHCP lease list.

### 4. DMZ web server
An Ubuntu VM in the ORANGE zone (192.168.220.5) runs **Apache2**, reachable from the GREEN network through a firewall rule.

### 5. Firewall rules
Custom rules were added under **Firewall → Firewall Rules** (see the next section).

### 6. Security and management services
- **IDS/IPS:** selected the network zones that need traffic analysis, then added and customized the rule set
- **SSH and web administration:** password-based authentication, TCP forwarding enabled, default SSH port 22; web interface used for all configuration
- **OpenVPN:** configured the IP settings and allowed UDP port 1194 for secure remote access
- **Web proxy:** enabled URL filtering, configured blocked domains, and turned on logging to track usage
- **Quality of Service:** assigned traffic priorities by source and destination port

## Firewall Rules

| # | Protocol | Source | Destination | Purpose |
|---|----------|--------|-------------|---------|
| 1 | TCP | Any | RED: port 25 (SMTP) | Block outbound SMTP to the internet |
| 2 | TCP | GREEN | Firewall:80 → 192.168.220.5 (HTTP) | GREEN access to the DMZ web server |
| 3 | TCP | 192.168.92.128 (BLUE) | 192.168.133.3:12345 (GREEN) | TCP access from BLUE to GREEN |
| 4 | ICMP | 192.168.92.128 (BLUE) | 192.168.133.3 (GREEN) | Ping from BLUE to GREEN |
| 5 | ICMP | 192.168.92.128 (BLUE) | 192.168.220.5 (ORANGE) | Ping from BLUE to ORANGE |
| 6 | ICMP | 192.168.220.5 (ORANGE) | 192.168.92.128 (BLUE) | Ping from ORANGE to BLUE |
| 7 | TCP | Any | Firewall (RED):12345 → 192.168.220.5:12345 | Port forwarding from RED to ORANGE |

**Resulting zone policy** (as shown in the IPFire rules summary):

| From ↓ / To → | Internet | GREEN | BLUE | ORANGE |
|---------------|----------|-------|------|--------|
| **GREEN** | Allowed | n/a | Allowed | Allowed |
| **ORANGE** | Allowed | Blocked | Blocked | n/a |
| **BLUE** | Allowed | Blocked | n/a | Blocked |

Blocked paths are opened only by the specific rules above. IPFire's default ruleset also keeps RED closed to all internal zones, so inbound access needs port forwarding or a VPN.

## Testing and Results

Connectivity was verified with `ping` from VMs placed in each zone.

| Test | Expected | Result |
|------|----------|--------|
| GREEN → RED | Allowed
| GREEN → BLUE | Allowed 
| BLUE → GREEN | Allowed (rule 4) 
| BLUE → RED | Allowed 
| BLUE → ORANGE | Allowed (rule 5) 
| ORANGE → BLUE | Allowed (rule 6) 
| ORANGE → RED | Allowed 
| RED → BLUE | Blocked 
| RED → GREEN | Blocked 

The blocked tests matter as much as the allowed ones. They confirm the firewall keeps the WAN side from reaching internal networks by default.

## Key Takeaways

- **Default-deny works.** IPFire's zone model blocks inter-zone traffic by default, so every allowed path has to be a deliberate rule.
- **Least privilege.** Rules name specific hosts and protocols (one BLUE host, ICMP only, specific ports) rather than opening whole subnets.
- **DMZ isolation.** The web server sits in ORANGE, which cannot initiate connections to GREEN or BLUE, limiting the damage if it were compromised.
- **Test the negatives.** Showing that RED cannot reach BLUE or GREEN is as important as showing allowed traffic works.

## Future Work

Features deprioritized in the MoSCoW analysis, plus possible improvements:

- Load balancing and failover (multiple WAN links)
- IPv6 configuration
- Logging and monitoring with email alerts
- Restrict the broad rules further and add a dedicated management network
- Add tests that exercise the IDS/IPS, proxy filtering, and VPN end to end

## Tech Stack

IPFire 2.29 · VMware Workstation · Ubuntu · Apache2 · OpenVPN · IDS/IPS · Web proxy · QoS · DHCP · ICMP/TCP testing

## References

- IPFire Development Team. *What is IPFire?* https://www.ipfire.org/docs/what-is-ipfire
- IPFire Development Team. *Creating Firewall Rules.* https://www.ipfire.org/docs/configuration/firewall/rules
- IPFire Development Team. *Intrusion Prevention System (IPS).* https://www.ipfire.org/docs/configuration/firewall/ips
- IPFire Development Team. *SSH Access.* https://www.ipfire.org/docs/configuration/system/ssh
- IPFire Development Team. *OpenVPN on IPFire.* https://www.ipfire.org/docs/configuration/services/openvpn
- IPFire Development Team. *Web Proxy.* https://www.ipfire.org/docs/configuration/network/proxy
- IPFire Development Team. *Quality of Service.* https://www.ipfire.org/docs/configuration/services/qos
- IPFire Development Team. *Logs.* https://www.ipfire.org/docs/configuration/logs
- Zenarmor. *What is IPFire?* https://www.zenarmor.com/docs/network-security-tutorials/what-is-ipfire
