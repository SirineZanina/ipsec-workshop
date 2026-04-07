# IPsec Workshop — Site-to-Site VPN with StrongSwan

> Hands-on lab guide · Ubuntu 24.04 LTS

---

## What You Will Build

A real Site-to-Site IPsec VPN between two Ubuntu 24.04 virtual machines, simulating two geographically separate office gateways (Sousse and Tunis). By the end of the lab, traffic between the two gateways will be fully encrypted using IKEv2 + ESP Tunnel Mode.

```
 ┌─────────────────────┐                        ┌─────────────────────┐
 │   GW-A (Sousse)     │      IPsec ESP         │   GW-B (Tunis)      │
 │  192.168.56.10      │◄══════════════════════►│  192.168.56.20      │
 │                     │        IKEv2           │                     │
 └─────────────────────┘                        └─────────────────────┘
          ↕ Isolated virtual network (ipsec-lab)
```

---

## Lab Environment

| Parameter | Value |
|-----------|-------|
| Platform | Ubuntu 24.04 LTS VMs on Linux host (QEMU/KVM) or Windows host (VirtualBox/ VMware Workstation) |
| VPN Type | Site-to-Site IPsec — IKEv2 + ESP Tunnel Mode |
| Tools | StrongSwan 5.9.x, swanctl, ip-xfrm, tcpdump |
| Level | Intermediate — Linux CLI required |

---

## Workshop Structure

```
ipsec-workshop/
├── README.md                        ← You are here — workshop overview
├── steps/
│   ├── 00-snapshots.md              ← Create VM snapshots before starting
│   ├── 01-network-setup.md          ← Create isolated network + attach VMs
│   ├── 02-static-ip.md              ← Assign static IPs via Netplan
│   ├── 03-install-packages.md       ← Install StrongSwan + required packages
│   ├── 04-configure-gw-a.md         ← swanctl config for GW-A (Sousse)
│   ├── 05-configure-gw-b.md         ← swanctl config for GW-B (Tunis)
│   ├── 06-bring-up-tunnel.md        ← Start StrongSwan, initiate tunnel
│   ├── 07-verify-tunnel.md          ← Verify SAs, capture ESP traffic
│   └── 08-troubleshooting.md        ← Common errors and fixes
└── assets/
    └── (screenshots go here)
```

---

## Prerequisites

- Linux CLI comfortable (`cd`, `ip`, `systemctl`, `cat`, `nano/vim`)
- Basic networking: IP addresses, subnets, routing
- Virtualisation: VM creation, network types (NAT, isolated)
- Theoretical IPsec knowledge (IKE, ESP, SA, modes) — see the presentation PDF

---

## Quick Start

Follow the steps in order:

1. [Step 0 — Create Snapshots](steps/00-snapshots.md)
2. [Step 1 — Network Setup](steps/01-network-setup.md)
3. [Step 2 — Static IP Configuration](steps/02-static-ip.md)
4. [Step 3 — Install Packages](steps/03-install-packages.md)
5. [Step 4 — Configure GW-A](steps/04-configure-gw-a.md)
6. [Step 5 — Configure GW-B](steps/05-configure-gw-b.md)
7. [Step 6 — Bring Up the Tunnel](steps/06-bring-up-tunnel.md)
8. [Step 7 — Verify the Tunnel](steps/07-verify-tunnel.md)
9. [Step 8 — Troubleshooting](steps/08-troubleshooting.md)

---

## Key Concepts Covered

| Concept | Where |
|---------|-------|
| IKEv2 handshake (IKE_SA_INIT + IKE_AUTH) | Step 6 + Step 7 |
| ESP Tunnel Mode encapsulation | Step 7 |
| Pre-Shared Key authentication | Step 4 + Step 5 |
| Security Association Database (SAD) | Step 7 |
| XFRM kernel subsystem | Step 7 |
| Traffic selectors | Step 4 + Step 5 |
| Dead Peer Detection (DPD) | Step 4 + Step 5 |

---

## ✅ Final Success Criteria

The lab is complete when:

- `sudo swanctl --list-sas` shows **ESTABLISHED** IKE SA + **INSTALLED** Child SA
- `sudo ip xfrm state` shows **exactly 2 entries** (one per direction)
- `sudo tcpdump -i <iface> -n 'esp or icmp'` shows **ESP packets only** when pinging the peer — no visible ICMP
- `sudo journalctl -u strongswan` shows no `AUTHENTICATION_FAILED` or `NO_PROPOSAL_CHOSEN` errors

---
