---
title: "Phase 3 starts: TV lands, plus HDR→SDR tonemapping"
date: 2026-05-15
description: "TV scanner, schema, episode playback, continue-watching, auto-play-next — and an explicit HDR→SDR tonemap pipeline in the transcoder."
tags: ["status"]
---

Two strands of work just landed in `main`. The first kicks Phase 3 off
with TV; the second closes a gap that was always going to bite as soon
as people pointed a 4K HDR file at the server.

## TV — Phase 3a, 3c, 3d

Mythos now scans, identifies, enriches, and plays TV episodes.

- **Schema.** Migration `0008_tv.sql` adds `series → seasons → episodes`
  plus a per-user `episode_progress` table. Each `episodes` row FKs 1:1
  to a `media_files` row, so the same byte-range streaming, HLS
  transcoder, subtitle pipeline, and ffprobe metadata that movies rely
  on work for episodes with no branch in the streaming code.
- **Scanner.** The filesystem walker has a TV branch keyed on
  `LibraryKind::Shows`. The filename identifier recognises both
  `S01E01` and `1x01` patterns, with a `Season NN/` directory fallback
  for filenames that only carry an episode number.
- **TMDb enrichment.** `search_tv` + `tv/{id}/season/{n}` resolve
  series, seasons, and per-episode titles, overviews, and stills.
  Series identity before enrichment is `(library_id, sort_title)`, so
  a re-scan can't duplicate a series even if its TMDb ID is still
  `NULL`.
- **Playback.** `/api/episodes/{id}/{stream,play,progress,hls/*,subtitles/.../vtt}`
  mirrors the movie surface. A shared `SessionKey { user_id, item_id, kind }`
  keeps movie and episode transcode sessions from colliding. The
  `/movie/[id]` and `/episodes/[id]` pages both render a shared
  `Player.svelte`.
- **Stitching (Phase 3d).** A `GET /api/users/me/continue-watching`
  endpoint aggregates progress across movies and episodes. At the end
  of an episode the player shows an auto-play-next countdown card and
  navigates to the next episode with `state: { autoplay: true }`, so
  consecutive episodes play seamlessly while "click thumbnail = paused"
  stays the default for manual navigation.

Music, photos, and books are the remaining Phase 3 sub-phases.

## HDR → SDR tonemapping

A 4K HEVC HDR file pointed at a Firefox tab used to come out grey
and washed-out — the transcoder was happily encoding HDR samples as
SDR without telling anyone. The fix is an explicit tonemap stage in
the filter graph, chosen by an admin-editable setting:

- **Pipelines.** `software`, `vaapi`, `opencl`, `cuda`. The server
  probes ffmpeg at startup for which tonemap filters are actually
  compiled in (`tonemap_vaapi` / `tonemap_opencl` / `tonemap_cuda`);
  pipelines whose filter is missing silently fall back to `software`,
  so a stale DB value can't break playback.
- **Algorithms.** `hable` (default), `mobius`, `reinhard`, `bt2390`.
  Honoured by the software / OpenCL / CUDA pipelines; the VAAPI
  filter has no algorithm knob, so it ignores this setting.
- **HDR detection.** Migration `0009_media_color_metadata.sql` adds
  `color_primaries` / `color_transfer` / `color_space` columns on
  `media_files`. Libraries scanned before this migration have those
  columns NULL; rather than force a rescan, the first HDR play
  self-heals the row by ffprobing on demand.

Alongside this, the NVENC path stays on the GPU end-to-end now via
`NVDEC` + `scale_cuda` — no system-RAM round-trip during a transcode.
And there are two new env vars, `MYTHOS_FFMPEG_BIN` and
`MYTHOS_FFPROBE_BIN`, so you can pin a custom ffmpeg build with the
encoders and tonemap filters you actually want.

## What's next

The next Phase 3 sub-phases — music, then photos, then books. Phase 6,
the Jellyfin-API compatibility shim, is still after that.

Source on [GitLab](https://gitlab.com/darkspar/mythos).
