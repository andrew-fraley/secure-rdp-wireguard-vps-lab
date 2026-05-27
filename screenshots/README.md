# Testing Evidence

## Pre-Hardening Findings

**winserport445vulnfindings.jpg**
- Windows Server 2022 port 445 (SMB) vulnerability findings
- Shows SMB exposed on VPS before firewall hardening

**nessusessentialscans01.jpg**
- Nessus Essentials scan overview
- Scan targets: 10.0.0.1 (host), 10.0.0.23 (VPS), 10.0.0.15 (Client A)
- Pre-hardening vulnerability status

**nessusessentialscans02.jpg**
- Detailed Nessus findings
- SMB Signing Not Required vulnerability on 10.0.0.23
- High severity findings on host (out of scope)

**ness_ess_networkscan_01.pdf**
- Full Nessus Essentials Network Scan Report
- Complete pre-hardening baseline
- Lists all discovered services and vulnerabilities

## Post-Hardening Validation

See security.md for:
- Firewall rules applied
- Ports blocked/filtered post-hardening
- RDP accessible over tunnel, blocked from WAN
- Network isolation verified
- NIC rotation test results

## Testing Commands

Port scan:
```bash
nmap -p 135,445,3389,51820 10.0.0.23
```

WireGuard status:
```powershell
wg show
```

RDP test:
```powershell
mstsc /v:10.0.8.1       # Works (tunnel)
mstsc /v:10.0.0.23      # Fails (WAN)
```

Network isolation:
```powershell
ping 192.168.30.154     # No route (Client B private)
```
