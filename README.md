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

On your cloud instance:

```bash
curl -fsSL https://tailscale.com/install.sh | sh && tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 2 && tailscale up --ssh
```

Then SSH from your local machine:

```bash
ssh root@<TAILSCALE_IP>
```

## Key Features

- **Container-optimized** - Uses `--tun=userspace-networking` for Docker environments
- **Persistence** - Stores state in `/workspace/` to survive restarts
- **SSH enabled** - The `--ssh` flag is critical for container SSH access
- **Multi-platform** - Works on RunPod, Vast.ai, Lambda Labs, Paperspace

## Why This Skill?

Standard Tailscale docs don't cover the container-specific requirements for cloud GPU platforms. This skill includes the gotchas we discovered:

1. Must use `--tun=userspace-networking` (no kernel TUN in containers)
2. Must use `--ssh` flag (enables Tailscale's built-in SSH server)
3. Must persist state to survive pod restarts

## License

MIT
