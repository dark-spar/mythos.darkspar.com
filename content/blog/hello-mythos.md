---
title: "Hello, Mythos"
date: 2026-04-12
description: "Why I'm building another media server, and what's actually in it right now."
tags: ["announcement"]
---

There are already good media servers in the world. Plex, Jellyfin, Emby — each
has earned a place on home servers everywhere. So why a new one?

The honest answer is _shape_. I wanted a media server that's a single
statically linked binary with the web UI baked in, that doesn't drag a
container runtime or a Node process or a separate database into the room,
and that takes streaming seriously enough to do hardware-accelerated HLS
without me babysitting it.

Mythos is that — a small Rust workspace with the engineering rigor of a
database and the surface area of a CLI tool.

## What's in `main` today

- **Movies.** Library CRUD, filesystem scan via `jwalk` + `ffprobe`, TMDb
  enrichment, browse and detail UI.
- **Streaming.** Direct play with HTTP byte ranges, plus HLS transcoding via
  `ffmpeg` when the browser can't decode the file. Multi-rendition ABR.
- **Hardware acceleration.** NVENC, QSV, VAAPI, and VideoToolbox are probed
  and smoke-tested at startup; libx264 is the fallback.
- **Subtitles.** Text subs ride along as WebVTT sidecars; image subs (PGS,
  VobSub) burn in during transcode.
- **Auth.** argon2id password hashes, JWT cookies, admin/non-admin split.
- **One binary.** SvelteKit build is embedded via `rust-embed`.

## What's not yet

- TV, music, photos, books. The schema accepts them; the scanners that fill
  them in are Phase 3.
- Jellyfin-API compatibility shim for Findroid / Swiftfin / jellyfin-web —
  Phase 6.
- Pre-built binaries and a Docker image. Real `v0.1.0` will ship those.

If any of that sounds interesting, the [docs](/docs/) walk through the
build-from-source path. The source lives on
[GitLab](https://gitlab.com/darkspar/mythos).
