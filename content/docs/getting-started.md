---
title: "Getting Started"
description: "Clone, build, and serve your first movie."
weight: 10
---

Two install paths work today: pull the rolling Docker image, or build from
source. Both run against `main` — there's no tagged release binary or package
manager entry yet.

## Docker (quickest)

```sh
docker run -d --name mythos \
  -p 8080:8080 \
  -v mythos-data:/data \
  -v /path/to/media:/media:ro \
  darkspar/mythos-server:edge
```

The image bundles `jellyfin-ffmpeg` (so HW tonemap filters and the
SIMD-CPU `tonemapx` kernel are available) and is published for `linux/amd64`
and `linux/arm64`. Bind-mount your media read-only; everything Mythos writes
lives under `/data`. For hardware acceleration, see the
[Configuration page](../configuration/#hardware-acceleration).

Once it's up, open <code>http://localhost:8080</code> and jump to
[First-run setup](#first-run-setup) below.

## Build from source

If you'd rather build it yourself — or you want to hack on it — `main`
builds end-to-end. The numbered steps below are the from-source path.

### 1. Install prerequisites

- **Rust 1.95+** — `rustup` will pick it up from `rust-toolchain.toml`.
- **Node 22+** and **pnpm 10+** — the SvelteKit UI is built and embedded.
- **ffmpeg / ffprobe** on PATH — used by the scanner and the HLS transcoder.

### 2. Clone and run

```sh
git clone https://gitlab.com/darkspar/mythos
cd mythos
cargo run --release --bin mythos-server
```

The first build is slow — Cargo compiles the workspace and the `build.rs` in
`mythos-server` runs `pnpm install && pnpm build` to produce `web/build/`,
which `rust-embed` bakes into the binary.

When it's up, the log line you're looking for is:

```
INFO mythos: ready on http://0.0.0.0:8080
```

The server binds to all interfaces by default, so it's reachable from any
device on the LAN. To restrict it to localhost, set
`MYTHOS_LISTEN=127.0.0.1:8080` (or put `listen = "127.0.0.1:8080"` in
`mythos.toml`).

## First-run setup

Visit <code>http://localhost:8080</code>. A 3-step setup wizard walks you
through bringing the server online:

1. **Admin account.** Username + password, hashed with argon2id.
2. **TMDb API key.** Optional — scans still index files without one,
   they just won't have titles or art beyond what's in the filename.
   You can drop the key in later from the admin settings page; the
   live `TmdbHandle` swaps on save, so a new key takes effect on the
   next scan without a restart.
3. **First library.** Point it at a directory of movies or TV on disk;
   the scan starts immediately. The wizard advances in-page via local
   state, so the layout's "redirect to login once an admin exists"
   rule doesn't fight the flow mid-setup.

## Watch something

Pick any movie. Mythos serves the file directly (HTTP byte-range) if your
browser can decode it, and falls back to an on-the-fly HLS transcode if it
can't. Hardware acceleration is picked automatically at startup if it's
available.

## Want a native client?

The repo also ships a Tauri 2 desktop app at `apps/mythos-desktop/`
sharing the exact same SvelteKit UI as the embedded SPA, but switching
playback to libmpv via IPC at runtime — HEVC, AV1, and HDR files play
natively without the browser's codec constraints. See the
[Download page](../../download/#native-desktop-client-early) for the
build commands and current limitations. It's an early scaffold, not
yet a polished install path.

## What's next

- [Configuration](../configuration/) — the `mythos.toml` keys and `MYTHOS_*`
  env vars that actually exist.
- [Library layout](../library-layout/) — how the scanner reads filenames
  today, and what's still scheduled.
- [Architecture](../architecture/) — the workspace, the runtime, the
  shape of the streaming pipeline, and how the persistent mini-bar
  player is wired up.
