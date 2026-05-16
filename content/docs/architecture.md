---
title: "Architecture"
description: "The workspace, the runtime, and the shape of the streaming pipeline."
weight: 40
---

Mythos is a Cargo workspace. Each crate has one responsibility; the
`mythos-server` binary wires them together with `axum`.

## The crates

| Crate | Responsibility |
|---|---|
| `mythos-server` | Main binary. Loads config, runs migrations, builds the `axum` app, embeds and serves the SvelteKit SPA. |
| `mythos-core` | Shared domain types (`MediaItem`, `MediaKind`, …). |
| `mythos-db` | SQLite pool + `sqlx::migrate!` runner. Re-exports `SqlitePool`. |
| `mythos-auth` | argon2id password hashing, HS256 JWT issuance, `AuthUser` / `AdminUser` extractors. Errors deliberately don't implement `IntoResponse` — translation to HTTP lives in `mythos-api` so the crate stays usable from non-HTTP contexts. |
| `mythos-api` | `axum` routers and handlers: `auth`, `library`, `movie`, `series`, `episode`, `scan`, `play`, `hls`, `subtitles`, `settings`, `search`. |
| `mythos-scan` | Filesystem walker (`jwalk`) and `ffprobe` driver. Movie and TV branches share the walk; the TV branch parses `SxxEyy` / `1x01` with a season-dir fallback. |
| `mythos-meta` | TMDb client with an on-disk poster cache. Handles movies, TV series, seasons, and episode stills. MusicBrainz / OpenLibrary land later in Phase 3. |
| `mythos-stream` | Direct-play byte-range responses, FFmpeg HLS transcoder, ABR ladder, hardware-encoder probe, subtitle burn-in, HDR→SDR tonemapping pipeline. Movies and episodes share the streaming surface via a `SessionKey { user_id, item_id, kind }` so their transcode sessions can't collide. |

Workspace dependencies live at the root `Cargo.toml` under
`[workspace.dependencies]`; member crates pull them with `dep.workspace = true`.

## The runtime

A single Tokio runtime hosts everything. The HTTP server, the scanner workers,
and any active transcoding sessions share the same scheduler.

The SPA gets into the binary at build time: `crates/mythos-server/build.rs`
runs `pnpm install && pnpm build` in `web/`, producing `web/build/`.
`rust-embed` bakes that directory in. The fallback handler in
`crates/mythos-server/src/web.rs` serves embedded assets for known paths and
falls back to `index.html` for everything else, so client-side routing works.

In debug builds, `rust-embed` reads from disk at runtime — SPA changes show
up after `pnpm build` without a Rust rebuild.

## State

SQLite is the only backend. The schema is migration-managed (`migrations/`)
and small enough to inspect by hand. Posters, transcode segments, and
extracted subtitles live on disk under `data_dir`.

The current schema (migrations 0001–0012):

- `users` — argon2id hashes, `token_version` for forced logout.
- `libraries` — root paths the scanner walks.
- `media_files` — one row per file on disk, with `ffprobe` columns and
  `color_primaries` / `color_transfer` / `color_space` for HDR detection
  (migration 0009).
- `media_backdrops` — per-item backdrop images proxied from TMDb so the
  UI's Plex-style ambient gradient and featured-backdrop hero never hit
  TMDb directly (migration 0010).
- `media_file_keyframes` — per-file keyframe index so remux-mode HLS aligns
  its segment boundaries to real IDR frames instead of best-guess
  timestamps (migration 0011).
- `movies` — one row per identified movie, pointing at a `media_files` row.
- `series` → `seasons` → `episodes` — TV identity. Each `episodes` row FKs
  1:1 to a `media_files` row, mirroring how `movies` does, so subtitles,
  byte-range streaming, and HLS transcoding work for episodes without
  any branch in the streaming code.
- `movie_progress` / `episode_progress` — debounced watch position per
  user, per kind.
- `media_subtitles` — extracted text subs, image-sub render artifacts, and
  discovered `.srt` sidecars (a `sidecar` flag on the row marks the last
  group; added in migration 0012).
- `settings` — runtime-configurable settings (TMDb key, tonemap pipeline
  + algorithm, etc.).

Tables for tracks, albums, artists, photos, and books **don't exist
yet** — they ship later in Phase 3 alongside their scanners.

IDs are UUID v7, stored as `TEXT`. Timestamps are ISO-8601 UTC strings.

## Auth

Passwords are hashed with argon2id. On login the server issues an HS256 JWT,
which it delivers two ways:

- **Web clients** get a `SameSite=Lax` HttpOnly cookie. The cookie's `Secure`
  flag follows the `cookie_secure` config knob (defaults to `true` in
  release builds, `false` in debug).
- **API clients** can present the same token as `Authorization: Bearer …`.

The signing key resolves in this order: `MYTHOS_JWT_SECRET` (base64,
≥32 bytes) → `{data_dir}/jwt.secret` → a freshly generated 32-byte key,
atomically written to disk. Tokens carry a per-user `token_version` so
revoking sessions is one bump.

## The streaming pipeline

Two paths converge at the player:

- **Direct play.** `GET /api/movies/:id/stream` returns the file with HTTP
  byte-range support. The web client wraps a plain `<video>` element in
  [`media-chrome`](https://www.media-chrome.org/) for the player UI; HLS
  is fed by [`hls.js`](https://github.com/video-dev/hls.js) when the
  transcode path kicks in. Watch progress is debounced and persisted
  server-side so resume works across devices.
- **HLS transcode.** If the client's declared profile says the file is
  unplayable, Mythos spawns an `ffmpeg` session via the `TranscodeManager`,
  produces segmented HLS, and serves segments on demand. The player calls
  `DELETE /api/movies/:id/hls` on teardown so `ffmpeg` subprocesses don't
  leak — every new transcode entry point routes through `TranscodeManager`
  to keep the lifecycle centralized.

The HLS pipeline supports multi-rendition ABR. Hardware encoders are probed
and smoke-tested at startup — a build with NVENC compiled in but no working
driver falls back cleanly. Priority order is NVENC → QSV → VAAPI →
VideoToolbox → libx264. The NVENC path stays on the GPU end-to-end via
`NVDEC` + `scale_cuda` so the frame never round-trips through system RAM.

### HDR → SDR tonemapping

For HDR sources, Mythos applies an explicit tonemap filter in the
transcode graph before encoding. The choice of *filter pipeline*
(software / Tonemapx / VAAPI / OpenCL / CUDA) and *algorithm*
(Hable / Mobius / Reinhard / BT.2390) is admin-configurable from the
settings UI and persisted in the `settings` table. Mythos probes
`ffmpeg` at startup for the tonemap filters it actually has compiled
in, and pipelines whose filter isn't present silently fall back to
software so a missing build feature can't break playback.

`Tonemapx` is jellyfin-ffmpeg's SIMD-optimised CPU tonemap kernel —
much faster than the stock `tonemap` filter on the CPU path, but only
available when ffmpeg is jellyfin-ffmpeg. The Docker image already
points `MYTHOS_FFMPEG_BIN` / `MYTHOS_FFPROBE_BIN` at it; on bare-metal
installs set those env vars to enable the option.

Source HDR detection uses the `color_primaries` / `color_transfer` /
`color_space` columns on `media_files`; if those are still `NULL` (a
library scanned before migration 0009), the first HDR play self-heals
them by ffprobing on demand.

### Title search

`GET /api/search?q=…` returns a flat, ranked list of movies and series
that match the query. The current implementation is a case-insensitive
`LIKE` against `sort_title` via `mythos_db::SearchRepo` — it'll graduate
to SQLite FTS5 once libraries get big enough to chug. The web client
binds the endpoint to a single search box with keyboard navigation
(<kbd>↑</kbd>/<kbd>↓</kbd> walks results, <kbd>Enter</kbd> opens).

## Why Rust

Three reasons:

1. **Single binary.** `cargo build --release` produces one statically linked
   executable. The SPA is baked in. There is no runtime to install, no
   `node_modules` to ship.
2. **Predictable performance.** No GC pauses while you're seeking. The
   transcode supervisor can hold its frame budget without surprises.
3. **Memory honesty.** Long-running media servers accumulate small leaks.
   Lifetimes and `Drop` make those visible at compile time.
