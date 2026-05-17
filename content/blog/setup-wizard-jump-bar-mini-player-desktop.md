---
title: "Setup wizard, alphabet jump bar, mini-bar player, and a desktop client"
date: 2026-05-17
description: "A 3-step onboarding wizard, a Plex-style alphabet jump bar on library browse, a persistent mini-bar player that survives navigation, the start of a native Tauri desktop client, and a TMDb identifier that finally gets Blade Runner 2049 right."
tags: ["status"]
---

Another batch of work landed in `main` overnight — none of it opens a new
phase, all of it takes the rough edges off how Mythos *feels*. The themes:
first-run, library browse, the player, and (newest) a native desktop
shell.

## A 3-step setup wizard

First-run used to be a single form: pick a username, pick a password,
optionally drop in a TMDb key, optionally point at a library — submit,
hope. It worked, but it asked you to make every decision at once before
you'd seen anything load.

`/setup` is now a three-step wizard:

1. **Admin account.** Username + password, argon2id-hashed.
2. **TMDb key (optional).** Skip it and scans still index files; add it
   later from admin settings without a restart (the live `TmdbHandle`
   swaps on save).
3. **First library.** Point at a directory; the scan starts immediately.

The route re-uses the existing `POST /api/auth/register`,
`PUT /api/settings/tmdb`, and `POST /api/libraries` endpoints — the
wizard is purely a frontend reshape that advances in-page via local
state, so the `+layout.svelte` "redirect setup → login once an admin
exists" rule doesn't fight us mid-flow.

## A Plex-style alphabet jump bar on library browse

Bigger movie libraries hit the same wall every browse view eventually
does: scrolling to *T* takes forever. `/library/[id]` now renders a
full-height vertical alphabet rail down the right edge — click *T*, the
view jumps to the *T* bucket. Two things worth pointing at:

- **Leading articles are stripped before bucketing.** "The Matrix"
  buckets under *M*, not *T*. "An American Werewolf in London" goes
  under *A* (American), not the article. This matches how the rest of
  the UI sorts.
- **Scroll alignment respects the sticky top nav.** Buckets are
  anchored by `id="bucket-…"` with `scroll-mt-20`, so clicking a
  letter doesn't bury the first row under the floating `TopNav`.

The rail itself uses a pointer cursor on the letter buttons —
small thing, but `<button>` defaults to the text cursor and the rail
read as "label, not control" until that flipped.

## The PiP button is now a real mini-bar

The biggest behaviour change in this batch. The player used to be a
modal: open it, watch, close, gone. If you opened a movie, then
clicked back to browse, you got the equivalent of `pause + close`.

The player is now hoisted into the root layout
(`web/src/routes/+layout.svelte`) and managed by a singleton
`playbackSession` (`web/src/lib/player/session.svelte.ts`). Pages
don't mount the player — they call `playbackSession.open(...)` and a
single overlay component decides whether to render as a fullscreen
modal or as an 88 px bar pinned to the bottom of the page.

What that means in practice:

- **Playback survives navigation.** Hit the PiP button on a playing
  movie and the player collapses to a mini-bar; the underlying
  `<video>` / `media-controller` / HLS session aren't torn down —
  the same instance is just re-styled. Browse anywhere else on the
  site, the bar stays.
- **Click the small video to expand.** `playbackSession.expand()`
  restores the modal layout. Esc on the modal *minimizes* rather
  than closes — the explicit X is the only true "close" affordance.
- **Modal-only chrome is `{#if mode === 'modal'}`-gated.** Top
  overlay, scrubber + buttons bars, subs menu, up-next countdown,
  info strip — all hidden in mini mode. The mini chrome is a
  `.mini-row` sibling of `<media-controller>` driven by the same
  `backendPaused` / `backendPositionMs` mirrors the modal uses, so
  it works identically in browser and (new) Tauri/mpv modes.

The fullscreen modal got its own polish pass at the same time —
Plex-style: backdrop blurred out behind a centered black player
frame, scrubber and controls breathing into the chrome bars below the
video, info strip (title, episode label, year, runtime) under the
controls.

## A native desktop client (early)

This one is the biggest *shape* change of the batch. There's now a
Tauri 2 desktop client living at `apps/mythos-desktop/`, sharing the
exact same SvelteKit codebase as the embedded server SPA. Same
routes, same components, same `Player.svelte` — runtime backend
selection in `playback.ts` checks `'__TAURI_INTERNALS__' in window`
and dispatches to *libmpv via IPC* instead of `<video>` + hls.js.

The shape:

- **One UI, two playback backends.** Browser builds keep using
  `<video>` + `media-chrome` + hls.js. The Tauri build forwards every
  `play / pause / seek / setVolume / loadUrl` to libmpv running in
  the Rust host process via a Tauri IPC command surface — and mirrors
  mpv's position / paused state back into the same Svelte stores the
  browser backend writes to, so `Player.svelte` doesn't branch.
- **Wire contract is `mythos_core::playback`.** `PlaybackRequest`
  and `PlaybackDecision` are the same types the future Jellyfin
  shim will speak, so the desktop client and the shim are isomorphic
  from the server's point of view.
- **libmpv embeds into the Tauri window.** The setup hands libmpv the
  window's raw handle (via `raw-window-handle`'s `WindowHandle::as_raw`)
  and sets mpv's `wid` property. Mpv renders with `alpha=yes` +
  `background=none`, and the SPA reports its `<video>` bounding rect
  back to Rust so mpv's `video-margin-ratio-{l,r,t,b}` carve out the
  exact rectangle the SvelteKit chrome wants — the webview shows
  through everywhere else.
- **mpv's event loop runs on its own thread**, with a *separate*
  `EventContext::new(mpv.ctx)` (the libmpv2 test pattern), because
  `Mpv::event_context_mut` wants `&mut Mpv` and IPC commands need
  `Arc<Mpv>`.
- **Dev links system libmpv via pkg-config; release ships
  `libs/<platform>/libmpv.*`.** See
  `apps/mythos-desktop/README.md` in the source tree.

It's a scaffold — usable, but rough. Two known limits today:

- **The embed surface is the top-level Tauri window.** mpv's child
  surface overlays the SvelteKit chrome during playback; a child
  native widget *below* the webview (so chrome and video coexist
  natively) is the next milestone.
- **Wayland sessions that report `wl_surface` handles aren't
  supported by the current attach path.** There's a
  `Mpv::enable_standalone_window` fallback that flips
  `force-window=yes` so video still plays — just in a separate
  window from the chrome.

The motivation is the same as Phase 6's Jellyfin shim: meet people
where their devices already are. The Tauri client is the first
non-browser surface on the same UI codebase.

## TMDb year-in-title rescue

A small-but-satisfying fix. Files whose *title* contains a year were
getting misidentified — `Blade Runner 2049 (2017)/...` was scanning
as "Blade Runner 2049 → search 2049" instead of "Blade Runner 2049
→ search 2017". Same story for `Cyberpunk 2077 (2020)`,
`2001 A Space Odyssey 1968`, anything where a year token is part of
the name rather than a release-year tag.

`mythos_scan::identify_movie` now:

- **Prefers `(YYYY)`-bracketed years** anywhere over bare year tokens
  — so `Blade Runner 2049 (2017)` parses as `title="Blade Runner 2049",
  year=2017`.
- **Greedy-matches the *last* year token** in the stem when there's
  no paren form — so `2001 A Space Odyssey 1968` parses as
  `title="2001 A Space Odyssey", year=1968` rather than the other
  way round.

`enrich_pass` also retries the TMDb search *without* the year hint
when the first attempt misses — which rescues year typos too
(`Independence Day 1966` finds the 1996 film instead of returning
nothing).

## What's next

Still Phase 3: music, photos, books. Phase 6 (the Jellyfin-API
compatibility shim) waits for that to land. The desktop client gets
its child-native-widget embed surface and proper Wayland support as
follow-ups in parallel.

Source on [GitLab](https://gitlab.com/darkspar/mythos).
