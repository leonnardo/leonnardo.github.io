---
title: "Docker Desktop vs. systemd 256: How I Fixed my Broken WSL Distro"
description: "How Docker Desktop's invasive WSL integration breaks systemd 256 user sessions—and how to fix it with a clean socket proxy architecture."
date: 2026-04-15
tags: [wsl, docker, systemd, linux, archlinux]
---

# Docker Desktop Broke My WSL Distro. Here's How I Fixed It (And Made It Better).

I work at Microsoft. The challenge is _awesome_: building an IDP on top of Azure, Kubernetes, and Linux hosts; basically all the “cool kids” tech.

But there’s one thing that sucks, and it’s hard to avoid: working on a machine running Windows.

Not much I can do about it. I work at Microsoft.

Sure. I *could* set up a native Linux environment as my daily driver. But to be honest, I haven’t had the time.

Between features, bugs, incidents, the AI agents hype, and—*cough*—writing (now that I’ve committed to it), it’s hard to justify going down the rabbit hole of making a Microsoft-compliant Linux setup work.

And even with documentation, not everything works out of the box. There’s Intune, there’s this, there’s that… it’s way too easy for me to get dragged into a rabbit hole and forget about actual work (thanks, ADHD).

So I settled for the middle ground: **WSL**.

It’s not perfect. I don’t get the freedom of running bleeding-edge Wayland compositors like Hyprland or Niri. But I do get access to most of the CLI tools I care about.

Docker, Git, tmux, Neovim — you name it. If it has a CLI or TUI, there’s a way.

And yes, I use Arch, by the way. Had to get that out of the way.

So, WSL... it feels comfortable... until it doesn’t.

There’s no such thing as a free lunch.

## Where It All Went Wrong

If you're running a bleeding-edge distro on WSL — anything with **systemd 256+** — alongside Docker Desktop on Windows, there’s a high chance your user session is silently broken.

I didn’t notice it at first, until this error showed up:

```text
wsl: Failed to start the systemd user session for 'lrabello'. See journalctl for more details.
```

What followed was a deep dive into strace, cgroup internals, and the frustrating realization that Docker Desktop’s WSL integration does far more than it advertises.

It’s not just “helping.” It’s overstepping.

The fix? Evict Docker Desktop from your distro entirely.

Instead of letting it manage your environment, we’re moving to a cleaner architecture that keeps the convenience of the GUI but stops the meddling with your systemd session.

## What Docker Desktop Is Actually Doing to Your Distro

When you toggle that *WSL Integration* switch in Docker Desktop, it’s doing far more than just exposing a Docker socket. Behind the scenes, it’s performing some pretty invasive surgery on your namespace:

1. **Bind-mounting its own filesystem** into your distro’s mount namespace (you'll see dozens of mounts cluttering `/mnt/wsl/docker-desktop-bind-mounts/`).
2. **Injecting processes** into your distro’s cgroup hierarchy—including "ghost threads" that never clean up.
3. **Double-mounting** `/run/user`, which triggers a nasty race condition between WSL's init and systemd’s own startup sequence.

On systemd 255 (common baseline in LTS distros like Ubuntu), this mess mostly goes unnoticed because service manager follows a simpler execution path.

But on systemd 256+, the game changes.

Systemd's stricter handling of user sessions and cgroups turns those Docker Desktop quirks into a total dealbreaker.

## The Root Cause

I’ll be honest: I wouldn't have cracked this in 30 minutes on my own. 

Through some rapid-fire research with coding agents (shout out to Claude Code), I figured out that **systemd 256** introduced a significant architectural change: `systemd-executor`.

This is a dedicated binary responsible for sandboxing services before execution.

To place the executor into the correct cgroup, systemd now leverages the `clone3`  syscall:

```c
clone3({
    flags = CLONE_VM | CLONE_PIDFD | CLONE_VFORK | CLONE_INTO_CGROUP,
    cgroup = <fd>
})
```

The `CLONE_INTO_CGROUP` flag tells the kernel: 

“Spawn this process directly into this specific cgroup.”

Clean. Efficient. Correct.

Unless, of course, Docker Desktop has been “decorated” your cgroup tree.

Armed with that systemd insight, I traced the actual failure by running `strace` on PID 1:

```text
openat(AT_FDCWD, "/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service",
       O_RDONLY|O_CLOEXEC|O_PATH|O_DIRECTORY) = 57
clone3({..., CLONE_INTO_CGROUP, ..., cgroup=57}, 88) = -1 EBUSY (Device or resource busy)
```

The kernel returns `EBUSY` because cgroup v2 enforces a strict "no internal processes" rule: you cannot fork a process into a cgroup that has `subtree_control` enabled while it already contains processes in child cgroups. 

The smoking gun? Docker Desktop's integration leaves behind "ghost threads" — literal **PID 0** entries — inside those child cgroups (`init.scope`, `session.slice`, etc.), and they never go away.

```text
$ hexdump -C /sys/fs/cgroup/.../init.scope/cgroup.procs
00000000  30 0a 30 0a    |0.0.|
```

Two entries of PID 0. That is not normal. It’s not cleanable. And it’s enough to make `clone3(CLONE_INTO_CGROUP)` fail permanently.

The kicker? Systemd only falls back to a legacy retry without the flag if it receives `ENOSYS` or `EOPNOTSUPP`. `EBUSY` is not on that list. So, systemd simply gives up. Your user session never starts, and every `wsl -d Arch` attempt ends in a silent death.

## The Fix: Gently Evicting Docker Desktop

The solution is surprisingly simple and, in its architectural cleanliness, strictly better than the default setup. We want the Docker Desktop engine, but we don't want its "integration" mess.

### 1. Disable WSL Integration
Go to **Docker Desktop → Settings → Resources → WSL Integration** and toggle it **OFF** for your distro.

This immediately stops Docker Desktop from side-loading mounts, hijacking cgroups, and spawning those ghost processes into your environment.

### 2. Disable the Native Docker Daemon
If you're running Docker natively on your distro, you don't want 2 daemons fighting for the same resources:

```bash
sudo systemctl disable --now docker.service docker.socket containerd.service
```

### 3. Create a Socket Proxy
Docker Desktop still exposes its API to WSL via a Unix socket at `/mnt/wsl/docker-desktop/shared-sockets/guest-services/docker.proxy.sock`. However, it's owned by root with restrictive permissions.

We can bridge this gap with a simple `socat` proxy that runs as a systemd service, making the socket accessible to your user group:

```ini
# /etc/systemd/system/docker-desktop-proxy.service
[Unit]
Description=Docker Desktop Socket Proxy
ConditionPathExists=/mnt/wsl/docker-desktop/shared-sockets/guest-services/docker.proxy.sock
After=network.target

[Service]
Type=simple
ExecStartPre=/bin/rm -f /run/docker-desktop-proxy.sock
ExecStart=/usr/bin/socat \
    UNIX-LISTEN:/run/docker-desktop-proxy.sock,fork,mode=0660,group=docker \
    UNIX-CONNECT:/mnt/wsl/docker-desktop/shared-sockets/guest-services/docker.proxy.sock
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker-desktop-proxy.service
```

### 4. Point the Docker CLI at the Proxy
Finally, tell your Docker CLI to use our new, clean socket:

```bash
docker context create docker-desktop \
    --docker "host=unix:///run/docker-desktop-proxy.sock"
docker context use docker-desktop
```

Done. `docker images`, `docker run`, `docker compose` — everything works perfectly. Same images, same containers, zero `sudo`, and most importantly: **zero broken systemd sessions.**

## Why This Is Better

The default WSL Integration is a classic "easy but messy" tradeoff. It trades system isolation for a bit of convenience. By switching to a proxy-based setup, you reclaim control:

| Feature | WSL Integration ON | Socket Proxy (The Fix) |
| :--- | :---: | :---: |
| **Injected Mounts** | Dozens | **Zero** |
| **Cgroup Pollution** | Yes (PID 0 ghosts) | **No** |
| **Compatible with systemd 256+** | No (Broken) | **Yes** |
| **Shares Images/Containers** | Yes | Yes |
| **Requires `sudo`** | No | No (via socat group) |
| **Docker Desktop Resilient** | Yes | Yes (socat reconnects) |
| **Namespace Integrity** | Compromised | **Clean** |

You get the exact same Docker experience with none of the side effects. Your distro's mount namespace stays clean, your cgroup tree remains strictly yours, and systemd can finally do its job without tripping over Docker Desktop's baggage.

It’s a cleaner, more robust architecture that treats your Linux distro as a first-class citizen, not just a side-car for a Windows app.

## The Bigger Picture

This is a symptom of a broader problem: Docker Desktop treats WSL distros as extensions of itself rather than independent Linux environments. 

That works fine when your distro is an LTS release like Ubuntu with systemd 255 and you never look under the hood. It falls apart the moment you run anything that expects a clean, standards-compliant Linux system.

If you're on Arch, Fedora, or anything shipping systemd 256+, do yourself a favor: **cut the cord.** 

Set up the proxy. Reclaim your cgroups. Move on.

Your user session (and your future self) will thank you.
