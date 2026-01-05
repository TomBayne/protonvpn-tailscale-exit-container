# Tailscale Exit Node via ProtonVPN (Docker Compose)

A Docker Compose stack that creates a **Tailscale exit node** with **split tunneling**:
- **Tailscale peer traffic** routes directly via the VPS public IP (for direct connections)
- **Exit node client traffic** routes through ProtonVPN (for privacy)

## Features

- **Direct peer connections** - Tailscale UDP traffic bypasses the VPN, enabling direct peer-to-peer connections instead of DERP relay
- **VPN-routed exit traffic** - All traffic from exit node clients is routed through ProtonVPN
- **DNS leak prevention** - DNS queries from exit node clients use ProtonVPN DNS
- **Fully containerized** - No changes to host networking required
- **Automatic setup** - All routing rules configured automatically on container start

## Architecture

```
VPS Host
+---------------------------------------------------------------+
|                                                               |
|  +---------------+      +-------------------------------+     |
|  | gluetun       |      | tailscale                     |     |
|  | (ProtonVPN)   |      |                               |     |
|  |               |      |  eth0 -----> Direct Internet  |     |
|  | 172.30.0.2    |<-----+  eth1: 172.30.0.3             |     |
|  | tun0 --> VPN  |      |  tailscale0: Exit node traffic|     |
|  +---------------+      +-------------------------------+     |
|         |                        |                            |
|   vpn_internal                default                         |
|  (172.30.0.0/24)         (Docker bridge)                      |
|                                  |                            |
|                          UDP 41641 exposed                    |
+------------------------------+--------------------------------+
                               |
                      Direct peer connections
```

**Traffic flow:**
1. Tailscale's own UDP traffic -> direct internet via VPS IP
2. Exit node client traffic -> marked with fwmark -> routed via gluetun -> ProtonVPN

## Requirements

- Linux VPS with Docker and Docker Compose
- ProtonVPN account with OpenVPN credentials
- Tailscale account with auth key
- UDP port 41642 open in cloud firewall for direct connections

## Quick Start

1. Clone this repository
2. Copy the example environment file:
   ```bash
   cp .env.example .env
   ```
3. Edit `.env` with your credentials (see Configuration below)
4. Deploy:
   ```bash
   docker compose up -d
   ```
5. Approve the exit node in [Tailscale Admin Console](https://login.tailscale.com/admin/machines)

## Configuration

Edit `.env` with your credentials:

```dotenv
# Tailscale auth key (from https://login.tailscale.com/admin/settings/keys)
TS_AUTHKEY=tskey-auth-xxxxx
TS_HOSTNAME=proton-exitnode

# ProtonVPN OpenVPN credentials (from https://account.protonvpn.com/account#openvpn)
# NOTE: These are NOT your Proton account email/password
PROTON_OPENVPN_USER=xxxxx
PROTON_OPENVPN_PASSWORD=xxxxx

# Server location
PROTON_SERVER_COUNTRIES=Norway
```

## Firewall Setup

For direct peer connections (avoiding DERP relay), open UDP port 41642 on your VPS:

**Oracle Cloud (OCI):**
- VCN -> Security Lists -> Add Ingress Rule
- Source: 0.0.0.0/0, Protocol: UDP, Port: 41642

**Other providers:** Configure equivalent UDP ingress rule.

## Verification

### Check direct connectivity
```bash
docker exec tailscale-exitnode tailscale netcheck
```
Should show your VPS public IP, not the VPN IP.

### Check VPN routing
```bash
docker exec tailscale-exitnode ip route get 8.8.8.8 mark 0x1
```
Should show routing via 172.30.0.2 (gluetun).

### Test from client
1. Enable the exit node on a Tailscale client
2. Check your IP: `curl ifconfig.me` - should show ProtonVPN IP
3. Run a DNS leak test - should show ProtonVPN DNS

## Containers

| Container | Purpose |
|-----------|---------|
| `protonvpn-gateway` | Gluetun - establishes VPN tunnel to ProtonVPN |
| `gluetun-firewall` | Configures forwarding rules in gluetun |
| `tailscale-exitnode` | Tailscale exit node with dual networking |
| `routing-setup` | Configures split tunneling iptables/routing rules |

## Management

```bash
# Start
docker compose up -d

# Stop
docker compose down

# View logs
docker compose logs -f

# Restart after config change
docker compose down && docker compose up -d
```

## Troubleshooting

**DERP relay instead of direct connection:**
- Ensure UDP 41642 is open in cloud firewall
- Check `docker exec tailscale-exitnode tailscale netcheck` shows your VPS IP

**DNS leaking (showing VPS DNS instead of ProtonVPN):**
- Check routing-setup logs: `docker logs routing-setup`
- Verify DNS config: `docker exec tailscale-exitnode cat /etc/resolv.conf`

**Connection timeouts when using exit node:**
- Check gluetun is healthy: `docker ps`
- Check gluetun logs: `docker logs protonvpn-gateway`
- Verify forwarding rules: `docker exec protonvpn-gateway iptables -L FORWARD`

## License

MIT
