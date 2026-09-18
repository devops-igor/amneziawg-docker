# AmneziaWG 3.1 Docker Client

[![docker](https://github.com/devops-igor/amneziawg-docker/actions/workflows/docker.yml/badge.svg)](https://github.com/devops-igor/amneziawg-docker/actions/workflows/docker.yml)
[![Docker Pulls](https://img.shields.io/docker/pulls/devopsigor/amneziawg)](https://hub.docker.com/r/devopsigor/amneziawg)
[![License](https://img.shields.io/github/license/devops-igor/amneziawg-docker)](LICENSE)
[![Docker Image Size](https://img.shields.io/docker/image-size/devopsigor/amneziawg/latest)](https://hub.docker.com/r/devopsigor/amneziawg)
[![Platform](https://img.shields.io/badge/platform-linux%2Farm64%20%7C%20linux%2Famd64-blue)]()

> Lightweight Docker image for running AmneziaWG 3.1 VPN on ARM64 and x86_64 devices

> Image home moved to [devopsigor/amneziawg](https://hub.docker.com/r/devopsigor/amneziawg) (multi-arch). The old devopsigor/awg2-arm64 image is frozen at its last arm64-only version.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Pull](#pull)
- [Volume Mount](#volume-mount)
- [Required Capabilities & Sysctls](#required-capabilities--sysctls)
- [Healthcheck](#healthcheck)
- [Example Config File](#example-config-file)
- [Docker Compose](#docker-compose)
- [Build from Source](#build-from-source)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

A minimal Docker image that runs [AmneziaWG 3.1](https://github.com/amnezia-vpn/amneziawg-go) client using the userspace `amneziawg-go` implementation. No kernel module required - works on Raspberry Pi, ARM servers, x86 servers, NAS devices, and anywhere Docker runs.

---

## Quick Start

```bash
# Run
docker run \
  --name amneziawg-client \
  --cap-add NET_ADMIN \
  --device /dev/net/tun:/dev/net/tun \
  --sysctl net.ipv4.ip_forward=1 \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  -v /path/to/your/amneziawg.conf:/config/amneziawg.conf:ro \
  devopsigor/amneziawg:latest
```

---

## Pull

```bash
docker pull devopsigor/amneziawg:latest            # native arch of your host
docker pull --platform linux/amd64 devopsigor/amneziawg:latest
docker pull --platform linux/arm64 devopsigor/amneziawg:latest
```

Versioned tags are published automatically on every `v*` git tag (e.g. `v3.1.20260828-1`).

### Platform support

- **linux/arm64** — primary target, fully tested (live tunnel verification with header protection)
- **linux/amd64** — built by the same CI pipeline from identical source; verified at artifact level (correct platform manifest, x86-64 binaries). Recommended: run a quick smoke on your x86 host before production use.

---

## Volume Mount

The config file **must** be mounted read-only at `/config/amneziawg.conf`:

```bash
-v /path/to/config:/config/amneziawg.conf:ro
```

The entrypoint validates:
- File exists
- Contains `[Interface]` section
- Contains at least one `[Peer]` section

### AmneziaWG 3.1 parameters

Header protection (`HeaderProtectionKey`) is the headline 3.1 feature. The key
MUST be identical on client and server, and S1-S4 must all be >= 12 when
header protection is enabled. A key mismatch fails silently - the tunnel never
comes up and no error is printed. See the fully documented example config.

---

## Required Capabilities & Sysctls

```bash
docker run \
  --cap-add NET_ADMIN          # Required to create TUN interface
  --device /dev/net/tun:/dev/net/tun  # TUN device access
  --sysctl net.ipv4.ip_forward=1   # Enable packet forwarding
  --sysctl net.ipv4.conf.all.src_valid_mark=1  # Required for WireGuard/AmneziaWG
  ...
```

> **Note:** On some systems (Synology NAS, OpenWrt), you may also need `--privileged`. Try that if you get permission errors.

---

## Healthcheck

The container has a built-in healthcheck:

```bash
# Check status
docker inspect --format='{{.State.Health.Status}}' amneziawg-client

# Verify interface is up inside container
docker exec amneziawg-client awg show
```

Healthcheck runs `awg show` every 30s. If it fails 3 times, the container is marked **unhealthy**.

---

## Example Config File

See [`config/amneziawg.conf.example`](config/amneziawg.conf.example) for a fully documented example.

---

## Docker Compose

```yaml
services:
  amneziawg:
    image: devopsigor/amneziawg:latest 
    container_name: amneziawg-client
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    volumes:
      - /path/to/amneziawg.conf:/config/amneziawg.conf:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "awg", "show"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Build from Source

GitHub Actions builds both architectures on native runners and publishes to Docker Hub automatically on every push to `main` (and on `v*` tags). Local builds are only needed for custom patches or testing.

### Prerequisites

- Docker 20.10+ with buildx

### Build

```bash
# Clone the repo
git clone https://github.com/devops-igor/amneziawg-docker.git
cd amneziawg-docker

# Build for your current architecture
docker build -t amneziawg-client:local .

# Or build for a specific platform explicitly
docker buildx build --platform linux/amd64 -t amneziawg-client:local .
docker buildx build --platform linux/arm64 -t amneziawg-client:local .

# Override version pins (defaults: v3.1.20260828, v3.1.20260812)
docker buildx build \
  --build-arg AWG_GO_VERSION=v3.1.20260828 \
  --build-arg AWG_TOOLS_VERSION=v3.1.20260812 \
  -t amneziawg-client:local .
```

### Local multi-arch build (optional)

CI handles multi-arch automatically; to reproduce it locally:

```bash
# One-time: enable cross-arch emulation and a container-driver builder
docker run --privileged --rm tonistiigi/binfmt --install amd64
docker buildx create --name multiarch --driver docker-container --use

# Build and push both architectures as a single manifest
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <your-dockerhub-user>/<your-repo>:tag --push .
```

Pushing requires registry authentication (`docker login`). CI does this with repository secrets.

### Verify Build

```bash
# Check image size (should be <100MB)
docker images amneziawg-client:local

# Test it starts (with a valid config)
docker run --rm \
  --cap-add NET_ADMIN \
  --device /dev/net/tun:/dev/net/tun \
  --sysctl net.ipv4.ip_forward=1 \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  -v /path/to/test.conf:/config/amneziawg.conf:ro \
  amneziawg-client:local

# Check healthcheck
docker inspect --format='{{.State.Health.Status}}' <container_id>
```

Expected image size: **< 100MB** (target: < 50MB, optimized for a minimal footprint)

---

## Troubleshooting

### "Device or resource busy" when accessing `/dev/net/tun`

The TUN device is already in use or not available. Try:
```bash
# Check if tun module is loaded
lsmod | grep tun

# Load it manually
sudo modprobe tun
```

### "RTNETLINK: Operation not permitted"

Missing `NET_ADMIN` capability or running in an unprivileged environment (some NAS, WSL2):
```bash
# Try privileged mode (less secure)
docker run --privileged ...
```

### Healthcheck shows "unhealthy" immediately

1. Verify your config file is valid
2. Check logs: `docker logs <container>`
3. Run interactively to see errors:
   ```bash
   docker run --rm -it \
     --cap-add NET_ADMIN \
     --device /dev/net/tun:/dev/net/tun \
     --sysctl net.ipv4.ip_forward=1 \
     --sysctl net.ipv4.conf.all.src_valid_mark=1 \
     -v /path/to/config:/config/amneziawg.conf:ro \
     amneziawg-client:local /bin/sh
   ```

### Container exits immediately with code 0

The config may be invalid or the tunnel failed to start. Check logs:
```bash
docker logs <container>
```

---

## Security Notes

- No hardcoded secrets or keys in any file
- Config file should be mounted read-only (`:ro`)
- Container starts as root (required for networking) and drops to non-root user `amneziawg:1000` for the long-running process
- The `amneziawg` user has passwordless sudo restricted to networking commands (`ip`, `iptables`, `awg`, `awg-quick`) - this is defense-in-depth, not a true sandbox, since the container already holds `NET_ADMIN`
- Review Docker's capability requirements before running in production

---

## License

MIT - See repository for details.
