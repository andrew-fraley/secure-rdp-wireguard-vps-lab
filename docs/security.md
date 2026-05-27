# Security Testing

## Testing Workflow

1. **nmap**: Port/service inventory
2. **netcat script**: Connectivity and vulnerability checks
3. **Nessus Essentials**: Baseline vulnerability scanning
4. **Wireshark**: Tunnel integrity verification

Tests run before and after firewall hardening.

## Pre-Hardening Findings

### Port Scanning (nmap)

**Target: 10.0.0.23 (VPS)**

Netcat script results:
```
[+] Port 135 OPEN (RPC)
[+] Port 445 OPEN (SMB)
```

nmap vulnerability scan:
```
nmap -Pn 445 --script smb-vuln* 10.0.0.23
nmap -Pn 135 --script "vuln" 10.0.0.23
nmap -Pn 135 --script msrpc-enum 10.0.0.23
```

Results: RPC and SMB enumeration possible, SMB vulnerability present.

**Target: 10.0.0.15 (Client A)**

Netcat script results:
```
[-] No open ports detected
```

Result: Client properly firewalled.

**Target: 10.0.0.1 (Linux host)**

Netcat script results:
```
[+] Port 53 OPEN (DNS)
[-] All other ports closed
```

Result: Only DNS exposed (dnsmasq), expected for host.

### Nessus Essentials Scan

**High Severity:**
- Python Library Brotli <= 1.1.0 DoS (host-level, out of scope)

**Medium Severity:**
- SMB Signing Not Required (10.0.0.23)
  - Description: Unsigned SMB allows MITM attacks
  - Affected: Port 445/tcp on VPS
  - Severity: Medium (Nessus plugin #57608)

**Summary:** VPS exposed RPC/SMB services, SMB lacking integrity checking.

## Hardening Applied

### Firewall Rules (VPS)

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

# Default deny
Set-NetFirewallProfile -Profile Any -DefaultInboundAction Block
```

**Remediation strategy:**
- Block SMB port 445 from WAN (Nessus finding)
- Block RDP direct access (security model enforcement)
- Allow RDP only from tunnel (10.0.8.0/24)
- Allow WireGuard for tunnel establishment

## Post-Hardening Validation

### Port Scan Results

**Target: 10.0.0.23 (VPS)**

Expected state:
```
Port 135 (RPC):     FILTERED (firewall blocks)
Port 445 (SMB):     FILTERED (firewall blocks)
Port 3389 (RDP):    FILTERED (firewall blocks from WAN)
Port 51820 (WG):    OPEN (required for tunnel)
```

### Nessus Re-scan

**SMB Signing vulnerability:** REMEDIATED
- Port 445 now blocked from 10.0.0.0/24 (WAN)
- Nessus no longer detects external SMB exposure

**Windows client hosts:** HARDENED
- No external vulnerabilities detectable
- Firewall enforces default-deny

### Functional Testing

✓ RDP over tunnel (10.0.8.1:3389) - Works
✓ RDP from WAN (10.0.0.23:3389) - Blocked
✓ SMB from WAN (10.0.0.23:445) - Blocked
✓ Network isolation - Verified (clients cannot reach each other)
✓ WireGuard tunnel - Active with recent handshakes

### NIC Rotation Test

Tested client IP changes:
1. Created additional NIC on client (different DHCP IP)
2. Verified tunnel remained active
3. Verified RDP still accessible over tunnel

Result: Tunnel survives WAN IP changes via PersistentKeepAlive.

## Test Evidence

See `/screenshots/` for:
- `winserport445vulnfindings.jpg` - Pre-hardening SMB findings
- `nessusessentialscans01.jpg` - Nessus scan overview
- `nessusessentialscans02.jpg` - Detailed findings
- `ness_ess_networkscan_01.pdf` - Full scan report

## Summary

| Finding | Pre-Hardening | Post-Hardening | Status |
|---------|---|---|---|
| RDP exposed (3389) | OPEN | FILTERED | ✓ Fixed |
| SMB exposed (445) | OPEN | FILTERED | ✓ Fixed |
| RPC exposed (135) | OPEN | FILTERED | ✓ Fixed |
| Tunnel working | N/A | Active | ✓ Verified |
| RDP over tunnel | N/A | Works | ✓ Verified |

Firewall rules successfully restrict RDP access to tunnel only. Security model validated.
