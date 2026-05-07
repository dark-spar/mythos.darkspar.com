---
title: "Configuration"
description: "Every knob in config.toml."
weight: 20
---

Mythos reads its configuration from <code>config.toml</code>. Every setting also has an environment-variable equivalent, prefixed with <code>MYTHOS_</code>, which takes precedence — useful in containerized deployments.

## A minimal config

```toml
[server]
host = "0.0.0.0"
port = 7878

[database]
url = "sqlite:///var/lib/mythos/mythos.db"

[[libraries]]
name = "Films"
path = "/media/films"
kind = "movies"

[[libraries]]
name = "Shows"
path = "/media/shows"
kind = "tv"
```

## Server

| Key | Default | Description |
|---|---|---|
| `host` | `0.0.0.0` | Address to bind |
| `port` | `7878` | TCP port |
| `data_dir` | `/var/lib/mythos` | Where Mythos stores its state |
| `tls.cert` | _(none)_ | Path to a TLS certificate. Enables HTTPS. |
| `tls.key` | _(none)_ | Path to the private key |

## Database

Mythos uses SQLite by default — fast, transactional, and zero-administration for home use. PostgreSQL is supported for households that want shared deployments.

```toml
[database]
url = "sqlite:///var/lib/mythos/mythos.db"
# or
url = "postgres://mythos:secret@localhost/mythos"
```

## Libraries

Each library is a top-level collection of media. You can mix kinds — movies, tv, music, photos — across as many libraries as you like.

```toml
[[libraries]]
name = "Films"
path = "/media/films"
kind = "movies"
metadata = ["tmdb", "local-nfo"]

[[libraries]]
name = "Family Photos"
path = "/media/photos"
kind = "photos"
```

## Transcoding

```toml
[transcoding]
hardware = "auto"      # one of: auto, vaapi, qsv, nvenc, videotoolbox, none
threads = 0            # 0 = use all cores
segment_duration = 6   # seconds per HLS segment
```

Mythos auto-detects the best hardware encoder available. Set <code>hardware = "none"</code> if you'd rather stick to CPU encoding.
