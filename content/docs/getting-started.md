---
title: "Getting Started"
description: "Clone, build, and serve your first movie."
weight: 10
---

This is the path that works today. There is no released binary, no Docker
image, no package manager entry — Mythos is built from source on `main`.

## 1. Install prerequisites

- **Rust 1.95+** — `rustup` will pick it up from `rust-toolchain.toml`.
- **Node 22+** and **pnpm 10+** — the SvelteKit UI is built and embedded.
- **ffmpeg / ffprobe** on PATH — used by the scanner and the HLS transcoder.

## 2. Clone and run

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

## 3. First-run setup

Visit <code>http://localhost:8080</code>. You'll be walked through:

1. Creating the first administrator account.
2. (Optional) Setting your TMDb API key, so scans enrich titles and posters.
   Without one, scans still index files — they just won't have titles or
   art beyond what's in the filename.
3. Adding a library — point it at a directory of movies on disk. The scan
   starts immediately.

## 4. Watch something

Pick any movie. Mythos serves the file directly (HTTP byte-range) if your
browser can decode it, and falls back to an on-the-fly HLS transcode if it
can't. Hardware acceleration is picked automatically at startup if it's
available.

## What's next

- [Configuration](../configuration/) — the `mythos.toml` keys and `MYTHOS_*`
  env vars that actually exist.
- [Library layout](../library-layout/) — how the scanner reads filenames
  today, and what's still scheduled.
- [Architecture](../architecture/) — the workspace, the runtime, and the
  shape of the streaming pipeline.
