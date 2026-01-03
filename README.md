# Tailscale Exit Node via ProtonVPN (Docker Compose)

A small Docker Compose stack that turns a VPS into a **Tailscale exit node** whose **exit traffic is routed through ProtonVPN**, while keeping **peer-to-exit-node connectivity on the VPS public IP** (to avoid being forced onto DERP purely because the VPN egress hides the endpoint).

## Key features

- **Tailscale exit node advertised from the VPS public IP**
  - Tailscale runs in `host` networking, so peers can connect directly to the VPS when UDP is reachable.
- **All exit-node internet traffic goes out via ProtonVPN**
  - Policy routing marks traffic arriving on `tailscale0` and routes it to the ProtonVPN (Gluetun) gateway container.
- **No manual changes inside containers**
  - Routing rules are applied automatically by the `netsetup` container at startup.
- **Kill-switch capable**
  - Gluetun can enforce firewall rules, depending on your configuration.
- **State is persisted**
  - Separate volumes for Gluetun and Tailscale state.

## Architecture overview

- `tailscale` (host network): joins your tailnet and advertises `--advertise-exit-node`
- `gluetun` (bridge network): establishes the ProtonVPN tunnel and provides a gateway IP
- `netsetup` (host network, privileged): configures:
  - IP forwarding
  - iptables mangle marking for packets arriving via `tailscale0`
  - `ip rule` + routing table to send marked traffic via the Gluetun gateway

Result:
- **Tailscale control + peer connectivity**: via VPS public IP (direct when possible)
- **Exit-node internet egress**: via ProtonVPN tunnel

## Requirements

### VPS / host
- Linux VPS with a public IPv4 address (IPv6 optional).
- Docker + Docker Compose.
- Ability to run a privileged container (for policy routing + iptables on the host namespace).

### Network / firewall
- Allow inbound **UDP 41641** to the VPS (cloud firewall + OS firewall if you manage one).
  - Without this, devices are more likely to fall back to DERP.
- Allow normal outbound UDP/TCP (for ProtonVPN and Tailscale).

### Accounts / credentials
- **Tailscale auth key** (preferably pre-authorised, and ideally reusable/ephemeral depending on your preference).
- **ProtonVPN OpenVPN “service credentials”**
  - These are not your Proton account email/password.

## Files

- `docker-compose.yml` - the stack
- `.env` - environment variables (recommended)

## Configuration

Create a `.env` file alongside your `docker-compose.yml`:

```dotenv
# Tailscale
TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxxxxxx
TS_HOSTNAME=proton-exitnode

# ProtonVPN OpenVPN service credentials
PROTON_OPENVPN_USER=xxxxxxx
PROTON_OPENVPN_PASSWORD=yyyyyyy

# Optional server selection
# Examples: "United Kingdom", "Netherlands", etc
PROTON_SERVER_COUNTRIES=United Kingdom
# PROTON_SERVER_CITIES=London
````

### About Tailscale setup on the VPS (outside the container)

This stack runs `tailscaled` in a container using the host network. You do **not** need to install Tailscale on the VPS host OS for it to work, but you can if you want.

However, you **do** need to handle any VPS-level setup outside the containers, such as:

* Opening inbound UDP 41641 in your cloud firewall/security group.
* Ensuring the VPS allows forwarding/routing (the stack sets forwarding via `sysctl`, but hardened hosts may override or block it).
* Any organisational Tailscale admin settings (e.g. whether exit nodes require approval).

## Deploy

From the directory containing `docker-compose.yml` and `.env`:

```bash
docker compose up -d
```

Then:

1. In the Tailscale admin console, confirm the new device has joined.
2. If your tailnet requires approval for exit nodes, enable “Use as exit node” for this device.

## Using the exit node

On a client device in your tailnet, select the new node as your exit node (UI varies by platform).

Once enabled, the client's internet traffic should appear to originate from **ProtonVPN**, not from your VPS.

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

* **Performance**: Exit-node traffic is now double-tunnelled (Tailscale overlay → VPS → ProtonVPN). Expect higher latency and lower throughput than "direct to VPS" without ProtonVPN.
* **Direct vs DERP**: Even with UDP 41641 open, some networks still force DERP (symmetric NAT, restrictive firewalls, captive networks).
* **IPv6**: This stack enables IPv6 forwarding, but behaviour depends on ProtonVPN configuration and your VPS IPv6 support.
