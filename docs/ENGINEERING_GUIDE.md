# Engineering Guide

## Purpose

This document defines how Magnet Player V1 should be implemented. Product behavior lives in `V1_REQUIREMENTS.md`; agent-level constraints live in `../AGENTS.md`.

## Recommended modules

```
:app
:core:model
:core:database
:core:settings
:browser
:adblock
:torrent
:streaming
:player
:downloads
```

Keep dependency direction one-way:

```
UI
-> controller/repository/domain API
-> engine
-> native/storage
```

Avoid service-locator globals and UI-owned engine lifecycle.

## Suggested implementation phases

### Phase 0 — foundation

- Gradle project and version catalog;
- module structure;
- DI choice;
- logging abstraction;
- domain error model;
- build variants;
- unit/instrumentation test setup;
- debug and release builds green.

Create/maintain:

- README.md
- AGENTS.md
- docs/ARCHITECTURE.md
- docs/DECISIONS.md

### Phase 1 — torrent metadata core

Implement only:

```
magnet
-> libtorrent session
-> metadata
-> file list
```

Support state observation and resume data.

### Phase 2 — streaming core

Implement:

- TorrentDataSource
- PieceMapper
- SeekCoordinator
- PieceScheduler
- StreamingCacheManager
- Media3 integration

First prove sequential playback from a single video file.

### Phase 3 — seek-first optimization

Instrument the full seek pipeline and validate cancellation of old generations.

No major browser UI work until this path is stable.

### Phase 4 — browser core

Add:

- WebView;
- toolbar/address/search;
- tabs;
- magnet interception;
- external intents;
- Multi-Profile private mode.

### Phase 5 — browser persistence

Add:

- bookmarks;
- history;
- tab restore;
- normal cookies/login persistence;
- private profile cleanup.

### Phase 6 — ad block

Add adblock-rust/JNI facade, network filtering, cosmetic filtering, scriptlets, subscriptions, My Filters, allowlist, update pipeline.

### Phase 7 — downloads

Add Download Mode and HTTP/HTTPS download manager.

### Phase 8 — proxy/resume/recovery

Add browser/torrent proxy settings, strict proxy validation, task recovery and crash-safe resume.

### Phase 9 — player/system polish

Add gestures, PiP, MediaSession, lockscreen controls, UI polish and broad performance testing.

## Threading

Main thread: UI state only.

IO dispatcher: network/storage.

Native access: centralized dispatcher/owner for libtorrent/JNI.

Do not allow uncontrolled multi-threaded mutation of the same torrent/session object.

## Errors

Use a typed domain model, not raw strings. Suggested categories:

- MetadataUnavailable
- NoPeers
- TargetPieceUnavailable
- NetworkUnavailable
- ProxyUnavailable
- DiskFull
- CacheLimitReached
- UnsupportedMedia
- TorrentCorrupted
- WebViewRendererGone
- FilterUpdateFailed

Map technical errors to user-friendly UI messages.

## Logging

Use categories:

- BROWSER
- TORRENT
- STREAM
- SEEK
- PLAYER
- ADBLOCK
- DOWNLOAD
- PROXY
- STORAGE

Release builds must never log cookies, passwords, proxy credentials, Authorization headers or sensitive browsing history.

## Metrics

Debug metrics should include:

- metadata acquisition duration;
- peer count;
- download throughput;
- video bitrate estimate;
- buffer duration;
- SeekToFirstFrame;
- piece request latency;
- piece availability;
- cache hit rate;
- cache size;
- ad-block match time.

## Native ownership

Every native resource must have a clear owner and explicit release strategy.

Do not rely on GC for libtorrent session/handle wrappers, Rust engine pointers or native buffers.

## Storage

Streaming cache should use app-controlled storage and support safe eviction.

Full downloads should ultimately appear in the user-visible Downloads destination using MediaStore/SAF without MANAGE_EXTERNAL_STORAGE.

Keep storage APIs abstract enough that a future direct-FD backend can replace the V1 export/copy path without changing higher layers.

## WebView

Implement `onRenderProcessGone`; renderer failure must not crash the whole app.

Avoid `addJavascriptInterface` unless a documented requirement needs it.

Do not bypass HTTPS certificate errors.

## Ad blocking

Keep JNI behind a Kotlin `AdBlockEngine` facade.

Filter update pipeline:

```
download
-> validate
-> build replacement engine
-> atomic swap
```

If build/update fails, retain the old engine.

Intercept both WebView and Service Worker requests where supported.

## Download manager

HTTP download tasks and torrent download tasks may share UI/state presentation, but their transfer engines remain separate.

Sanitize filenames and prevent path traversal.

## Test requirements

### PieceMapper

Cover:

- file starts at piece boundary;
- file starts mid-piece;
- range spans one piece;
- range spans many pieces;
- final piece;
- multiple torrent files;
- offsets >4 GB.

Use Long for byte offsets.

### SeekCoordinator

Cover:

- single seek;
- rapid A -> B -> C seeks;
- old generation cancellation;
- seek to cached target;
- seek to missing target;
- seek backward.

Only the newest generation may continue scheduling.

### Cache

Cover:

- LRU eviction;
- protected data;
- runtime cache-limit change;
- clear one torrent;
- clear all;
- disk full.

### Browser

Cover:

- normal URL;
- redirects;
- target=_blank;
- window.open;
- magnet href;
- external magnet;
- tab restore;
- normal cookie persistence;
- private profile isolation.

### AdBlock

Cover:

- blocking rule;
- exception;
- third-party rule;
- cosmetic filtering;
- scriptlet;
- site allowlist;
- subscription enable/disable;
- engine atomic swap;
- update failure.

### Proxy

Cover:

- valid proxy;
- invalid host;
- wrong credentials;
- timeout;
- strict mode;
- network switching;
- proxy disappearing mid-session.

Verify no direct fallback in strict mode.

### Downloads

Cover:

- Range resume;
- server without Range support;
- cancel;
- disk full;
- share;
- delete;
- SAF destination;
- MediaStore destination.

Use legal deterministic torrent fixtures in automated tests.
