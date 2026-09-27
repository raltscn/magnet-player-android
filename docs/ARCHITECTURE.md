# Architecture

## High-level view

```
┌──────────────────────────────────────┐
│                :app                 │
│ Compose / navigation / app wiring   │
└───────────────┬──────────────────────┘
                │
      ┌─────────┼──────────┐
      ▼         ▼          ▼
   Browser    Player    Downloads
      │         │          │
      ▼         ▼          ▼
   AdBlock   Streaming   Torrent Core
                │          │
                └────┬─────┘
                     ▼
                 libtorrent
```

## Core playback path

```
Media3
-> TorrentDataSource
-> PieceMapper
-> SeekCoordinator
-> PieceScheduler
-> libtorrent4j
-> verified data
-> Media3 decoder
```

This path is intentionally direct. A localhost HTTP server is not the main internal playback transport.

## Browser

Browser responsibilities:

- WebView lifecycle;
- tabs;
- navigation;
- bookmarks/history;
- normal/private profiles;
- magnet interception;
- browser proxy;
- handing downloads to the download domain.

The browser never directly manipulates torrent handles.

## MagnetRouter

All magnet sources converge on one domain entry point:

```
Internal WebView ----┐
target=_blank -------┤
window.open ----------> MagnetRouter -> Torrent orchestration
external VIEW intent ┘
```

## Torrent core

Torrent core owns:

- libtorrent session lifecycle;
- metadata state;
- file list;
- peer/tracker/DHT/PEX/LSD state;
- piece state;
- upload policy;
- torrent proxy;
- resume data.

## Streaming domain

### TorrentDataSource

Adapts Media3 byte-range reads to torrent-backed data.

### PieceMapper

Maps file-relative byte positions to global torrent offsets and piece ranges.

### SeekCoordinator

Owns seek generations and cancels superseded requests.

### PieceScheduler

Assigns urgency/deadlines/priorities according to current playback need.

### StreamingCacheManager

Owns cache limits, protection rules and eviction.

## Player

UI talks to a `PlayerEngine` abstraction rather than ExoPlayer directly.

V1 implementation: Media3/ExoPlayer.

Future engines must not force UI rewrite.

## Ad block

```
WebView / Service Worker request
-> Kotlin AdBlockEngine facade
-> JNI
-> adblock-rust
-> allow/block/cosmetic/scriptlet decision
```

Filter subscriptions are compiled into a replacement engine and swapped atomically.

## Storage

Streaming data and full-download output have different lifecycles.

Streaming cache:

- app-managed;
- LRU;
- protected current playback regions;
- default 10 GB, user-configurable.

Download output:

- explicit user action;
- durable;
- exported to Downloads/Magnet Player/downloads or SAF destination;
- never evicted by Streaming Cache.

## Persistence

Room stores structured user/task history.

DataStore stores preferences.

libtorrent resume data is persisted separately as part of torrent task recovery.

## Services

Long-lived torrent/download/playback activity must not be owned by an Activity/Composable.

Foreground service lifecycle should only exist while playback/download/seeding work actually requires it.

## State restoration

On restart:

- normal browser tabs restore;
- private tabs do not;
- streaming state may restore without autoplay;
- unfinished downloads resume by default;
- strict-proxy tasks stay paused if proxy is unavailable.
