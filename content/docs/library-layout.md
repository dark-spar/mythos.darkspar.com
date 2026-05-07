---
title: "Library Layout"
description: "How to organize files so Mythos understands them."
weight: 30
---

Mythos infers as much as it can from filenames and folder structure. The conventions below are the same ones used by Plex, Jellyfin, and Emby — so an existing library should work out of the box.

## Movies

```
/media/films/
├── The Princess Bride (1987)/
│   ├── The Princess Bride (1987).mkv
│   └── poster.jpg
├── My Neighbor Totoro (1988)/
│   └── My Neighbor Totoro (1988).mp4
└── Spirited Away (2001)/
    ├── Spirited Away (2001).mkv
    └── Spirited Away (2001).en.srt
```

Each film lives in its own folder, named <code>Title (Year)</code>. External subtitles, posters, and trailers picked up automatically.

## TV shows

```
/media/shows/
└── Bluey/
    ├── Season 01/
    │   ├── Bluey - S01E01 - Magic Xylophone.mkv
    │   └── Bluey - S01E02 - Hospital.mkv
    └── Season 02/
        └── Bluey - S02E01 - Dance Mode.mkv
```

Seasons are folders, episodes follow <code>Title - SXXEYY - Episode Name</code>.

## Music

```
/media/music/
└── Nick Drake/
    └── Pink Moon (1972)/
        ├── 01 Pink Moon.flac
        ├── 02 Place To Be.flac
        └── cover.jpg
```

Mythos reads tags first and falls back to folder structure. Both work.

## Sidecar files

| File | Purpose |
|---|---|
| `poster.jpg` / `cover.jpg` | Library art |
| `fanart.jpg` / `backdrop.jpg` | Background art |
| `<basename>.srt`, `.ass`, `.vtt` | External subtitles |
| `<basename>.nfo` | Local metadata override |
| `.mythosignore` | Glob list of paths to skip |
