# Network Documentation

Network architecture documentation for a segmented, enterprise-style homelab built around
OPNsense, VLAN-aware switching, centralized DNS, and remote access through Tailscale.
This repository documents the network design decisions, policy boundaries, and operational
constraints that support the broader Windows, Linux, virtualization, and security stack.

> **Built by:** Lamar Scott | **GitHub:** [lamsec94](https://github.com/lamsec94) | **Last updated:** May 2026

---

## Architecture Overview

The network is designed around a single internal routing and security boundary, with OPNsense
acting as the primary firewall, inter-VLAN router, and NAT point for all internal segments.
Rather than treating the lab as a flat network, the design separates management, server,
guest, and IoT traffic into distinct security zones with explicit policy control.

```text
Internet
   ↓
TP-Link ER7206
   ↓
Transit network
   ↓
OPNsense
   ↓
Internal VLANs
   ├── MGMT
   ├── LAB
   ├── GUEST
   └── IOT
```

This keeps routing logic and firewall enforcement centralized while preserving flexibility for
future segmentation, rule tuning, and service isolation.

---

## Design Goals

- Separate infrastructure management from application and endpoint traffic.
- Enforce security boundaries between trusted, semi-trusted, and untrusted network zones.
- Use OPNsense as the single policy enforcement point for routing, firewalling, DHCP, and IDS.
- Keep DNS resolution centralized while allowing Active Directory to remain authoritative for the internal domain.
- Support secure remote administration without exposing internal services directly to the internet.

---

## VLAN Model

| VLAN | Name | Purpose |
|---|---|---|
| 1 | MGMT | Management interfaces for Proxmox, switching, Pi services, and infrastructure administration |
| 10 | LAB | Primary server and admin network for Windows, Linux, containers, and internal services |
| 20 | GUEST | Internet-only guest wireless segment isolated from lab infrastructure |
| 30 | IOT | Restricted segment for IoT devices with limited east-west access |
| 100 | TRANSIT | Upstream handoff network between ER7206 and OPNsense |

The segmentation model is intentionally simple enough to operate in a home environment while
still mapping well to real-world enterprise concepts such as management plane isolation,
user/application segmentation, and upstream transit design.

Detailed VLAN and switch notes are in [vlan-design.md](vlan-design.md).

---

## Core Components

| Component | Role |
|---|---|
| OPNsense | Primary firewall, inter-VLAN routing, DHCP, Suricata IDS on LAB |
| Netgear GS308EP | VLAN-aware Layer 2 switching and trunk/access port assignment |
| TP-Link ER7206 | Upstream edge router providing transit toward the internet |
| TP-Link AX1800 | Wireless AP in AP-only mode for GUEST and IOT SSIDs |
| Raspberry Pi 5 | Proxmox cluster QDevice (quorum tiebreaker), Tailscale jumpbox, network diagnostics |

Each component has a defined role, which helps avoid overlapping services and unclear control boundaries.
That makes troubleshooting cleaner and better reflects how production networks are typically structured.

---

## Routing Strategy

The network uses a single-NAT design, with OPNsense acting as the only internal NAT boundary.
The upstream router is reduced to a transit role rather than serving as a second policy layer.

This keeps packet flow easier to reason about and avoids the confusion that comes from stacked
consumer routing behavior. It also makes firewall troubleshooting, DNS behavior, and remote access
far more predictable.

A key implementation detail is that management traffic on the OPNsense trunk must remain untagged.
Tagging VLAN 1 on the OPNsense interface breaks LAN connectivity, so MGMT remains native while
LAB, GUEST, IOT, and TRANSIT operate as tagged sub-interfaces.

---

## DNS Design

DNS is centralized through AdGuard Home, which serves as the primary resolver for the environment.
For internal domain services, `homelab.local` is conditionally forwarded to Active Directory DNS.

This preserves AD-integrated name resolution for Windows systems while still allowing AdGuard to
handle upstream recursion, filtering, and centralized client resolver behavior. The result is a
clean split between internal authority and general-purpose resolution.

---

## Remote Access

Remote administration is provided through Tailscale with subnet routing for management and lab access.
This allows secure access to internal services and infrastructure without directly publishing them to the public internet.

This approach is a good fit for a homelab because it keeps exposure low while still demonstrating
modern remote access design patterns that map well to enterprise VPN and zero-trust concepts.

---

## Firewall Policy

Firewall policy is organized around explicit segmentation rather than permissive lateral access.
GUEST and IOT are intentionally isolated, while LAB is the trusted services segment and MGMT is reserved
for infrastructure administration.

Detailed rule references and policy notes are documented in [firewall-rules.md](firewall-rules.md).

---

## Operational Notes

Several implementation details became important enough to document as permanent design constraints:

- OPNsense MGMT must remain untagged on the trunk interface.
- `su2` virtual machines require VLAN tag 10 for LAB access; VLAN tag 11 caused complete connectivity failure during LAB-DC2 deployment.
- ISC DHCP must remain disabled when Dnsmasq is used as the active DHCP service.
- Centralized DNS and conditional forwarding are required to keep AD name resolution stable across all segments.

These are exactly the kinds of findings that matter in real infrastructure work because they turn
trial-and-error into repeatable operational knowledge.

---

## Repository Contents

| File | Purpose |
|---|---|
| [vlan-design.md](vlan-design.md) | VLAN layout, segmentation details, and switch assignment notes |
| [firewall-rules.md](firewall-rules.md) | OPNsense firewall policy and access control documentation |

---

## Skills Demonstrated

`OPNsense` `VLAN segmentation` `Inter-VLAN routing` `Layer 2 switching` `Network security zoning`
`DNS architecture` `Conditional forwarding` `Wireless isolation` `Tailscale` `Infrastructure documentation`
