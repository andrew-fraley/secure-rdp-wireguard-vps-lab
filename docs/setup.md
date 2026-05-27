# Setup Guide

## Network Setup

Create these virtual networks in virt-manager:

**WAN (virbr-wan)** - NAT, public simulation
```
Network: 10.0.0.0/24
Gateway: 10.0.0.1
DHCP: 10.0.0.10-10.0.0.50
Mode: NAT
STP: on
```

**Client A Private (virbr-cla)** - Isolated
```
Network: 192.168.20.0/24
Gateway: 192.168.20.1
DHCP: 192.168.20.100-192.168.20.200
Mode: Isolated
STP: on
```

**Client B Private (virbr-clb)** - Isolated
```
Network: 192.168.30.0/24
Gateway: 192.168.30.1
DHCP: 192.168.30.100-192.168.30.200
Mode: Isolated
STP: on
```

**VPS Private (virbr-vps)** - Isolated
```
Network: 192.168.10.0/24
Gateway: 192.168.10.1
DHCP: Disabled (static only)
Mode: Isolated
STP: on
```

## VM Creation

**Windows Server 2022 (VPS)**
- Name: wglab-vps
- CPU: 4 cores
- RAM: 8 GB
- Disk: 60 GB
- NIC1: virbr-wan (DHCP → 10.0.0.23)
- NIC2: virbr-vps (static 192.168.10.165)

**Windows 10 Client A**
- Name: wglab-client-a
- CPU: 2 cores
- RAM: 2 GB
- Disk: 40 GB
- NIC1: virbr-wan (DHCP → 10.0.0.15)
- NIC2: virbr-cla (static 192.168.20.162)

**Windows 10 Client B**
- Name: wglab-client-b
- CPU: 2 cores
- RAM: 2 GB
- Disk: 40 GB
- NIC1: virbr-wan (DHCP → 10.0.0.22)
- NIC2: virbr-clb (static 192.168.30.154)

## WireGuard Installation

On all three VMs:
1. Download WireGuard from wireguard.com
2. Install (default settings)
3. Do NOT create tunnel yet

## WireGuard Configuration

### Generate Keys (on Linux host)
```bash
wg genkey | tee server-private.key | wg pubkey > server-public.key
wg genkey | tee clienta-private.key | wg pubkey > clienta-public.key
wg genkey | tee clientb-private.key | wg pubkey > clientb-public.key
```

### Deploy Configs
Use sanitized templates from `configs/wireguard/`, fill in actual keys, save as `wg0.conf` in:
- VPS: `C:\Users\<user>\AppData\Local\WireGuard\Configs\`
- Clients: Same location

### Activate Tunnels
In WireGuard GUI:
1. Add Empty Tunnel
2. Browse to wg0.conf
3. Activate

Verify:
```powershell
wg show
# Should show peers with recent "latest handshake"
```

## Firewall Rules

### VPS (Windows Server)
Open PowerShell as Admin:

```powershell
# Block RDP from WAN
New-NetFirewallRule -DisplayName "RDP Block WAN" `
  -Direction Inbound -Action Block -Protocol TCP -LocalPort 3389 `
  -RemoteAddress "10.0.0.0/24"

# Allow RDP from tunnel
New-NetFirewallRule -DisplayName "RDP Allow Tunnel" `
  -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3389 `
  -RemoteAddress "10.0.8.0/24"

# Block SMB from WAN
New-NetFirewallRule -DisplayName "SMB Block WAN" `
  -Direction Inbound -Action Block -Protocol TCP -LocalPort 445 `
  -RemoteAddress "10.0.0.0/24"

# Allow WireGuard
New-NetFirewallRule -DisplayName "WireGuard Inbound" `
  -Direction Inbound -Action Allow -Protocol UDP -LocalPort 51820

# Set default to deny
Set-NetFirewallProfile -Profile Any -DefaultInboundAction Block
```

### Clients (Windows 10)
Open PowerShell as Admin:

```powershell
# Allow WireGuard
New-NetFirewallRule -DisplayName "WireGuard Inbound" `
  -Direction Inbound -Action Allow -Protocol UDP -LocalPort 51820

# Set default to deny
Set-NetFirewallProfile -Profile Any -DefaultInboundAction Block
```

## Validation

**RDP over tunnel (should work):**
```powershell
mstsc /v:10.0.8.1
```

**RDP from WAN (should fail):**
```powershell
mstsc /v:10.0.0.23
# Expected: Connection refused
```

**Network isolation:**
```powershell
ping 192.168.30.154
# Expected: 100% packet loss
```

**WireGuard status:**
```powershell
wg show
# Check for recent "latest handshake" on both peers
```

## NIC Rotation Test

To simulate client IP changes:
1. Create additional virtual NIC on client VM
2. Attach to virbr-wan (gets different DHCP IP)
3. Verify WireGuard tunnel stays active
4. Verify RDP still works over tunnel

Alternatively, swap private NICs between clients to test isolation.
