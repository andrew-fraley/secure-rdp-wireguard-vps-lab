# WireGuard Configuration

## Overview

WireGuard creates an encrypted tunnel between VPS and clients. This lab uses a hub-and-spoke model: VPS is the hub, clients initiate connections.

## VPS Config (server.conf)

```ini
[Interface]
PrivateKey = <SERVER-PRIVATE-KEY>
ListenPort = 51820
Address = 10.0.8.1/24

[Peer]
PublicKey = <CLIENT-A-PUBLIC-KEY>
AllowedIPs = 10.0.8.2/32
PersistentKeepAlive = 25

[Peer]
PublicKey = <CLIENT-B-PUBLIC-KEY>
AllowedIPs = 10.0.8.3/32
PersistentKeepAlive = 25
```

**Key points:**
- No `Endpoint` on server (VPS is passive, clients initiate)
- `AllowedIPs = /32` restricts each client to exact IP
- `PersistentKeepAlive` on peers keeps tunnel alive through NAT

## Client A Config (client-a.conf)

```ini
[Interface]
PrivateKey = <CLIENT-A-PRIVATE-KEY>
ListenPort = 51820
Address = 10.0.8.2/24

[Peer]
PublicKey = <SERVER-PUBLIC-KEY>
Endpoint = 10.0.0.23:51820
AllowedIPs = 10.0.8.1/32
PersistentKeepAlive = 25
```

**Key points:**
- `Endpoint = 10.0.0.23:51820` points to VPS WAN IP (must be static)
- `AllowedIPs = 10.0.8.1/32` means traffic to VPS only (no client-to-client)
- `PersistentKeepAlive = 25` maintains tunnel through NAT (heartbeat every 25 sec)

## Client B Config (client-b.conf)

Identical to Client A except `Address = 10.0.8.3/24`.

## Key Generation

```bash
# On Linux host, run once:
wg genkey | tee server-private.key | wg pubkey > server-public.key
wg genkey | tee clienta-private.key | wg pubkey > clienta-public.key
wg genkey | tee clientb-private.key | wg pubkey > clientb-public.key
```

- Private keys: Keep secret, never share
- Public keys: Distribute to peers

## Deployment

1. Generate keys on Linux host
2. Fill templates with actual key values
3. Save as `wg0.conf` on each machine
4. Load via WireGuard GUI
5. Activate tunnels

## Verification

```powershell
wg show
```

Look for:
- `latest handshake` timestamp (should be recent, within 60 seconds)
- `transfer` stats showing traffic flowing
- Both peers listed on VPS

If "latest handshake: never", check:
- VPS firewall allows UDP 51820
- Client endpoint IP is correct (10.0.0.23)
- Keys are properly formatted

## What AllowedIPs Does

`AllowedIPs = 10.0.8.1/32` on client means:
- Traffic destined for 10.0.8.1 → encrypted through tunnel
- Traffic destined for 10.0.8.2 or 10.0.8.3 → NOT encrypted, dropped by kernel
- This prevents client-to-client communication even if they knew each other's IPs

## What PersistentKeepAlive Does

NAT gateways (home routers) forget connection state after ~30 seconds idle. Without keepalive:
1. Client sends packet to VPS (creates NAT entry)
2. Time passes, NAT entry expires
3. VPS sends packet back → router drops it (doesn't know the connection)
4. Tunnel appears broken

`PersistentKeepAlive = 25` sends minimal encrypted packet every 25 seconds:
- Refreshes NAT entry before timeout
- Tunnel stays active during idle periods
- Minimal bandwidth impact (~50 bytes per 25 sec)
