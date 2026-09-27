# Magnet Player V1 Requirements

Status: **Frozen V1 baseline**

## Product goal

Magnet Player is an Android 16-only application that combines:

- a multi-tab WebView browser;
- persistent normal browsing and isolated private browsing;
- ABP-compatible ad blocking with selectable/updatable lists;
- automatic interception of `magnet:` links;
- BitTorrent streaming optimized for fast seek recovery;
- explicit full-download mode;
- Media3 playback;
- HTTP/HTTPS download management.

Primary UX flow:

```
Browse web
-> tap magnet:
-> intercept
-> fetch torrent metadata
-> select playable video
-> stream
-> seek quickly, including uncached positions
```

The primary performance metric is **seek request to first rendered frame**.

## Platform

- Android 16 / API 36 only
- minSdk = 36
- targetSdk = 36
- compileSdk = 36

No legacy Android compatibility work is required.

## Browser

Must support:

- address bar/search;
- back/forward/refresh;
- multi-tab browsing;
- bookmarks;
- browsing history;
- favicon;
- persistent cookies/login state in normal mode;
- session restoration of normal tabs;
- private mode;
- magnet interception from normal navigation, target=_blank, window.open, and external intents.

Normal mode persists:

- cookies;
- login state;
- LocalStorage;
- IndexedDB;
- history;
- normal tabs.

Private mode:

- uses AndroidX WebKit Multi-Profile;
- never persists private tabs;
- keeps cookies/storage/service workers isolated from the normal profile;
- deletes private profile data when the private session ends;
- if Multi-Profile is unsupported, tell the user to update Android System WebView rather than silently degrading privacy.

## Ad blocking

Use **adblock-rust** behind a Kotlin facade/JNI boundary.

Required capabilities:

- network filtering;
- cosmetic filtering;
- scriptlets;
- exceptions;
- subscriptions;
- My Filters;
- per-site allowlist.

Default subscriptions:

- EasyList: ON
- EasyPrivacy: ON
- regional lists: OFF
- Annoyances: OFF
- Social: OFF
- Acceptable Ads: OFF

Users can:

- enable/disable lists;
- add custom subscription URLs;
- remove custom subscriptions;
- add My Filters;
- whitelist individual sites;
- update one list or all lists;
- see rule count/update time/update status.

Filter updates must build a replacement engine and use atomic swap. If update fails, keep using the previous working engine.

## Magnet handling

All magnet entry points must converge on a single `MagnetRouter`.

Sources:

- internal WebView navigation;
- target=_blank / window.open;
- external Android VIEW intents.

## Torrent engine

Use libtorrent4j.

Required core capabilities:

- magnet metadata;
- DHT;
- PEX;
- LSD;
- trackers;
- peers;
- piece verification;
- resume data;
- proxy support.

## Torrent modes

### Streaming Mode

- downloads only what playback currently needs;
- uses dynamic piece scheduling;
- uses Streaming Cache;
- cache may be evicted automatically;
- does not guarantee the full media file is present.

### Download Mode

- user explicitly requests a complete file;
- complete file is preserved;
- not part of Streaming Cache LRU;
- already verified pieces from Streaming Mode must be reused.

## Upload / seeding

Default values:

- upload: OFF
- seeding: OFF
- seed after download: OFF

Settings may expose:

- allow upload;
- seed after download;
- Wi-Fi-only upload;
- upload speed limit.

No upload traffic should occur unless the user explicitly enables it.

## Streaming architecture

The internal playback path is fixed:

```
Media3
-> TorrentDataSource
-> PieceMapper
-> SeekCoordinator
-> PieceScheduler
-> libtorrent4j
```

Do not use localhost HTTP as the primary internal playback path.

## Player

Use Media3 / ExoPlayer for V1.

V1 does not implement mpv/libass/full ASS effects.

Player abstraction must leave room for future alternate engines.

Required features:

- playback speed;
- audio track selection;
- subtitle track selection;
- immersive landscape;
- PiP;
- background playback;
- MediaSession / lockscreen controls;
- floating playable-file menu for multi-file torrents.

## Playable file selection

Candidate extensions:

- mkv
- mp4
- webm
- m4v
- ts
- m2ts

De-prioritize filenames containing:

- sample
- trailer
- preview
- extras
- bonus

Rules:

- one obvious main video -> auto-play;
- one clearly largest main video -> auto-play;
- episode set / ambiguous collection -> show list;
- setting "auto select largest video" defaults ON.

The player must always offer a floating file list when multiple playable files exist.

## Seek behavior

Continuous gestures must only update preview position.

Real seek occurs once, on release.

Applies to:

- seek bar drag;
- horizontal scrub gesture;
- left/right long-press fast rewind/fast forward.

Seek generation rules:

- every real seek creates a new generation;
- previous generation waiters/urgency are cancelled;
- only the latest generation can continue to affect scheduling.

Initial priority model:

- P0 Seek Landing Zone
- P1 Immediate Playback
- P2 Forward Buffer
- P3 Back Buffer / warm cache
- P4 Background

Initial targets:

- Landing Zone = max(8 MB, about 2-4 seconds of media)
- Forward Buffer about 90 seconds
- Back Buffer about 45 seconds

These must be adaptive to bitrate, actual torrent speed, piece availability and buffer health.

## Seek performance targets

Instrument:

- seekRequestedAt
- targetPieceRequestedAt
- targetPieceReadyAt
- decoderReadyAt
- firstFrameRenderedAt

Target engineering goals:

- cached seek < 200 ms
- back-buffer seek < 300 ms
- uncached seek on excellent swarm < 1.5 s
- uncached seek on normal healthy swarm < 3 s

These are engineering goals under sufficient peer/network conditions, not unconditional guarantees.

## Player gestures

- single tap: show/hide controls;
- double tap left: -10 s;
- double tap right: +10 s;
- long press left: continuously accumulate rewind target;
- long press right: continuously accumulate forward target;
- horizontal drag: preview seek;
- left vertical drag: brightness;
- right vertical drag: volume.

Long-press and drag gestures perform one actual seek on release.

## Streaming Cache

Default: 10 GB.

Settings must provide a free-form numeric maximum in GB.

Eviction: LRU plus protection.

Protection priority:

1. current playback;
2. header/tail/index;
3. current playback window;
4. back buffer;
5. forward buffer;
6. recent cache;
7. old cache.

After playback exit, cache should be preferentially retained for 24 hours, then become normal LRU candidate.

Provide:

- per-torrent cache usage;
- clear one torrent cache;
- clear all streaming cache.

## Download storage

Default user-visible destination:

`Downloads/Magnet Player/downloads`

On first full download, offer:

- use default location;
- choose another directory.

Custom directories use SAF.

Download files:

- are not counted toward Streaming Cache;
- are never auto-deleted;
- support Play / View Folder / Share / Info / Delete.

## HTTP/HTTPS downloads

Use a dedicated browser download manager, separate from torrent download internals.

Required:

- pause;
- resume;
- cancel;
- HTTP Range resume where supported;
- progress;
- speed;
- ETA;
- open;
- view folder;
- share;
- delete.

Torrent and HTTP download tasks may share a unified UI, but not the same engine.

## Proxy

Browser proxy and Torrent proxy are configured separately.

Browser default: follow system proxy.

Torrent default: OFF.

Torrent proxy supports at least:

- SOCKS5
- HTTP
- host
- port
- optional username/password

Strict torrent proxy mode must never silently fall back to direct connections.

Provide a proxy connection test.

## Slow / unavailable torrent UX

Metadata has no hard failure timeout.

After ~30 seconds, show a "taking longer than expected" state while continuing to search.

Peers = 0 does not mean "dead torrent"; keep searching and show a clear status.

If download throughput is below estimated playback requirement, show warning with:

- actual speed;
- video requirement;
- remaining buffer.

Do not stop automatically unless buffer reaches zero.

If an uncached seek cannot obtain target data promptly, offer "Return to previous playback position".

## Playback history

Key: `infoHash + fileIndex`.

Store:

- position;
- duration;
- lastPlayedAt;
- completed.

Rules:

- playback < 30 s does not enter Continue Watching;
- >95% watched or <3 minutes remaining -> completed.

Home screen exposes Continue Watching sorted by recency.

## Resume / recovery

Persist:

- infoHash;
- magnet URI;
- fileIndex;
- mode;
- save location;
- playback position;
- selected audio/subtitle;
- libtorrent resume data;
- task state;
- last active time.

On app restart:

- normal browser tabs restore;
- streaming session state may restore but must not autoplay audio;
- unfinished downloads resume by default;
- strict-proxy tasks wait if proxy is unavailable.

## V1 exclusions

Do not implement in V1:

- mpv;
- libass/full ASS;
- Chromecast;
- DLNA;
- Android TV UI;
- accounts/cloud sync;
- online media aggregation;
- TMDB/IMDb/Douban metadata;
- RSS;
- browser extension system;
- self-maintained Chromium;
- WebDAV/SMB/NAS;
- cloud torrenting;
- multi-device progress sync;
- media poster wall.
