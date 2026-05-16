---
title: "Configuration"
description: "The mythos.toml keys and MYTHOS_* env vars that actually exist."
weight: 20
---

Mythos loads configuration via [`figment`](https://github.com/SergioBenitez/Figment),
merging three sources in this order — later sources win:

1. Built-in defaults.
2. A TOML file at `MYTHOS_CONFIG`, or `./mythos.toml` if present.
3. `MYTHOS_*` environment variables.

## A minimal config

```toml
listen     = "0.0.0.0:8080"
data_dir   = "/var/lib/mythos"
log_filter = "info,mythos=debug,sqlx=warn"
```

## Keys

All keys live at the top level — there are no sections.

| Key | Default | Description |
|---|---|---|
| `listen` | `0.0.0.0:8080` | Socket address the HTTP server binds to. Defaults to all interfaces so the server is reachable from other devices on the LAN out of the box; bind to `127.0.0.1:8080` if you'd rather keep it localhost-only. |
| `data_dir` | `./data` | Where Mythos stores the SQLite DB, posters, transcode segments, subtitles, and the JWT secret. |
| `log_filter` | `info,mythos=debug,sqlx=warn` | `tracing-subscriber` env-filter directive. |
| `cookie_secure` | `true` in release, `false` in debug | Sets the `Secure` flag on auth cookies. Override to `false` if you terminate TLS upstream. |
| `token_ttl_days` | `30` | Lifetime of issued JWTs, in days. |
| `tmdb_api_key` | _(none)_ | TMDb v3 API key. Without one, metadata enrichment is disabled — scans still index files. Also settable from the admin UI at runtime; `MYTHOS_TMDB_API_KEY` wins over the admin-UI value. Saves swap the live `TmdbHandle`, so a new key takes effect on the next scan without a restart. |

## Environment variables

Any TOML key has a `MYTHOS_*` upper-snake-case env-var equivalent that takes
precedence over the file. The non-TOML env vars are:

| Var | Description |
|---|---|
| `MYTHOS_CONFIG` | Path to the TOML file, if not `./mythos.toml`. |
| `MYTHOS_JWT_SECRET` | Base64-encoded JWT signing key, ≥32 bytes. If unset, Mythos generates one and persists it to `{data_dir}/jwt.secret`. |
| `MYTHOS_HW_ENCODER` | One of `auto` (default), `cpu`, `nvenc`, `qsv`, `vaapi`, `videotoolbox`. Pins a specific encoder; `auto` smoke-tests in priority order. |
| `MYTHOS_FFMPEG_BIN` | Path or name of the `ffmpeg` binary to invoke (default: `ffmpeg` on PATH). Useful for pinning a custom build with the encoders or tonemap filters you need — e.g. `jellyfin-ffmpeg`, which ships HW tonemap filters (`tonemap_vaapi` / `tonemap_opencl`) that distro builds often omit. |
| `MYTHOS_FFPROBE_BIN` | Path or name of the `ffprobe` binary to invoke (default: `ffprobe` on PATH). |
| `MYTHOS_TMDB_API_KEY` | Same as `tmdb_api_key` in the TOML file. |
| `MYTHOS_SKIP_WEB_BUILD` | Build-time only: skips `pnpm build` so `cargo` doesn't rebuild the SPA. |

## Runtime settings (admin UI)

A handful of settings live in the `settings` table and are edited from
the admin UI rather than the TOML file — they take effect on the next
scan or transcode without a restart:

| Setting | Description |
|---|---|
| TMDb API key | Same value as `tmdb_api_key` / `MYTHOS_TMDB_API_KEY`. The env var wins if set. A save here swaps the live `TmdbHandle` so new keys apply on the next scan without a restart. |
| Tonemap pipeline | Which filter graph to apply for HDR→SDR: `software`, `tonemapx`, `vaapi`, `opencl`, or `cuda`. Pipelines whose ffmpeg filter isn't compiled in fall back to `software`. `tonemapx` is jellyfin-ffmpeg's SIMD-optimised CPU kernel — much faster than the stock `tonemap` filter on the CPU path, but only available when ffmpeg is jellyfin-ffmpeg (it ships in the Docker image; set `MYTHOS_FFMPEG_BIN=/usr/lib/jellyfin-ffmpeg/ffmpeg` on a bare-metal install). |
| Tonemap algorithm | `hable` (default), `mobius`, `reinhard`, or `bt2390`. Honored by the software / Tonemapx / OpenCL / CUDA pipelines; the VAAPI pipeline ignores it (the filter doesn't expose an algorithm knob). |

## Where state lives

`data_dir` ends up holding:

- `mythos.db` — SQLite, schema managed by `sqlx::migrate!`.
- `posters/` — proxied TMDb art so clients never hit TMDb directly.
- `transcode/` — HLS segments during active sessions; torn down when the
  player goes away.
- `subtitles/` — extracted text subs (WebVTT) and burn-in artifacts.
- `jwt.secret` — auto-generated 32-byte signing key, atomic-written on first
  boot.

There is no separate database section — SQLite is the only supported backend.

## Hardware acceleration

Mythos probes `ffmpeg -encoders` at startup and smoke-tests each candidate.
Priority order is NVENC → QSV → VAAPI → VideoToolbox → libx264. Pin a
specific encoder with `MYTHOS_HW_ENCODER=vaapi` (etc.), or force CPU with
`MYTHOS_HW_ENCODER=cpu`.

For container deployments:

```sh
# VAAPI / QSV — pass the render node through
docker run ... --device /dev/dri:/dev/dri darkspar/mythos-server:edge

# NVENC — NVIDIA container runtime
docker run ... --gpus all darkspar/mythos-server:edge
```

## Docker image defaults

The published image (`darkspar/mythos-server:edge`) sets a few env vars so
the container does the right thing out of the box:

| Variable | Default in the image | Note |
|---|---|---|
| `MYTHOS_LISTEN` | `0.0.0.0:8080` | |
| `MYTHOS_DATA_DIR` | `/data` | The single writable volume — bind-mount it. |
| `MYTHOS_FFMPEG_BIN` | `/usr/lib/jellyfin-ffmpeg/ffmpeg` | The image ships `jellyfin-ffmpeg7`. |
| `MYTHOS_FFPROBE_BIN` | `/usr/lib/jellyfin-ffmpeg/ffprobe` | Same. |

On a bare-metal install set `MYTHOS_FFMPEG_BIN` / `MYTHOS_FFPROBE_BIN`
yourself if you'd rather not put `jellyfin-ffmpeg` on PATH — the HW tonemap
filters (`tonemap_vaapi` / `tonemap_opencl` / `tonemap_cuda`) and the
`tonemapx` SIMD CPU kernel are jellyfin-ffmpeg-only on most distros.
