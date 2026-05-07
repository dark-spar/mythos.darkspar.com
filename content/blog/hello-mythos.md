---
title: "Hello, Mythos"
date: 2026-04-12
description: "Why we're building another media server, and what we want it to feel like."
tags: ["announcement"]
---

There are already good media servers in the world. Plex, Jellyfin, Emby — each has earned a place on home servers everywhere. So why a new one?

The honest answer is _feel_. The software that lives in our living rooms gets touched on cold winter Sundays, summer afternoons, late nights after the kids are asleep. It should feel calm, warm, and trustworthy. It should never make you fight it. It should never feel like work.

Mythos is our attempt at that — a media server with the engineering rigor of a database and the temperament of a paperback novel pulled off a shelf.

## What's in the first cut

- A complete library scanner that handles movies, TV, and music
- Hardware-accelerated transcoding (VAAPI, QSV, NVENC, VideoToolbox)
- A web UI that loads in under 200ms on a Pi
- TMDB and MusicBrainz metadata, with local NFO overrides
- Single static binary, ~12MB

## What's coming

Live TV recording. A real mobile app. Watch-together rooms. Photo libraries that actually work for families.

If any of that sounds interesting, the [docs](/docs/) are the best place to start.
