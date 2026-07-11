# Self-Hosted AI Image Upscaler (Upscayl / Real-ESRGAN)

A self-hosted image upscaling service running on Proxmox, wrapped in a small
Flask web app so upscaling can be triggered from any device on the LAN
instead of running Upscayl locally on a laptop.

## Overview

- **Host:** Proxmox (tested on an LXC container, unprivileged, Debian 12)
- **GPU:** any Vulkan-capable GPU (tested on an AMD Stoney Ridge iGPU),
  passed through from the Proxmox host into the container
- **Engine:** `realesrgan-ncnn-vulkan` — the same NCNN + Vulkan engine the
  official Upscayl desktop app wraps
- **Interface:** Flask web app — upload an image, get the upscaled result
  back

## Architecture

```
Client (any device on the LAN)
        │  HTTP (upload / download)
        ▼
Proxmox host
  └── GPU (/dev/dri/renderD128, /dev/dri/card0)
        │  device passthrough (gid-mapped, unprivileged LXC)
        ▼
  LXC container
        ├── Vulkan driver — GPU compute API
        ├── realesrgan-ncnn-vulkan — runs the AI model
        ├── model files (.bin/.param weights)
        └── Flask app (app.py) — web front end, calls the binary via subprocess
```

## Setup

### 1. GPU passthrough (host → container)

On the Proxmox host, confirm the render node exists:
```bash
ls -l /dev/dri
```

Get the host GIDs for the `render` and `video` groups:
```bash
getent group render
getent group video
```

Add device passthrough to the container config — see
[`lxc/gpu-passthrough.conf`](lxc/gpu-passthrough.conf) for the exact lines to
append to `/etc/pve/lxc/<CTID>.conf`.

Restart the container (a full stop/start, not a reboot from inside) for the
config to take effect:
```bash
pct stop <CTID>
pct start <CTID>
```

### 2. Vulkan + drivers (inside the container)

```bash
apt update
apt install -y mesa-vulkan-drivers vulkan-tools mesa-va-drivers vainfo
vulkaninfo --summary
```
Confirms the physical GPU shows up as a `VkPhysicalDevice` (not just the
`llvmpipe` CPU fallback).

### 3. Real-ESRGAN NCNN binary

```bash
apt install -y wget unzip libgomp1
wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.5.0/realesrgan-ncnn-vulkan-20220424-ubuntu.zip
unzip realesrgan-ncnn-vulkan-20220424-ubuntu.zip -d /opt/realesrgan
```

### 4. Flask web app

```bash
apt install -y python3 python3-venv python3-pip
python3 -m venv /opt/upscayl-web
source /opt/upscayl-web/bin/activate
pip install -r app/requirements.txt
mkdir -p /opt/upscayl-web/{uploads,outputs}
```

Copy [`app/app.py`](app/app.py) to `/opt/upscayl-web/app.py`, then run:
```bash
python3 /opt/upscayl-web/app.py
```

Access from any device on the LAN:
```
http://<container-ip>:5000
```

## Known issue: GPU job timeout on weak/old GPUs

Running a full-size image through in one shot can produce:
```
radv/amdgpu: The CS has been cancelled because the context is lost.
vkQueueSubmit failed -4
```

Cause: the Linux kernel's `amdgpu.lockup_timeout` (default ~2 seconds) kills
GPU jobs it thinks have hung. Weak/old GPUs can take longer than that on a
full-size tile, so the kernel kills the job even though the GPU wasn't
actually stuck.

**Fix:** reduce the tile size (`-t` flag) so each GPU submission is smaller
and finishes well under the timeout. `-t 32` is a safe starting point —
slower overall, but no crashes. Already set in `app/app.py`.

## Available models

Whatever ships in `/opt/realesrgan/models` — check with:
```bash
ls /opt/realesrgan/models
```
Common ones: `realesrgan-x4plus`, `realesrgan-x4plus-anime`,
`realesr-animevideov3-x2/x3/x4`.

## Roadmap

- [ ] Let the client choose which model to use via a dropdown
- [ ] Inline before/after preview instead of a forced download
- [ ] Explicit "Download" / "Discard" actions so the client controls
      whether the result is kept
- [ ] Live progress percentage (background job + `/status/<job_id>`
      polling endpoint)

## Tech stack

Proxmox VE · LXC (unprivileged) · Debian 12 · Vulkan (Mesa RADV) ·
Real-ESRGAN NCNN Vulkan · Python 3 · Flask
