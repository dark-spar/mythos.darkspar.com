---
title: "Getting Started"
description: "From zero to streaming in five minutes."
weight: 10
---

This guide gets you from a fresh install to playing your first movie. We'll use Docker, but every other install path lands in the same place.

## 1. Run the server

```sh
docker run -d \
  --name mythos \
  -p 7878:7878 \
  -v /srv/media:/media:ro \
  -v mythos-data:/var/lib/mythos \
  ghcr.io/mattmoore/mythos:latest
```

A couple of things to notice. The media volume is mounted **read-only** — Mythos never writes to your library. The data volume is where Mythos keeps its own database, posters, and transcoded segments.

## 2. Open the web UI

Visit <code>http://localhost:7878</code>. You'll be walked through:

1. Creating the first administrator account
2. Adding a library (point it at <code>/media</code>)
3. Choosing a metadata source

The first scan starts immediately. On a typical library, posters and metadata are populated within a few minutes.

## 3. Watch something

Click any title. Mythos picks the best playback strategy automatically — direct play if your client can handle the file, hardware-accelerated transcoding if it can't.

## What's next

- [Configuration reference](../configuration/) — the full <code>config.toml</code>
- [Library layout](../library-layout/) — how Mythos discovers and groups your files
- [Architecture](../architecture/) — what's happening under the hood
