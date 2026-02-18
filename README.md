# tailscale-runpod

SSH into RunPod, Vast.ai, and other cloud GPU instances using Tailscale mesh VPN.

## Install

```bash
npx skills add koshimazaki/tailscale-runpod
```

## What This Skill Does

Provides instructions for setting up Tailscale on cloud GPU instances (RunPod, Vast.ai, Lambda Labs, Paperspace) to enable:

- **Direct SSH access** without port forwarding
- **Persistent Tailscale IP** across pod restarts
- **Secure connection** via WireGuard encryption
- **No exposed ports** to the public internet

## Quick Start

On your cloud instance (via JupyterLab terminal or RunPod web SSH):

```bash
curl -fsSL https://tailscale.com/install.sh | bash && /usr/sbin/tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 5 && tailscale up --ssh
```

> Most cloud GPU containers don't have Tailscale in their default apt repos, so the official install script is required.

Click the auth URL to login. Then SSH from your local machine:

```bash
ssh root@<TAILSCALE_IP>
```

## Key Features

- **Container-optimized** - Uses `--tun=userspace-networking` for Docker environments
- **Persistence** - Stores state in `/workspace/` to survive restarts
- **SSH enabled** - `--ssh` flag enables Tailscale's built-in SSH server
- **Multi-platform** - Works on RunPod, Vast.ai, Lambda Labs, Paperspace
- **Official install** - Uses Tailscale's install script (apt repos unavailable on most cloud GPU images)

## Troubleshooting

If SSH connection fails, run these on the pod:

```bash
# Check if Tailscale is connected
tailscale status

# Re-authenticate if needed
tailscale up --ssh
```

If Tailscale daemon died (after pod sleep/wake):

```bash
/usr/sbin/tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 5 && tailscale up --ssh
```

## Why This Skill?

Standard Tailscale docs don't cover the container-specific requirements for cloud GPU platforms. This skill includes the gotchas we discovered:

1. Must use `--tun=userspace-networking` (no kernel TUN in containers)
2. Must use `--ssh` flag (enables Tailscale's built-in SSH server for container environments)
3. Must persist state to `/workspace/` to survive pod restarts
4. Browser auth required — authkey doesn't work reliably in unprivileged containers

## License

MIT
