---
title: "Docker images, title search, and a new player"
date: 2026-05-16
description: "Four pieces of work since Phase 3 landed: a published Docker image, a title-search box across movies and series, a media-chrome player rebuild, and the Tonemapx CPU pipeline."
tags: ["status"]
---

A handful of things landed in `main` since the [Phase 3 +
HDR→SDR](/blog/phase-3-tv-and-tonemapping/) post — none of them open a
new phase, but together they take a real chunk out of the "early
development" rough edges.

## Docker images on Docker Hub

The fastest way to try Mythos is now a `docker pull`. Multi-arch images
(`linux/amd64` and `linux/arm64`) get rebuilt and pushed on every commit to
`main`:

```sh
docker run -d --name mythos \
  -p 8080:8080 \
  -v mythos-data:/data \
  -v /path/to/media:/media:ro \
  -e MYTHOS_TMDB_API_KEY=... \
  darkspar/mythos-server:edge
```

A few opinions baked into the image:

- **`jellyfin-ffmpeg7` bundled.** That gets you the `tonemap_vaapi` /
  `tonemap_opencl` / `tonemap_cuda` HW filters plus `tonemapx` (more on
  that below) without having to hunt them down distro-by-distro. The
  image already points `MYTHOS_FFMPEG_BIN` / `MYTHOS_FFPROBE_BIN` at it.
- **PID 1 is `tini`.** Signals reach `mythos-server` cleanly; reaping is
  handled.
- **`/data` is the single writable volume.** SQLite DB, JWT secret,
  poster cache, transcode segments, sidecar subtitles — everything Mythos
  writes lands there. Bind-mount your media read-only.

Tags follow Docker conventions: `:edge` is rolling, `:sha-<short>` is
immutable per-commit (pin this for production-ish setups so an unattended
`docker pull` doesn't surprise you), and semver tags will appear when the
first `v*` git tag is pushed. There's intentionally no `:latest` yet —
by Docker convention that means "newest stable release," and Mythos
hasn't cut one.

## Title search across the library

A single search box. One endpoint, `GET /api/search?q=…`, returns a
flat ranked list across movies and series. The implementation is
deliberately humble — case-insensitive `LIKE` against `sort_title` via
`mythos_db::SearchRepo` — and will graduate to SQLite FTS5 once
libraries get big enough to chug. For now it's instant on the libraries
people actually have.

The UI is the part I'm happier with. Results render inline as you type;
<kbd>↑</kbd>/<kbd>↓</kbd> walks the list, <kbd>Enter</kbd> opens, and
<kbd>Esc</kbd> dismisses. There's no separate "search page" — the
results panel sits under the box on whatever page you're on, and the
keyboard never leaves the input.

## A new player, built on `media-chrome`

The old player was the native `<video controls>` element with a couple
of overlays. It worked, but customising the chrome ranged from
hard-to-impossible across browsers, and the more features Mythos grew
(continue-watching cards, auto-play-next countdowns, an in-player CC
picker) the worse the seams showed.

`Player.svelte` is now rebuilt on top of
[`media-chrome`](https://www.media-chrome.org/) — a set of Web
Components that wrap a plain `<video>` element with named slots for
every control. The video element underneath is still a real
`HTMLVideoElement`; `media-chrome` is purely a chrome layer.

What that unlocked:

- **YouTube/Netflix-flavoured chrome.** A two-bar layout with all
  controls visible; the volume slider is a hover-reveal so it doesn't
  clutter mobile; the close button lives on the player's top-right
  corner.
- **An actual CC menu.** A subtitle picker as a fixed-position panel,
  not a hard-coded `<track>` dropdown. Toggling it no longer pauses the
  video.
- **Click / dblclick gestures on the video frame.** Single click toggles
  play; double click toggles fullscreen. `media-chrome`'s built-in
  gesture receiver is disabled so they don't double-fire with our own.
- **Hover-reveal play buttons on tiles and detail pages.** A poster art
  hover now shows a Play affordance; movie + episode tiles have a
  watch-progress bar baked in.

## `Tonemapx` — SIMD CPU tonemap

The newest pipeline option in the HDR→SDR settings is `Tonemapx`,
jellyfin-ffmpeg's SIMD-optimised replacement for the stock CPU `tonemap`
filter. On the CPU path it's dramatically faster than software — fast
enough that a CPU-only box can keep up with HDR sources where it
previously couldn't.

The catch: `tonemapx` is jellyfin-ffmpeg-only on most distros. Mythos
probes for it at startup, so picking it from the admin UI on a stock
distro ffmpeg silently falls back to `software` rather than breaking
playback. The Docker image already ships jellyfin-ffmpeg and points the
ffmpeg binary env vars at it; on bare-metal installs set
`MYTHOS_FFMPEG_BIN=/usr/lib/jellyfin-ffmpeg/ffmpeg` (and the matching
ffprobe) to light it up.

The pipeline list is now: `software`, `tonemapx`, `vaapi`, `opencl`,
`cuda`. NVENC stays on the GPU end-to-end via `NVDEC` + `scale_cuda` +
`tonemap_cuda`; QSV / VideoToolbox boxes get the SIMD CPU kernel inline
where there's no GPU filter surface to download to.

## What's next

Still Phase 3: music, photos, books — separate sub-phases. After that,
Phase 6 lights up the Jellyfin-API compatibility shim so existing
clients like Findroid and Swiftfin work against Mythos without
changes.

Source on [GitLab](https://gitlab.com/darkspar/mythos).
