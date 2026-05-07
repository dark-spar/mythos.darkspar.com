---
title: "v0.1.0 — The first release"
date: 2026-04-28
description: "Mythos 0.1.0 is out. Here's what's in it, and what to expect from a 0.1."
tags: ["release"]
---

The first cut is here. It's small and a little rough around the edges, but it streams a movie, scans a library, and runs quietly on a Pi 5 — which were the three things we had to get right before we'd put a version number on it.

## Highlights

- **Library scanning.** Movies, TV, music. Handles the standard naming conventions, picks up sidecar art and subtitles, and watches the directory for changes after the initial scan.
- **Direct play & transcoding.** Picks the right path automatically. Transcoding uses VAAPI on Intel, NVENC on NVIDIA, and falls back to libx264 if no GPU is available.
- **Web UI.** Built with SvelteKit. Loads under 200ms on first paint, adapts to large TVs and small phones.
- **Single binary.** ~12 MB, statically linked, no runtime.

## Known limits in 0.1

- No clustering, no multi-server. One Mythos, one home.
- No mobile apps yet — the web UI is responsive, but a native iOS/Android pair is on the roadmap.
- Live TV is unimplemented. We want it; it's the headline feature for 0.3.

## What's next

We're already on 0.2. The headline change is **collaborative watch sessions** — the ability for two screens in two homes to play the same film, in sync, with a shared chat. We've wanted it for a long time. It's nearly ready.

Get the release on the [download page](/download/), or read the source on [GitHub](https://github.com/mattmoore/mythos).
