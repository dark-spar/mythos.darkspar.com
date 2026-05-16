---
title: "Download"
description: "Mythos is in early development. Two install paths today: the published Docker image or build-from-source."
layout: "download"
---

Mythos has not cut a tagged release yet. Two install paths work today: pull the
rolling Docker image published from `main`, or build from source. Tagged
release binaries and platform packages will land when the project hits a real
`v0.1.0`.

## Docker (recommended)

The fastest way in. Multi-arch images (`linux/amd64` and `linux/arm64`) are
published to Docker Hub on every push to `main`. The image bundles
[`jellyfin-ffmpeg`](https://github.com/jellyfin/jellyfin-ffmpeg) so HW tonemap
filters (`tonemap_vaapi` / `tonemap_opencl` / `tonemap_cuda`) and the
SIMD-optimised CPU `tonemapx` kernel are all available, and runs the server
as PID 1 under `tini`.

```sh
docker run -d --name mythos \
  -p 8080:8080 \
  -v mythos-data:/data \
  -v /path/to/media:/media:ro \
  -e MYTHOS_TMDB_API_KEY=... \
  darkspar/mythos-server:edge
```

| Tag | What it points at |
| --- | --- |
| `:edge` | **Rolling** — rebuilt on every push to `main`. Cutting-edge, may break between pulls. |
| `:sha-<short>` | Immutable per-commit image. Use this in production-ish setups so an unattended `docker pull` doesn't surprise you. |
| `:1.2.3` / `:1.2` | Semver tags — published when a `v*` git tag is pushed. (None exist yet; Mythos is pre-1.0.) |

There is intentionally no `:latest` tag yet — by Docker convention `:latest`
means "newest stable release," and Mythos hasn't cut one. It will appear once
the first tagged release ships.

Bind-mount your media read-only. Everything Mythos writes (SQLite DB, JWT
secret, poster cache, transcode segments, sidecar subtitles) lives under
`/data`.

For VAAPI / QSV pass the render node through with `--device /dev/dri:/dev/dri`.
For NVENC use the NVIDIA container runtime (`--gpus all`). The server probes
available encoders at startup; override the choice with
`MYTHOS_HW_ENCODER=nvenc|qsv|vaapi|videotoolbox|cpu|auto`.

## Build from source

If you'd rather build it yourself — or you want to hack on it — the
`main` branch builds end-to-end.

### Prerequisites

A working toolchain for both the Rust server and the embedded SvelteKit UI, plus
`ffmpeg`/`ffprobe` on PATH for library scans and HLS transcoding.

- Rust 1.95+ (pinned in `rust-toolchain.toml`)
- Node 22+
- pnpm 10+
- ffmpeg / ffprobe (recent enough to expose the encoders you want — `h264_nvenc`,
  `h264_qsv`, `h264_vaapi`, `h264_videotoolbox`, or `libx264` as a fallback;
  `jellyfin-ffmpeg` is the easiest way to get HW tonemap filters and the
  `tonemapx` SIMD CPU kernel)

### Build and run

```sh
git clone https://gitlab.com/darkspar/mythos
cd mythos
cargo run --release --bin mythos-server
```

The build script in `crates/mythos-server` invokes `pnpm install && pnpm build`
in `web/` so the SPA is compiled and embedded in the same `cargo` invocation.
Set `MYTHOS_SKIP_WEB_BUILD=1` if you only want to rebuild the Rust side.

Then open <code>http://localhost:8080</code> (or `http://<lan-ip>:8080` from
another device — the default `listen` is `0.0.0.0:8080`). First-run setup
walks through
creating an admin account and adding a library.

## Hardware acceleration

Mythos probes `ffmpeg -encoders` at startup and smoke-tests each candidate
before committing to it. Priority order: NVENC → QSV → VAAPI → VideoToolbox →
libx264. Pin a specific encoder with `MYTHOS_HW_ENCODER=vaapi` (etc.), or force
CPU with `MYTHOS_HW_ENCODER=cpu`. The Docker image preconfigures
`MYTHOS_FFMPEG_BIN` / `MYTHOS_FFPROBE_BIN` to point at `jellyfin-ffmpeg`
already; on a bare-metal install set those if you'd rather not put
`jellyfin-ffmpeg` on PATH.

## Status

What works today: movies and TV — scan, browse, direct-play, HLS transcoding
with hardware acceleration (NVENC stays on the GPU end-to-end), multi-rendition
ABR, HDR→SDR tonemapping with a configurable filter pipeline (now including
`tonemapx`, jellyfin-ffmpeg's SIMD CPU kernel), subtitle burn-in and WebVTT
sidecars, sidecar `.srt` discovery, TMDb metadata enrichment with
backdrops, title search across movies + series, continue-watching across
movies and episodes, auto-play-next, and a `media-chrome`-based player.

What's next: the remaining Phase 3 media types — music, photos, books —
and a Jellyfin-API compatibility shim (Phase 6) for existing clients like
Findroid and Swiftfin. See the [roadmap on the homepage](/#status).
