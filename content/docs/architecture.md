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
| `mythos-api` | `axum` routers and handlers: `auth`, `library`, `movie`, `scan`, `play`, `hls`, `subtitles`, `settings`. |
| `mythos-scan` | Filesystem walker (`jwalk`) and `ffprobe` driver. |
| `mythos-meta` | TMDb client with an on-disk poster cache. MusicBrainz / OpenLibrary land in Phase 3. |
| `mythos-stream` | Direct-play byte-range responses, FFmpeg HLS transcoder, ABR ladder, hardware-encoder probe, subtitle burn-in. |

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

The current schema (migrations 0001–0007):

- `users` — argon2id hashes, `token_version` for forced logout.
- `libraries` — root paths the scanner walks.
- `media_files` — one row per file on disk, with `ffprobe` columns.
- `movies` — one row per identified movie, pointing at a `media_files` row.
- `movie_progress` — debounced watch position per user.
- `media_subtitles` — extracted text subs + image-sub render artifacts.
- `settings` — runtime-configurable settings (TMDb key, etc.).

Tables for episodes, series, tracks, albums, artists, photos, and books
**don't exist yet** — they ship in Phase 3 alongside their scanners.

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
  byte-range support. Vidstack binds the URL as a `<video>` source. Watch
  progress is debounced and persisted server-side so resume works across
  devices.
- **HLS transcode.** If the client's declared profile says the file is
  unplayable, Mythos spawns an `ffmpeg` session via the `TranscodeManager`,
  produces segmented HLS, and serves segments on demand. The player calls
  `DELETE /api/movies/:id/hls` on teardown so `ffmpeg` subprocesses don't
  leak — every new transcode entry point routes through `TranscodeManager`
  to keep the lifecycle centralized.

The HLS pipeline supports multi-rendition ABR. Hardware encoders are probed
and smoke-tested at startup — a build with NVENC compiled in but no working
driver falls back cleanly. Priority order is NVENC → QSV → VAAPI →
VideoToolbox → libx264.

## Why Rust

Three reasons:

1. **Single binary.** `cargo build --release` produces one statically linked
   executable. The SPA is baked in. There is no runtime to install, no
   `node_modules` to ship.
2. **Predictable performance.** No GC pauses while you're seeking. The
   transcode supervisor can hold its frame budget without surprises.
3. **Memory honesty.** Long-running media servers accumulate small leaks.
   Lifetimes and `Drop` make those visible at compile time.
