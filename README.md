# secure-rdp-wireguard-vps-lab

A production-ready VPS design for small virtual organizations. This project demonstrates secure RDP access through WireGuard tunneling, allowing multiple clients to access a VPS hosting business operational software/apps.

## Lab Overview

**Host:** Linux Mint with KVM/QEMU/virt-manager

**Infrastructure:**
- 1x Windows Server 2022 Standard (VPS)
- 2x Windows 10 Pro (Clients)

**Networks:**
- WAN: 10.0.0.0/24 (simulated public)
- VPS Private: 192.168.10.0/24 (isolated)
- Client A Private: 192.168.20.0/24 (isolated)
- Client B Private: 192.168.30.0/24 (isolated)
- WireGuard Tunnel: 10.0.8.0/24 (encrypted)

## Architecture

```
┌─ Linux Mint Host ──────────────────────────────────────┐
│ KVM/QEMU Hypervisor                                    │
│                                                        │
│  Windows Server 2022 (10.0.0.23)                       │
│  ├─ RDP: 3389/TCP (tunnel only via firewall)          │
│  ├─ WireGuard: UDP/51820                              │
│  ├─ Tunnel IP: 10.0.8.1/24                            │
│  └─ Private IP: 192.168.10.165                        │
│                                                        │
│  Windows 10 Client A (10.0.0.15)                       │
│  ├─ Tunnel IP: 10.0.8.2/24                            │
│  ├─ Private IP: 192.168.20.162                        │
│  └─ AllowedIPs: 10.0.8.1/32 (VPS only)               │
│                                                        │
│  Windows 10 Client B (10.0.0.22)                       │
│  ├─ Tunnel IP: 10.0.8.3/24                            │
│  ├─ Private IP: 192.168.30.154                        │
│  └─ AllowedIPs: 10.0.8.1/32 (VPS only)               │
│                                                        │
│  WireGuard Tunnel: 10.0.8.0/24 (encrypted)            │
│  ├─ Transport: UDP/51820 over WAN (10.0.0.0/24)       │
│  └─ Clients cannot reach each other                   │
└────────────────────────────────────────────────────────┘
```

## Security Model

**Layer 1: Firewall**
- RDP (3389) blocked from WAN (10.0.0.0/24)
- RDP allowed from tunnel (10.0.8.0/24) only
- SMB (445) blocked from untrusted networks
- WireGuard (UDP/51820) allowed

**Layer 2: Encryption**
- WireGuard tunnel (ChaCha20-Poly1305)
- RDP traffic encapsulated over UDP/51820

**Layer 3: Isolation**
- Client AllowedIPs = 10.0.8.1/32 (VPS endpoint only)
- Clients cannot reach each other
- Private networks non-routable

**Result:** RDP accessible ONLY via encrypted tunnel. Direct WAN access denied.

## Testing & Validation

**Pre-hardening findings:**
- Port 135 (RPC): OPEN on 10.0.0.23
- Port 445 (SMB): OPEN on 10.0.0.23
- SMB signing not required (Nessus Medium severity)

**Post-hardening validation:**
- Firewall rules applied and verified
- RDP accessible over tunnel (10.0.8.1)
- RDP blocked from WAN (10.0.0.23)
- SMB vulnerability remediated via firewall block
- NIC rotation tested (tunnel survives IP changes)

See `docs/security.md` and `/screenshots/` for detailed scan results.

## Setup

1. Create VMs and networks (see `docs/setup.md`)
2. Install WireGuard on all three machines
3. Deploy configs from `configs/wireguard/`
4. Apply firewall rules from `configs/firewall/`
5. Verify: RDP works over tunnel, blocked from WAN

## Files

- `README.md` — This file
- `docs/setup.md` — Lab setup steps
- `docs/wireguard.md` — WireGuard configuration
- `docs/security.md` — Testing methodology and results
- `configs/wireguard/` — Sanitized WireGuard configs
- `configs/firewall/` — Firewall rules reference
- `screenshots/` — Testing evidence

## Scope

This is a lab proof-of-concept. Production deployment would require:
- Monitoring and logging
- High availability
- Certificate management
- Advanced hardening
