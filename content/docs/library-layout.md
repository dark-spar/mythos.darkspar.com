---
title: "Library Layout"
description: "How to organize files so Mythos understands them — today and on the roadmap."
weight: 30
---

Mythos infers what it can from filenames and folder structure. Conventions
match the ones used by Plex / Jellyfin / Emby — an existing library should
work without renaming anything.

Right now, **only movie libraries are wired up.** The `libraries.kind`
column accepts `movies`, `shows`, `music`, `photos`, `books`, but the
per-kind tables (episodes, series, tracks, albums, artists, photos, books)
**don't exist yet** — they ship alongside their scanners in Phase 3. You
can create a non-movies library in the admin UI today; the scanner just
won't find anything to put in it.

## Movies

```
/media/films/
├── The Princess Bride (1987)/
│   └── The Princess Bride (1987).mkv
├── My Neighbor Totoro (1988)/
│   └── My Neighbor Totoro (1988).mp4
└── Spirited Away (2001)/
    ├── Spirited Away (2001).mkv
    └── Spirited Away (2001).en.srt
```

Each film lives in its own folder, ideally named `Title (Year)`. The scanner
parses `(YYYY)` out of the folder name for matching against TMDb, walks the
directory with `jwalk`, and probes each video file with `ffprobe` for
container, codecs, duration, and resolution.

Files where `ffprobe` fails or is unavailable are still indexed — the
technical fields just stay `NULL` until a future re-scan fills them in.

### Sidecar files Mythos handles today

| File | Purpose |
|---|---|
| `<basename>.srt`, `.ass`, `.vtt` | External subtitles. Text formats become WebVTT; image subs are burned in during transcode. |

External posters, fanart, and NFO overrides are on the roadmap but not wired
up yet — posters come from TMDb if you've set an API key, otherwise the UI
falls back to a generic placeholder.

## TV, music, photos, books

Coming in Phase 3:

- TV series with the `Show/Season XX/Show - SxxEyy - Title.ext` layout.
- Music with tags via [`lofty`](https://github.com/Serial-ATA/lofty-rs) and
  metadata from MusicBrainz.
- Photos with thumbnails and EXIF, via `image` + `fast_image_resize` +
  `kamadak-exif`.
- Books via EPUB metadata, rendered with `epub.js`.

The `libraries` table accepts those kinds today, so the admin UI lets you
create one — there just isn't a scanner or per-kind schema behind it yet.

## Storage model

Paths are stored **relative to the library's `root_path`**, so moving a
library means updating one row, not rewriting every file path in the
database. The scanner combines `root_path + path` at serve time.

Deleting a library cascades through `media_files` → `movies`, so removing a
library is one DELETE with no orphan rows.
