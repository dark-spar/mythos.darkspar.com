---
title: "Phase 5 — hardware-accelerated streaming lands"
date: 2026-05-12
description: "Multi-rendition ABR HLS, subtitle burn-in, and client-declared device profiles are all in main."
tags: ["status"]
---

Phase 5 closed out this week. Mythos can now stream a 4K HEVC HDR file with
PGS subtitles to a Firefox tab and have it Just Work — that's what Phase 5
was supposed to unlock.

## What shipped

- **Phase 5a — hardware-accelerated transcoding.** `mythos-stream` probes
  `ffmpeg -encoders` at startup and smoke-tests each candidate (NVENC →
  QSV → VAAPI → VideoToolbox → libx264). A build with NVENC compiled in but
  no working driver falls back cleanly rather than hard-failing at first
  play.
- **Phase 5b — multi-rendition ABR HLS.** The HLS pipeline produces multiple
  bitrate ladders and hands the player a master playlist. hls.js picks the
  rendition.
- **Phase 5c — subtitle burn-in + WebVTT sidecar.** Text formats (SRT, ASS,
  VTT) ride along as WebVTT sidecar tracks. Image formats (PGS, VobSub) burn
  into the transcoded stream so they survive every device.
- **Phase 5d — client-declared device profile.** The web client tells the
  server what it can play; the server picks direct-play or transcode
  per-codec instead of guessing from the User-Agent. The transcode taxonomy
  (passthrough vs. remux vs. full transcode, per-stream) is exposed in the
  player's compatibility banner.

A handful of fixes followed: HLS playback on Firefox Android, VAAPI
full-pipeline corner cases (decode + scale + encode all on the GPU), text
subtitle lifecycle glitches, and tearing down sessions when the player
unmounts.

## What's next

Phase 3 — the **non-movie** media types: TV series, music, photos, books. The
schema already accepts those library kinds; the scanners and UIs are what's
missing. After that, Phase 6 lights up the Jellyfin-API compatibility shim
so existing clients like Findroid and Swiftfin work against Mythos without
changes.

Source on [GitLab](https://gitlab.com/darkspar/mythos).
