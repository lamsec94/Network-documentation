# Network Documentation

Network architecture documentation for a segmented, enterprise-style environment built around
OPNsense, VLAN-aware switching, centralized DNS, and remote access through Tailscale.

This repository documents the network design decisions, policy boundaries, and operational
constraints that support the broader Windows, Linux, virtualization, and security stack.

> **Built by:** Lamar Scott | **GitHub:** [lamsec94](https://github.com/lamsec94) | **Last updated:** August 2026

---

## Architecture Overview

![Network Architecture](images/HOMELAB-NETWORK-ARCHITECTURE.png)

The network is designed around a single internal routing and security boundary, with OPNsense
acting as the primary firewall, inter-VLAN router, and NAT point for all internal segments.
Rather than treating the environment as a flat network, the design separates management, server,
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
| 20 | GUEST | Internet-only guest wireless segment isolated from infrastructure |
| 30 | IOT | Restricted segment for IoT devices with limited east-west access |
| 100 | TRANSIT | Upstream handoff network between ER7206 and OPNsense |

The segmentation model is intentionally simple enough to operate at this scale while
still mapping to real-world enterprise concepts such as management plane isolation,
user/application segmentation, and upstream transit design.

MGMT remains untagged on the OPNsense trunk interface. VLAN 10, 20, 30, and 100 operate as
tagged sub-interfaces. The managed switch handles trunk and access port assignment; the
wireless AP runs in AP-only mode with routing and DHCP disabled so OPNsense retains addressing
authority for the wireless segments.

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

The upstream router maintains a static route for the internal supernet back toward OPNsense so
return traffic reaches the environment correctly.

A key implementation detail is that management traffic on the OPNsense trunk must remain untagged.
Tagging VLAN 1 on the OPNsense interface breaks LAN connectivity, so MGMT remains native while
LAB, GUEST, IOT, and TRANSIT operate as tagged sub-interfaces.

---

## DNS Design

DNS is centralized through AdGuard Home, which serves as the primary resolver for the environment
and handles upstream recursion and filtering. The internal domain is conditionally forwarded to
Active Directory DNS, preserving AD-integrated name resolution for Windows systems.

Both domain controllers run AD-integrated DNS and forward externally to AdGuard directly rather
than chaining through one another.

| Resolver role | Handled by |
|---|---|
| Internal AD domain, authoritative | Domain controllers |
| Service hostname rewrites | AdGuard Home |
| Upstream recursion and filtering | AdGuard Home |
| External resolution from domain controllers | Forwarded to AdGuard |

### Domain controllers forward directly rather than to each other

An earlier configuration had one domain controller forwarding to the other. It worked, but it
created a dependency — the first DC could not resolve external names if the second was down.
Pointing both directly at the upstream resolver removes that coupling.

### A DNS server with no forwarders fails silently

One domain controller was found running with no forwarders configured at all, falling back to
root hints that were not resolving.

Nothing appeared broken. Internal name resolution worked normally because the DC is authoritative
for its own zone and answers those queries directly. Only external resolution failed, and it
surfaced somewhere unrelated — a product activation attempt on a different host that happened to
be using that DC as its resolver.

```powershell
Get-DnsServerForwarder
Add-DnsServerForwarder -IPAddress <resolver>
```

Worth checking on every DNS server after any rebuild. The failure mode produces no error on the
DNS server itself and misattributes to whatever unrelated operation happens to need the internet
first.

### Authoritative zones do not forward names within themselves

Service hostnames under the internal domain are handled by resolver-level rewrites pointing at
the reverse proxy. Domain-joined machines do not resolve them.

The reason is structural rather than a misconfiguration: domain members point at the domain
controllers, and the DCs are authoritative for that zone. They answer from their own zone data
rather than forwarding, and the zone contains no records for those service hostnames. Forwarders
are only consulted for names the server is not authoritative for.

The fix is an A record in the AD-integrated zone for each service name that needs to resolve from
domain members. That creates a two-place maintenance burden — service names then live in both the
resolver rewrites and AD DNS, and both need updating when a service is added or retired.

### `.local` and systemd-resolved

`.local` is reserved for mDNS by RFC 6762. systemd-resolved intercepts that domain and refuses to
send those queries to a unicast DNS server unless a routing domain explicitly claims it. `dig`
bypasses resolved entirely, which is why it succeeds while every normal lookup on the same host fails.

Netplan's `search:` directive does not produce a routing domain. The durable fix is a resolved drop-in:

```ini
[Resolve]
DNS=<resolver>
Domains=~<internal-domain>
MulticastDNS=no
```

The `~` prefix is what marks it as a routing domain. Choosing `.local` as an internal domain name
is the root cause; it is a known trap and every systemd-resolved host will fight it.

### A retired resolver fails silently when a secondary exists

A fleet audit found five of eight hosts still pointing at a resolver that had been decommissioned
during a service consolidation. Every one of them listed a public resolver as secondary, so
external names resolved normally and nothing appeared broken. Internal name resolution had simply
been failing on those hosts for weeks.

After retiring any resolver, audit every host that referenced it. Absence of complaints is not
evidence that it was unused.

---

## Remote Access

Remote administration is provided through Tailscale with subnet routing for management and lab access.
This allows secure access to internal services and infrastructure without directly publishing them to the public internet.

Subnet routing runs bare-metal on the primary hypervisor host, which maintains an explicit route
for the LAB subnet via OPNsense so that traffic routes directly rather than looping back through
the tunnel. The Raspberry Pi runs a Tailscale client as a jumpbox but deliberately does not
advertise subnet routes — route advertisement belongs to exactly one host.

This keeps exposure low while demonstrating remote access design patterns that map to enterprise
VPN and zero-trust concepts.

### Do not run a VPN client on a domain controller

Two separate domain controllers were found running Tailscale clients that were never part of the
routing design, and both broke domain controller location in Active Directory.

In one case the virtual adapter self-assigned an APIPA address and advertised deprecated
site-local IPv6 DNS servers. In the other it held a CGNAT address from the `100.64.0.0/10` range.
In both cases `Get-ADDomainController` reported the VPN interface address rather than the host's
real LAN address, because the client had registered it.

```powershell
Get-Package -Name "*Tailscale*" | Uninstall-Package -Force
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress
ipconfig /registerdns
Restart-Service netlogon
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address
```

Both instances presented as something else entirely before the adapter was identified as the
cause. If DC address reporting looks wrong, enumerate every interface before assuming a real
network fault — and keep VPN clients off hosts that register their own addresses in a directory.

---

## Firewall Policy

Firewall policy is organized around explicit segmentation rather than permissive lateral access.
Default deny is in effect on all interfaces, and there is no catch-all allow rule on any segment.

| Segment | Policy posture |
|---|---|
| LAB | Trusted services segment. Twelve service-specific rules: individual service access, ICMP to gateway, intra-LAB traffic, ICMP outbound, outbound web on 80/443, and a path to MGMT |
| MGMT | Infrastructure administration. Carries a corresponding rule toward LAB |
| GUEST | Internet-only, isolated from all internal segments |
| IOT | Restricted, limited east-west access |
| TRANSIT | Upstream handoff only |

Writing rules service by service rather than allowing a segment wholesale means the rule list
itself documents what is actually reachable. It also means new services fail closed until a rule
exists, which surfaces intent rather than letting access accumulate silently.

Suricata runs on the LAB interface in detection mode with promiscuous mode enabled. It logs and
alerts; it does not block. That distinction is worth stating plainly — the environment has
intrusion detection, not inline prevention, and only on one segment. WAN, GUEST, IOT, and TRANSIT
traffic is uninspected.

---

## Operational Notes

Several implementation details became important enough to document as permanent design constraints:

- OPNsense MGMT must remain untagged on the trunk interface. Tagging VLAN 1 breaks LAN connectivity.
- Virtual machines on the secondary hypervisor node require VLAN tag 10 for LAB access; tag 11 caused complete connectivity failure during domain controller deployment.
- ISC DHCP must remain disabled when Dnsmasq is the active DHCP service. The two conflict.
- Centralized DNS with conditional forwarding is required to keep AD name resolution stable across segments.
- Every DNS server needs forwarders explicitly verified after a rebuild. Root-hint fallback fails silently.
- VPN clients do not belong on domain controllers. They register addresses that break DC location.
- Renaming a network interface leaves netplan files behind that still declare a default route. Rename to `.disabled` rather than deleting, and use `netplan try` for any remote routing change — it auto-reverts after 120 seconds.

These are the kinds of findings that matter in real infrastructure work, because they turn
trial-and-error into repeatable operational knowledge.

---

## Skills Demonstrated

`OPNsense` `VLAN segmentation` `Inter-VLAN routing` `Layer 2 switching` `Network security zoning`
`DNS architecture` `Conditional forwarding` `Wireless isolation` `Tailscale` `Suricata IDS`
`Infrastructure documentation`
