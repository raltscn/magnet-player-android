# AGENTS.md

## Scope

This repository implements **Magnet Player V1**, an Android 16 / API 36-only application combining:

- a WebView browser;
- ABP-compatible ad blocking;
- magnet URI interception;
- BitTorrent streaming and full-download modes;
- Media3 playback optimized for fast seek recovery.

The product and architecture baseline is frozen in:

- `docs/V1_REQUIREMENTS.md`
- `docs/ENGINEERING_GUIDE.md`
- `docs/ARCHITECTURE.md`
- `docs/DECISIONS.md`

Read all four before changing architecture or implementing a new phase.

## Highest priority

The most important technical metric is:

`Seek request -> first rendered frame`

Do not optimize total torrent throughput at the expense of seek responsiveness, playback continuity, data correctness, proxy privacy, or crash recovery.

## Fixed platform

- Android 16 only
- minSdk = 36
- targetSdk = 36
- compileSdk = 36
- Kotlin
- Jetpack Compose / Material 3
- Android System WebView + AndroidX WebKit
- Media3 / ExoPlayer
- libtorrent4j
- adblock-rust through JNI
- Room
- DataStore
- WorkManager

Use stable dependencies unless a stable API is demonstrably insufficient.

## Frozen architecture rules

1. Internal playback path must remain:
   `Media3 -> TorrentDataSource -> PieceMapper -> SeekCoordinator -> PieceScheduler -> libtorrent`.
2. Do not replace the main playback path with localhost HTTP streaming.
3. Streaming Mode and Download Mode are separate modes with separate lifecycle/storage semantics.
4. Upload and seeding are disabled by default.
5. Private browsing must use WebKit Multi-Profile isolation.
6. Strict torrent proxy mode must never silently fall back to direct connections.
7. Continuous seek gestures update preview only; perform one real seek on release.
8. Full downloads are not part of Streaming Cache LRU.
9. V1 does not implement mpv, libass, full ASS effects, Chromecast, DLNA, Android TV UI, cloud sync, RSS, WebDAV, SMB, NAS, cloud torrenting, or browser extensions.

## Dependency direction

Preferred direction:

```
UI
  -> controller / repository / domain API
    -> engine
      -> native / storage
```

Forbidden shortcuts include:

- Compose directly controlling TorrentHandle;
- WebView directly controlling libtorrent;
- Player UI directly accessing Room DAO;
- ad-block UI directly calling JNI functions.

## Work order

Implement in this order unless a BLOCKER requires otherwise:

1. Phase 0: project foundation
2. Phase 1: torrent metadata core
3. Phase 2: streaming core + Media3
4. Phase 3: seek instrumentation and optimization
5. Phase 4: browser + magnet interception + Multi-Profile
6. Phase 5: tabs/bookmarks/history/session restore
7. Phase 6: ad blocking + filter manager
8. Phase 7: Download Mode + HTTP downloads
9. Phase 8: proxy/resume/recovery
10. Phase 9: player gestures/PiP/MediaSession and UI polish

Do not spend significant effort on visual polish before the streaming/seek path is reliable.

## BLOCKER protocol

If the specification cannot be implemented as written, do not silently change the architecture.

Report:

1. original requirement;
2. verified technical limitation;
3. why the current design fails;
4. option A;
5. option B;
6. impact of each option;
7. recommended change.

Record accepted architectural changes in `docs/DECISIONS.md`.

## Completion report

For each coherent task or phase, report:

- Implemented
- Changed files/modules
- Tests
- Performance / metrics
- Known limitations
- Architecture notes
- Next recommended task

Never report "done" without build/test evidence.

## Testing expectations

At minimum, cover:

- PieceMapper boundaries and >4 GB offsets;
- rapid seek generation cancellation;
- cache protection/eviction;
- normal/target-blank/window.open/external magnet interception;
- private-profile isolation;
- ad-block block/allow/cosmetic/scriptlet/update failure;
- HTTP range resume;
- strict-proxy failure without direct fallback;
- session/resume recovery.

Use legal, deterministic torrent fixtures for automated tests.

## Security and privacy

- Never log cookies, passwords, proxy credentials, Authorization headers, or sensitive browsing history in release builds.
- Avoid `addJavascriptInterface` unless explicitly required.
- Do not ignore HTTPS certificate errors by default.
- Sanitize download filenames and prevent path traversal.
- Validate external filter subscriptions before atomic engine swap.
- Treat all magnet URI and webpage inputs as untrusted.
