# Tailscale Exit Node via ProtonVPN (Docker Compose)

A small Docker Compose stack that turns a VPS into a **Tailscale exit node** whose **exit traffic is routed through ProtonVPN**, while keeping **peer-to-exit-node connectivity on the VPS public IP**.

This version is designed for the common case where you **already run Tailscale on the VPS host** and want to keep using that for normal access to the VPS (SSH, admin, etc). The Docker stack **does not run `tailscaled`** to avoid `tailscale0`/TUN conflicts.

## Key features

- **Keeps your existing host Tailscale access**
  - You continue to access the VPS over the host’s Tailscale installation.
- **Tailscale exit node advertised from the VPS public IP**
  - The exit node is the host Tailscale node, so peers connect to the VPS public endpoint (direct when possible).
- **All exit-node internet traffic goes out via ProtonVPN**
  - Policy routing marks packets arriving on `tailscale0` and routes them to the ProtonVPN (Gluetun) gateway container.
- **No manual changes inside containers**
  - Routing/iptables rules are applied automatically by the `netsetup` container at startup.
- **Kill-switch capable**
  - Gluetun can enforce firewall rules, depending on your configuration.
- **State is persisted**
  - Gluetun state is persisted via a Docker volume.

## Architecture overview

- **Host Tailscale (`tailscaled` on the VPS host)**:
  - Joins your tailnet
  - Advertises `--advertise-exit-node`
  - Creates `tailscale0` on the host

- `gluetun` (bridge network):
  - Establishes the ProtonVPN tunnel
  - Provides a gateway IP on a dedicated Docker bridge

- `netsetup` (host network, privileged):
  - Enables IP forwarding
  - Adds iptables mangle marking for packets arriving via `tailscale0` (excluding tailnet-destined traffic)
  - Adds an `ip rule` + routing table so marked traffic uses the Gluetun gateway
  - Adds basic forward rules between `tailscale0` and the Docker bridge

Result:
- **Tailscale control + peer connectivity**: via VPS public IP (direct when possible)
- **Exit-node internet egress**: via ProtonVPN tunnel

## Requirements

### VPS / host
- Linux VPS with a public IPv4 address (IPv6 optional).
- Docker + Docker Compose.
- Ability to run a privileged container (for policy routing + iptables on the host namespace).
- **Tailscale installed and running on the VPS host** (this stack relies on the host `tailscale0` interface).

### Network / firewall
- Allow inbound **UDP 41641** to the VPS (cloud firewall + OS firewall if you manage one).
  - Without this, devices are more likely to fall back to DERP.
- Allow normal outbound UDP/TCP (for ProtonVPN and Tailscale).

### Accounts / credentials
- **ProtonVPN OpenVPN “service credentials”**
  - These are not your Proton account email/password.

## Files

- `docker-compose.yml` - the stack (Gluetun + netsetup)
- `.env` - environment variables (recommended)

## Configuration

Create a `.env` file alongside your `docker-compose.yml`:

```dotenv
# ProtonVPN OpenVPN service credentials
PROTON_OPENVPN_USER=xxxxxxx
PROTON_OPENVPN_PASSWORD=yyyyyyy

# Optional server selection
# Examples: "United Kingdom", "Netherlands", etc
PROTON_SERVER_COUNTRIES=United Kingdom
# PROTON_SERVER_CITIES=London
````

### About Tailscale setup on the VPS (outside the container)

This stack assumes **Tailscale is already set up on the VPS host** and that `tailscale0` exists.

Outside the containers, you need to:

* Configure the host Tailscale node to **advertise as an exit node** (and approve it in the admin console if required).
* Open inbound **UDP 41641** in your cloud firewall/security group (and OS firewall if applicable).
* Ensure the VPS allows forwarding/routing (the stack sets forwarding via `sysctl`, but hardened hosts may override or block it).
* Apply any organisational Tailscale admin settings (e.g. whether exit nodes require approval).

No host commands are included here; keep those steps in your own infra notes.

## Deploy

From the directory containing `docker-compose.yml` and `.env`:

```bash
docker compose up -d
```

Then:

1. In the Tailscale admin console, confirm the **host VPS** device is present.
2. If your tailnet requires approval for exit nodes, enable “Use as exit node” for the **host VPS** device.

## Using the exit node

On a client device in your tailnet, select the VPS host device as your exit node (UI varies by platform).

Once enabled, the client’s internet traffic should appear to originate from **ProtonVPN**, not from your VPS.

## Verification

### 1) Confirm direct connectivity (avoid DERP when possible)

From a client device:

* Use your normal Tailscale tooling/UI to check whether the connection to the exit node is **direct**.
* If it still shows DERP:

  * Re-check inbound UDP 41641 reachability on the VPS.
  * Confirm there’s no upstream NAT/firewall behaviour preventing UDP hole punching.

### 2) Confirm exit traffic is via ProtonVPN

From a client device with the exit node enabled:

* Check your public IP (browser "what is my IP").
* It should match a ProtonVPN egress range, not your VPS ISP/provider IP.

## Notes and limitations

* **Performance**: Exit-node traffic is double-tunnelled (Tailscale overlay → VPS → ProtonVPN). Expect higher latency and lower throughput than “direct to VPS” without ProtonVPN.
* **Direct vs DERP**: Even with UDP 41641 open, some networks still force DERP (symmetric NAT, restrictive firewalls, captive networks).
* **IPv6**: Forwarding is enabled, but behaviour depends on ProtonVPN configuration and your VPS IPv6 support.
* **Privilege**: `netsetup` is privileged because it modifies host routing/iptables. Treat this as infrastructure, not an “app container”.
