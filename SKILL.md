---
name: cloud-gpu-box
description: Connect to and operate a rented cloud GPU box (RunPod, Vast.ai, Lambda, Paperspace) — Tailscale SSH setup, plus the operational failure modes that silently waste credit: unusable tailnet bulk transfer, pgrep self-match over SSH, shared-GPU VRAM contention, no cross-instance volume, gated model repos, PEP-668 pip. Use when setting up Tailscale, SSHing to a cloud GPU, moving data to or from an instance, or running long training/inference jobs on rented hardware.
version: 2.0.0
author: koshimazaki
---

# Cloud GPU box — connect and operate

Boxes bill by the minute and nothing survives them. Part 1 gets you connected;
Part 2 is the failure modes that cost real credit to learn.

---

# Part 1 — Connect with Tailscale

- **Direct SSH** — no port forwarding or exposed services
- **Persistent IP** — same IP after pod restarts
- **Secure** — end-to-end encrypted, nothing public
- **Works anywhere** — bypasses NAT and firewalls

## Quick setup (cloud instance)

Run in the JupyterLab terminal or the provider's web SSH:

```bash
curl -fsSL https://tailscale.com/install.sh | bash && /usr/sbin/tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 5 && tailscale up --ssh --hostname=<name>
```

> **Why curl|bash?** Most cloud GPU containers don't ship Tailscale in their apt
> repos. The official install script is the reliable path. Skip if your image
> has it pre-installed.

Click the auth URL to log in, then SSH from your machine.

**Name the host.** `--hostname=<name>` beats memorising a 100.x address, and
matters once you have more than one box on the tailnet.

### Platform paths

| Platform | Persistent storage | Default user |
|---|---|---|
| RunPod | `/workspace/` | `root` |
| Vast.ai | `/workspace/` | `root` |
| Lambda Labs | `/home/ubuntu/` | `ubuntu` |
| Paperspace | `/storage/` | `paperspace` |

Adjust `--state=` to match, or auth is lost on restart.

## The three flags that matter

| Flag | Purpose |
|---|---|
| `--tun=userspace-networking` | required in containers — no kernel TUN device |
| `--state=/workspace/…` | persists auth across restarts |
| `--ssh` | enables Tailscale's SSH server — **the only way SSH works in userspace mode** |

## Commands

| Command | Description |
|---|---|
| `tailscale status` | connected devices, and **direct vs DERP relay** |
| `tailscale ping <host>` | round-trip time and path type |
| `tailscale ip -4` | your IPv4 |
| `tailscale netcheck` | network diagnostics |

## Troubleshooting

| Problem | Solution |
|---|---|
| SSH times out | you omitted `--ssh`; run `tailscale up --ssh` again |
| "tailscaled not running" | start the daemon with the userspace flag |
| Peer shows offline | run `tailscale up --ssh` on that machine |
| Auth expired | `tailscale up --ssh` again |
| Slow transfers | **expected — see Part 2. Do not fix this by waiting.** |

Full reset:

```bash
pkill tailscaled; sleep 1; /usr/sbin/tailscaled --tun=userspace-networking --state=/workspace/tailscale.state & sleep 5 && tailscale up --ssh
```

---

# Part 2 — Operating the box

## Moving data: control and bulk are different problems

**Tailscale SSH is for control. Do not move bulk data over it.**

With `--tun=userspace-networking` every tailnet route is relayed and CPU-bound.
Measured against a box on another continent:

| Path | Rate |
|---|---|
| object store → box (HuggingFace, S3) | **86–146 MB/s** |
| laptop → box over tailnet | **~11 KB/s** |
| box → box over tailnet | ~250 KB/s |

The box's own internet is usually excellent. The **tailnet path into it** is the
bottleneck, and no amount of waiting fixes it — a 364 MB test transfer ran 24
minutes before being killed.

**Check the path before trusting it:**

```bash
tailscale ping <host>     # reports direct vs DERP(<region>) and RTT
```

A DERP relay with 300 ms RTT means bulk transfer will not work. Route it
through an object store instead: **upload once from whichever machine has the
fast uplink, then let the box pull.** A datacenter box uploads orders of
magnitude faster than a home connection, so prefer box → store → box over
laptop → box.

### When rsync over Tailscale *is* fine

Small files, or a nearby box with a direct connection. Scripts, configs,
manifests, a few images — all fine.

```bash
rsync -avz --progress --partial root@<host>:/workspace/outputs/ ~/Downloads/out/
```

Many small files move faster in one stream than as separate `scp` calls:

```bash
ssh <host> 'cd /workspace && tar cf - <files>' | tar xf - -C dest/
```

### Pulling results back

Pulling *from* a box is often usable even when pushing *into* it is not — but
measure rather than assume. For anything large, have the box push to the object
store and pull from there.

## Traps that waste a session

**`pgrep -f` / `pkill -f` self-match over SSH.** When you run
`ssh box 'pkill -f foo.sh'`, the remote shell's own command line *contains*
`foo.sh`, so the pattern matches the shell executing it. Observed:

- `pkill -f x.sh` killed the SSH session mid-command. The background jobs it was
  about to launch never started — and the launch still reported success.
- A wait loop `while pgrep -f job; do sleep 20; done` deadlocked against
  processes that merely had the string in their argv, including an earlier
  backgrounded SSH wait-loop and the SSH call that wrote the script via heredoc
  (the whole script text sits in that process's argv).
- `pgrep` reports a finished job as still RUNNING — misleading in the worst
  direction.

**Wait on a terminal marker in the log, never a process pattern:**

```bash
until grep -q JOB_DONE /workspace/job.log; do sleep 20; done
```

If you must match a process, kill by PID, or split the literal
(`P="jo""b.sh"`) so it cannot appear contiguously in your own command line.

**Never edit a script while it is running.** Bash reads by byte offset, so
overwriting a running script makes it jump into the wrong stage. Observed: a
chain script skipped its setup stages and ran the final step against half-built
state. Write to a new path and relaunch.

**Two GPU consumers will fight.** A served inference app (ComfyUI and similar)
holds weights in VRAM after a run — 66 GB in one case — and a separate
training process then OOMs behind it. Free it explicitly between phases:

```bash
curl -s -X POST http://127.0.0.1:8189/free -H 'Content-Type: application/json' \
  -d '{"unload_models":true,"free_memory":true}'
```

**An extracted archive overwrites the directory it lands in.** Local edits to
scripts inside it revert silently on the next extract. Keep edits outside the
extract path, or re-apply them idempotently.

## Nothing survives the box

There is usually **no cross-instance volume**. When the instance dies,
everything on it dies.

- **Ship each stage as it completes**, never batch to the end. A credit wall
  then costs one stage, not the session.
- Make long chains **resumable** — skip a stage whose output already exists.
- **Verify before terminating.** Compare counts on both sides; do not trust
  "the rsync finished". Diff file counts per category.

## Bootstrap gotchas

- **`pip` is PEP-668 blocked** on current images. Use
  `uv tool install "<pkg>[extras]"`, or `pip --break-system-packages`.
- **Gated model repos fail with a misleading error** — "requires approval"
  rather than "unauthenticated". Put the token in place *before* a long pull,
  or you find out 60 GB later.
- **A served app may cache its model list at startup.** Files symlinked in
  afterwards are invisible until restart — the dropdown will not offer them and
  graphs referencing them fail validation with no useful message.

## Long jobs

Detach properly, or an SSH drop kills the work:

```bash
setsid nohup bash job.sh > /workspace/job.log 2>&1 < /dev/null &
disown
```

- Emit **explicit terminal markers** (`STAGE_DONE`, `EXIT=$?`) so progress can
  be waited on without process matching.
- **Measure the rate in the first minute** — sample a counter, sleep, sample
  again. Knowing MB/s or seconds-per-item early tells you whether to keep
  paying for the approach.
- **Batch everything that shares a model load.** Loading usually dominates
  runtime; N items through one load beats N loads by an order of magnitude.
  Check whether the underlying config supports per-item parameters even when
  the CLI does not expose them — in one case this turned 15 model loads into 3.

## Graph-driven tools (ComfyUI and similar)

- **Validate against the live schema before queueing.** Dump `/object_info` and
  check every class and input name. A wrong input name yields a generic
  "failed validation" naming no field.
- **A schema dump taken during startup may be incomplete.** Node packs register
  asynchronously; a class can look missing when it merely had not loaded. Re-dump
  from the running server before concluding a node is absent.
- **Widget `step` values are UI increments, not validation.** A dimension of
  720 may be silently rounded to 704 rather than rejected.
- **A missing custom node fails silently** — the graph falls back to core nodes,
  still "validates", and produces subtly wrong output.

## Choosing a box

- **Size the card from measured VRAM, not hope.** One 22B video model at
  720×1280 peaked at 41 GB quantised and 78 GB in bf16 — a 32 GB card holds
  neither.
- **Prefer local NVMe for the working volume.** Model *loading* often dominates
  batch jobs: the same load took **57 s** on local NVMe versus **~8 min** on a
  network mount. That 8× dwarfs any hourly price difference.
- **Check geography.** A distant box is fine for compute and painful for
  anything interactive.
- **Check the disk quota**, not just the host disk. Weights run to tens of GB
  and a CUDA venv adds 10–15 more.

## Secrets

Do not keep API tokens in plaintext files or paste them into terminals, chat
logs, or scripts. On macOS, store once in the Keychain and read at point of use:

```bash
# once — prompts for the value, echoes nothing, no shell history
security add-generic-password -a "$USER" -s hf-token -w

# at point of use
export HF_TOKEN=$(security find-generic-password -a "$USER" -s hf-token -w)
```

**Environment variable, never argv** — anything passed as a command argument is
visible in `ps` while it runs.

Choose the prompt policy deliberately: *Allow* each time is genuinely stronger
but interrupts unattended runs; *Always Allow* reduces the guarantee to "a file,
but tidier".

To move a token between your own machines without it touching a log, copy the
file directly rather than echoing it:

```bash
rsync -a -e ssh me@boxA:/workspace/.token /workspace/.token && chmod 600 /workspace/.token
```

**The rule that matters more than the mechanism:** an agent or script should
never receive the raw value — it invokes a command that reads it. Keychain
protects storage, not disclosure. A token that reaches a transcript still needs
rotating.

## Cost discipline

Renders are cheap; **idle boxes and re-downloads burn credit.** A 57-clip batch
cost about $0.30 of GPU while the 66 GB weight pull cost more.

**Do the cheap decisive experiment first** when its outcome changes how the
expensive work should be done — otherwise you do the expensive work twice.
