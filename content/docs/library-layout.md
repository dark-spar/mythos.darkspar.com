---
title: "Library Layout"
description: "How to organize files so Mythos understands them — today and on the roadmap."
weight: 30
---

Mythos infers what it can from filenames and folder structure. Conventions
match the ones used by Plex / Jellyfin / Emby — an existing library should
work without renaming anything.

Right now, **movie and TV libraries are wired up.** The `libraries.kind`
column accepts `movies`, `shows`, `music`, `photos`, `books`; the
`movies`, `series`, `seasons`, `episodes` tables exist today, while the
music / photos / books per-kind tables don't yet — they ship alongside
their scanners later in Phase 3. You can create a non-movies/shows
library in the admin UI today; the scanner just won't find anything to
put in it.

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

## TV

```
/media/shows/
├── Severance/
│   ├── Season 01/
│   │   ├── Severance - S01E01 - Good News About Hell.mkv
│   │   ├── Severance - S01E02 - Half Loop.mkv
│   │   └── …
│   └── Season 02/
│       └── …
└── Andor/
    └── Season 01/
        ├── Andor.1x01.mkv
        └── Andor.1x02.mkv
```

The TV scanner runs against any library with `kind = shows`. Identity
before TMDb enrichment is `(library_id, sort_title)`, so a re-scan
never duplicates a series even if its TMDb ID hasn't been resolved
yet — enrichment fills `tmdb_id`, `overview`, `poster_url`, and
per-episode stills without changing the natural key.

The filename identifier recognises both `S01E01` and `1x01` patterns,
and falls back to the parent `Season NN/` directory if the filename
itself only contains an episode number.

Episodes inherit everything `media_files` provides — `ffprobe`
columns, subtitle sidecars, HDR detection — so direct-play and HLS
work for episodes with no code branching on media kind.

## Music, photos, books

Coming later in Phase 3:

- Music with tags via [`lofty`](https://github.com/Serial-ATA/lofty-rs) and
  metadata from MusicBrainz.
- Photos with thumbnails and EXIF, via `image` + `fast_image_resize` +
  `kamadak-exif`.
- Books via EPUB metadata, rendered with `epub.js`.

The `libraries` table accepts those kinds today, so the admin UI lets you
create one — there just isn't a scanner or per-kind schema behind them yet.

## Storage model

Paths are stored **relative to the library's `root_path`**, so moving a
library means updating one row, not rewriting every file path in the
database. The scanner combines `root_path + path` at serve time.

Deleting a library cascades through `media_files` → `movies` and
`media_files` → `episodes`, so removing a library is one DELETE with
no orphan rows. Empty `seasons` / `series` rows left behind after the
prune pass are cleaned up by the scanner before it returns.
