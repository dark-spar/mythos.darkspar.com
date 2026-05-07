---
title: "Architecture"
description: "A short tour of how Mythos is built."
weight: 40
---

Mythos is a Cargo workspace. The crates split along clear seams so each piece does one thing and stays small.

## The crates

| Crate | Responsibility |
|---|---|
| `mythos-core` | Domain types, errors, configuration |
| `mythos-db` | SQLite/Postgres access via `sqlx`, migrations |
| `mythos-scan` | Filesystem watcher, library indexer |
| `mythos-meta` | Metadata fetchers (TMDB, MusicBrainz, local NFO) |
| `mythos-stream` | Transcoding pipeline, HLS muxer, hardware acceleration |
| `mythos-api` | HTTP API, authentication, RBAC |
| `mythos-server` | Binary that wires it all together with Axum |

## The runtime

A single Tokio runtime hosts everything. The HTTP server, the library scanner, and the transcoding workers share the same scheduler — which keeps overhead low and lets us coordinate work without crossing process boundaries.

## State

Almost all state lives in SQLite by default. The schema is migration-managed and small enough to inspect by hand. Posters, thumbnails, and transcoded segments live on disk under <code>data_dir</code>.

## Why Rust

Three reasons we keep coming back to:

1. **Memory honesty.** A long-running media server is a place where small leaks become big problems. Rust's lifetimes and the `Drop` discipline make the leaks visible at compile time.
2. **Predictable performance.** No GC pauses while you're trying to seek. The transcoding pipeline can hold to its frame budget without surprise.
3. **Single binary.** `cargo build --release` produces a static executable. No interpreter, no JIT warm-up, no extra runtime to install on the home server.
