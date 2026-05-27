# Windows Firewall Rules

## VPS Rules (Windows Server 2022)

Run in PowerShell as Admin:

```powershell
# Rule 1: Block RDP from WAN (priority)
New-NetFirewallRule -DisplayName "RDP Block WAN" `
  -Direction Inbound -Action Block -Protocol TCP -LocalPort 3389 `
  -RemoteAddress "10.0.0.0/24"

# Rule 2: Allow RDP from tunnel
New-NetFirewallRule -DisplayName "RDP Allow Tunnel" `
  -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3389 `
  -RemoteAddress "10.0.8.0/24"

# Rule 3: Block SMB from WAN
New-NetFirewallRule -DisplayName "SMB Block WAN" `
  -Direction Inbound -Action Block -Protocol TCP -LocalPort 445 `
  -RemoteAddress "10.0.0.0/24"

# Rule 4: Allow WireGuard
New-NetFirewallRule -DisplayName "WireGuard Inbound" `
  -Direction Inbound -Action Allow -Protocol UDP -LocalPort 51820

# Rule 5: Default deny
Set-NetFirewallProfile -Profile Any -DefaultInboundAction Block
```

## Client Rules (Windows 10)

Run in PowerShell as Admin:

```powershell
# Allow WireGuard only
New-NetFirewallRule -DisplayName "WireGuard Inbound" `
  -Direction Inbound -Action Allow -Protocol UDP -LocalPort 51820

# Default deny
Set-NetFirewallProfile -Profile Any -DefaultInboundAction Block
```

## Verify Rules Applied

```powershell
# On VPS, check inbound rules:
Get-NetFirewallRule -Direction Inbound -DisplayName "*RDP*","*SMB*","*WireGuard*" | `
  Format-Table DisplayName, Enabled, Action

# Expected:
# DisplayName           Enabled Action
# --------             ------- ------
# RDP Allow Tunnel        True Allow
# RDP Block WAN           True Block
# SMB Block WAN           True Block
# WireGuard Inbound       True Allow
```

## Test Access

```powershell
# RDP over tunnel (should work):
mstsc /v:10.0.8.1

# RDP from WAN (should fail):
mstsc /v:10.0.0.23
# Expected: Connection refused

# Port check:
Test-NetConnection -ComputerName 10.0.0.23 -Port 3389
# Expected: TcpTestSucceeded = False (blocked by firewall)
```

## Rule Order

Rules evaluated top-to-bottom, first match wins. Order matters:
1. **RDP Block WAN** (explicit deny from 10.0.0.0/24)
2. **RDP Allow Tunnel** (explicit allow from 10.0.8.0/24)
3. **SMB Block WAN** (deny SMB from untrusted)
4. **WireGuard Allow** (allow tunnel endpoint)
5. **Default deny** (implicit reject all else)

If rules reversed, RDP Allow would match before RDP Block.
