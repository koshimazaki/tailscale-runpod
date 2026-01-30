---
name: tailscale-runpod
description: Setup Tailscale for SSH access to RunPod, Vast.ai, or any cloud GPU instance. Use when user asks to "setup Tailscale", "SSH to RunPod", "connect to cloud GPU", or needs VPN/mesh networking.
version: 1.0.0
author: koshimazaki
---

# Tailscale for Cloud GPU (RunPod/Vast.ai)

Connect to your cloud GPU instances via SSH using Tailscale mesh VPN.

## Why Tailscale?

- **Direct SSH** - No port forwarding or exposed services
- **Persistent IP** - Same IP even after pod restarts
- **Secure** - End-to-end encrypted, no public exposure
- **Works anywhere** - Bypasses NAT and firewalls

## Quick Setup (Cloud Instance)

**One command to install and connect:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh && tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 2 && tailscale up --ssh
```

Then click the auth URL to login.

### Platform Paths

| Platform | Persistent Storage | Default User |
|----------|-------------------|--------------|
| RunPod | `/workspace/` | `root` |
| Vast.ai | `/workspace/` | `root` |
| Lambda Labs | `/home/ubuntu/` | `ubuntu` |
| Paperspace | `/storage/` | `paperspace` |

**For non-RunPod platforms**, adjust the state path:
```bash
# Lambda Labs
tailscaled --tun=userspace-networking --state=/home/ubuntu/tailscale.state &

# Paperspace
tailscaled --tun=userspace-networking --state=/storage/tailscale.state &
```

## SSH from Local Machine

Once connected:

```bash
ssh root@<TAILSCALE_IP>
```

Find the IP with `tailscale status` on either machine.

---

## Detailed Setup

### Cloud Instance (RunPod/Vast.ai)

#### 1. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

#### 2. Start daemon (container mode)

```bash
tailscaled --tun=userspace-networking --state=/workspace/tailscale.state &
```

| Flag | Purpose |
|------|---------|
| `--tun=userspace-networking` | Required for containers (no kernel TUN device) |
| `--state=/workspace/` | Persist auth across restarts |
| `&` | Run in background |

#### 3. Connect with SSH enabled

```bash
tailscale up --ssh
```

**The `--ssh` flag is critical** - it enables Tailscale's built-in SSH server, required for SSH to work in containers.

#### 4. Get your Tailscale IP

```bash
tailscale ip -4
```

### Local Machine (macOS)

```bash
brew install --cask tailscale
open -a Tailscale
```

Or download from: https://tailscale.com/download

### Local Machine (Linux)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### Local Machine (Windows)

Download from: https://tailscale.com/download/windows

---

## Persist Across Restarts

Use an auth key to avoid re-authenticating:

```bash
tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 2 && tailscale up --ssh --authkey=tskey-auth-xxxxx
```

Generate auth key: https://login.tailscale.com/admin/settings/keys

**Tip:** Add to your pod's startup script or save as `/workspace/start_tailscale.sh`

---

## Commands Reference

| Command | Description |
|---------|-------------|
| `tailscale status` | Show all connected devices |
| `tailscale ip -4` | Get your Tailscale IPv4 |
| `tailscale up --ssh` | Connect with SSH enabled |
| `tailscale down` | Disconnect |
| `tailscale ping <ip>` | Test connectivity to peer |
| `tailscale netcheck` | Network diagnostics |
| `tailscale logout` | Sign out completely |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| **SSH connection timeout** | Ensure you used `--ssh` flag: `tailscale up --ssh` |
| **"tailscaled not running"** | Start daemon: `tailscaled --tun=userspace-networking &` |
| **Peer shows "offline"** | Run `tailscale up --ssh` on that machine |
| **Slow connection (relay)** | Wait for direct connection, or check `tailscale netcheck` |
| **Auth expired** | Run `tailscale up --ssh` again, or use auth key |
| **Version mismatch warning** | Usually harmless, update if issues persist |

### Full Reset

```bash
pkill tailscaled; sleep 1; tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 2 && tailscale up --ssh
```

---

## Example Workflow

```bash
# On RunPod (first time)
curl -fsSL https://tailscale.com/install.sh | sh
tailscaled --tun=userspace-networking --state=/workspace/tailscale.state &
sleep 2
tailscale up --ssh
# Click the URL, login, note the IP (e.g., 100.x.x.x)

# On your Mac
tailscale status  # Verify pod is online
ssh root@100.x.x.x  # Connect!
```

---

## Security Notes

- Tailscale IPs are only accessible within your tailnet
- Traffic is end-to-end encrypted (WireGuard)
- No ports exposed to public internet
- Auth keys can be set to expire or be single-use
