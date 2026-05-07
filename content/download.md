---
title: "Download"
description: "Get Mythos running on your home server in a few minutes."
layout: "download"
---

Pick the path that fits your setup. Mythos ships as a single static binary, a Docker image, or source you can build yourself.

## Docker

The fastest way to try Mythos. Mounts your media directory read-only and exposes the web UI on port 7878.

```sh
docker run -d \
  --name mythos \
  -p 7878:7878 \
  -v /path/to/media:/media:ro \
  -v mythos-data:/var/lib/mythos \
  ghcr.io/mattmoore/mythos:latest
```

Then open <code>http://localhost:7878</code> and walk through the first-run setup.

## Pre-built binary

Static, single-file binaries for Linux, macOS, and Windows. No runtime, no dependencies.

```sh
curl -sSL https://mythos.app/install.sh | sh
mythos --config /etc/mythos/config.toml
```

Binaries are signed and reproducible. Verify the checksum from the release page.

## Build from source

You'll need Rust 1.95 or newer.

```sh
git clone https://github.com/mattmoore/mythos
cd mythos
cargo build --release
./target/release/mythos-server
```

## Platform packages

Native packages for common Linux distributions and homebrew. Pick yours below.

- **Arch Linux** — <code>yay -S mythos</code>
- **Debian / Ubuntu** — `.deb` on the [releases page](https://github.com/mattmoore/mythos/releases)
- **macOS** — <code>brew install mattmoore/tap/mythos</code>
- **NixOS** — flake on the repo, module included

## Hardware

Mythos is light. A Raspberry Pi 5 streams 1080p direct-play to the whole household; an old N100 mini-PC will transcode 4K on the fly. The server runs in well under 100 MB of RAM at idle.
