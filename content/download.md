---
title: "Download"
description: "Mythos is in early development. The only supported install path right now is build-from-source."
layout: "download"
---

Mythos has not cut a tagged release yet. The repository builds end-to-end on the
`main` branch — that's the path described below. Pre-built binaries, container
images, and platform packages will land when the project hits a real `v0.1.0`.

## Prerequisites

A working toolchain for both the Rust server and the embedded SvelteKit UI, plus
`ffmpeg`/`ffprobe` on PATH for library scans and HLS transcoding.

- Rust 1.95+ (pinned in `rust-toolchain.toml`)
- Node 22+
- pnpm 10+
- ffmpeg / ffprobe (recent enough to expose the encoders you want — `h264_nvenc`,
  `h264_qsv`, `h264_vaapi`, `h264_videotoolbox`, or `libx264` as a fallback)

## Build and run

```sh
git clone https://gitlab.com/darkspar/mythos
cd mythos
cargo run --release --bin mythos-server
```

The build script in `crates/mythos-server` invokes `pnpm install && pnpm build`
in `web/` so the SPA is compiled and embedded in the same `cargo` invocation.
Set `MYTHOS_SKIP_WEB_BUILD=1` if you only want to rebuild the Rust side.

Then open <code>http://127.0.0.1:8080</code>. First-run setup walks through
creating an admin account and adding a library.

## Hardware acceleration

Mythos probes `ffmpeg -encoders` at startup and smoke-tests each candidate
before committing to it. Priority order: NVENC → QSV → VAAPI → VideoToolbox →
libx264. Pin a specific encoder with `MYTHOS_HW_ENCODER=vaapi` (etc.), or force
CPU with `MYTHOS_HW_ENCODER=cpu`.

## Status

What works today: movies — scan, browse, direct-play, HLS transcoding with
hardware acceleration, multi-rendition ABR, subtitle burn-in and WebVTT
sidecars, TMDb metadata enrichment.

What's next: TV, music, photos, books (Phase 3), and a Jellyfin-API
compatibility shim (Phase 6) for existing clients like Findroid and Swiftfin.
See the [roadmap on the homepage](/#status).
