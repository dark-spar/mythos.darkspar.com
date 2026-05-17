---
title: "Architecture"
description: "The workspace, the runtime, and the shape of the streaming pipeline."
weight: 40
---

Mythos is a Cargo workspace. Each crate has one responsibility; the
`mythos-server` binary wires them together with `axum`. The repo also
hosts a Tauri 2 desktop client under `apps/` that reuses the SvelteKit
UI as a sibling workspace member.

## The crates

| Crate | Responsibility |
|---|---|
| `mythos-server` | Main binary. Loads config, runs migrations, builds the `axum` app, embeds and serves the SvelteKit SPA. |
| `mythos-core` | Shared domain types (`MediaItem`, `MediaKind`, …) plus the `playback::{PlaybackRequest, PlaybackDecision}` wire contract that the desktop client (and the future Jellyfin shim) speak. |
| `mythos-db` | SQLite pool + `sqlx::migrate!` runner. Re-exports `SqlitePool`. |
| `mythos-auth` | argon2id password hashing, HS256 JWT issuance, `AuthUser` / `AdminUser` extractors. Errors deliberately don't implement `IntoResponse` — translation to HTTP lives in `mythos-api` so the crate stays usable from non-HTTP contexts. |
| `mythos-api` | `axum` routers and handlers: `auth`, `library`, `movie`, `series`, `episode`, `scan`, `play`, `hls`, `subtitles`, `settings`, `search`. |
| `mythos-scan` | Filesystem walker (`jwalk`) and `ffprobe` driver. Movie and TV branches share the walk; the TV branch parses `SxxEyy` / `1x01` with a season-dir fallback. Movie identification prefers `(YYYY)`-bracketed years over bare year tokens so titles like `Blade Runner 2049 (2017)` parse correctly. |
| `mythos-meta` | TMDb client with an on-disk poster cache. Handles movies, TV series, seasons, and episode stills. The movie enrichment pass retries without the year hint when the first attempt misses, rescuing year-typo files. MusicBrainz / OpenLibrary land later in Phase 3. |
| `mythos-stream` | Direct-play byte-range responses, FFmpeg HLS transcoder, ABR ladder, hardware-encoder probe, subtitle burn-in, HDR→SDR tonemapping pipeline. Movies and episodes share the streaming surface via a `SessionKey { user_id, item_id, kind }` so their transcode sessions can't collide. |

Outside `crates/`, `apps/mythos-desktop/` is a Tauri 2 workspace member
whose Cargo crate lives at `apps/mythos-desktop/src-tauri/`. Its UI is
the same SvelteKit codebase as the server's embedded SPA; runtime
backend selection (`'__TAURI_INTERNALS__' in window`) picks libmpv-via-IPC
vs. `<video>` + hls.js at startup.

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
  timestamps (migration 0011). Only true IDR slice NALs are recorded
  (type 5 for H.264, types 19/20 for HEVC); CRA and BLA frames are
  excluded because their RASL leading pictures reference frames from
  before the boundary and would stutter when the player starts playback
  there. Open-GOP HEVC files (most modern release groups) therefore end
  up with empty indexes — `/play` auto-downgrades them to a full
  transcode instead of remuxing.
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
installs set those env vars to enable the option. It's also the
practical answer on Intel Gen 12+ (Iris Xe and newer), where the
`opencl` pipeline is broken at the NEO driver level — NEO no longer
advertises `cl_intel_va_api_media_sharing`, so the `hwmap` step
returns ENOSYS. The Docker image deliberately omits `intel-opencl-icd`
for the same reason.

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

### Player overlay lifecycle

The video player is hoisted into the root layout (`+layout.svelte`) and
managed by a singleton `playbackSession`
(`web/src/lib/player/session.svelte.ts`). Pages don't mount the player
themselves — they call `playbackSession.open(...)` and a single overlay
component decides whether to render as a fullscreen modal or as an
88&nbsp;px mini-bar pinned to the bottom of the page.

The overlay toggles between modes via a `data-mode` attribute; the same
`<Player>` instance is rendered in both modes so the `<video>` element,
the `media-controller`, the playback backend, and the active HLS session
are not torn down on a mode swap. Modal-only chrome (top overlay,
scrubber and buttons bars, subs menu, up-next countdown, info strip)
is `{#if mode === 'modal'}`-gated; the mini chrome is a `.mini-row`
sibling of `<media-controller>` driven by the same `backendPaused` /
`backendPositionMs` mirrors the modal uses, so it works identically in
browser and Tauri/mpv modes.

Clicking the small video in the mini-bar calls `playbackSession.expand()`
to restore the fullscreen modal. <kbd>Esc</kbd> on the modal minimizes
rather than closes — the explicit X is the only true "close" action.

### HLS session teardown

HLS sessions are torn down via two paths that both call
`DELETE /api/{movies,episodes}/:id/hls`:

- `playbackSession.close()` and `playbackSession.open()` (with a
  different item) fire `stopTranscodeSession` *synchronously*, so the
  DELETE goes out the instant the user closes or swaps the player
  rather than waiting on the overlay's fade transition.
- The Player's `$effect` cleanup also calls `stopTranscodeSession` on
  unmount — a safety net for paths that bypass the session (`beforeunload`
  on page reload, manual route navigation outside the SPA router).

Every new transcode entry point routes through `TranscodeManager` so
the lifecycle stays centralized and ffmpeg subprocesses don't leak.

### Desktop client (Tauri + libmpv)

`apps/mythos-desktop/` is a Tauri 2 shell around the same SvelteKit
codebase as the server's embedded SPA. The same routes, the same
components, the same `Player.svelte` — but at runtime the playback
layer dispatches to **libmpv via IPC** instead of `<video>` + hls.js.
Mpv's position / paused state is mirrored back into the same Svelte
stores the browser backend writes to, so `Player.svelte` doesn't
branch on backend.

Mpv runs as `Arc<Mpv>` in the host process; its event loop lives on a
separate thread with its own `EventContext::new(mpv.ctx)` (the libmpv2
test pattern, because `Mpv::event_context_mut` would need `&mut Mpv`
which is incompatible with the `Arc<Mpv>` shared by IPC commands).

Video embeds into the Tauri main window at setup time: Rust hands
libmpv the window's raw handle (via `raw-window-handle`'s
`WindowHandle::as_raw`) and sets mpv's `wid` property, with
`force-window=no` so nothing pops up until the embed lands. The SPA
reports its `<video>` bounding rect, Rust translates that to ratios
using the Tauri window's `inner_position` / `outer_position` /
`scale_factor`, and mpv renders with `alpha=yes` + `background=none`
plus its `video-margin-ratio-{l,r,t,b}` so the surface outside the
video rect is transparent and the webview shows through. This relies
on a compositing window manager and a mpv VO that honours `alpha=yes`
(default `gpu` VO does; older `xv` / `vaapi` fallbacks may not).

Two limits today, both planned follow-ups:

- The embed surface is the *top-level* Tauri window, so mpv's child
  surface overlays the SvelteKit chrome during playback. A child
  native widget below the webview is the next milestone.
- Wayland sessions that report `wl_surface` handles aren't supported
  by the current attach path. There's an
  `Mpv::enable_standalone_window` fallback that flips
  `force-window=yes` so video plays in a separate window from the
  chrome.

## Why Rust

Three reasons:

1. **Single binary.** `cargo build --release` produces one statically linked
   executable. The SPA is baked in. There is no runtime to install, no
   `node_modules` to ship.
2. **Predictable performance.** No GC pauses while you're seeking. The
   transcode supervisor can hold its frame budget without surprises.
3. **Memory honesty.** Long-running media servers accumulate small leaks.
   Lifetimes and `Drop` make those visible at compile time.
