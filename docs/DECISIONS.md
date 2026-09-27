# Architecture Decision Records

This file records accepted deviations or important architecture decisions.

---

## ADR-001 — Android 16 only

**Status:** Accepted

**Context:** The project intentionally targets modern Android only.

**Decision:** minSdk, targetSdk and compileSdk are all API 36 for V1.

**Consequence:** No legacy Android compatibility work is required.

---

## ADR-002 — Direct Media3 torrent data source

**Status:** Accepted

**Context:** Seek latency is a primary product goal.

**Decision:** The primary internal path is Media3 -> custom TorrentDataSource -> torrent piece scheduling.

**Rejected alternative:** localhost HTTP as the primary internal streaming layer.

**Consequence:** Media3 byte positions can immediately drive piece scheduling with less transport indirection.

---

## ADR-003 — Separate Streaming and Download modes

**Status:** Accepted

**Decision:** Streaming Mode is cache-oriented and evictable. Download Mode is explicit, complete and durable.

**Consequence:** Full downloads are never governed by Streaming Cache LRU.

---

## ADR-004 — Upload/seeding disabled by default

**Status:** Accepted

**Decision:** Torrent upload, seeding and post-download seeding default to OFF.

**Consequence:** Upload traffic only occurs after explicit user opt-in.

---

## ADR-005 — WebKit Multi-Profile private browsing

**Status:** Accepted

**Decision:** Private mode uses AndroidX WebKit Multi-Profile isolation.

**Rejected alternative:** sharing the normal profile and manually clearing cookies/storage afterward.

**Consequence:** If Multi-Profile is unavailable, private mode should fail closed and ask the user to update WebView.

---

## ADR-006 — Media3 for V1

**Status:** Accepted

**Decision:** V1 uses Media3/ExoPlayer.

**Deferred:** mpv/libass/full ASS effects.

**Consequence:** Player UI must use a PlayerEngine abstraction so a future engine can be added without UI rewrite.

---

## ADR-007 — Strict proxy must fail closed

**Status:** Accepted

**Decision:** In strict torrent proxy mode, proxy failure pauses torrent activity.

**Rejected alternative:** automatic fallback to direct connections.

**Consequence:** Privacy expectations take priority over availability.

---

## ADR-008 — Seek gestures commit on release

**Status:** Accepted

**Decision:** Continuous drag/long-press operations only update preview state. A single real seek occurs on release.

**Consequence:** Torrent piece scheduling does not thrash during gesture movement.

---

## ADR-009 — Ad blocking via adblock-rust

**Status:** Accepted

**Decision:** Use adblock-rust behind a Kotlin/JNI facade.

**Consequence:** Filter-list compatibility and high-frequency matching are handled by a dedicated filtering engine rather than simple URL blacklists.

---

## ADR-010 — Public download destination without broad storage permission

**Status:** Accepted

**Decision:** Use MediaStore/SAF for user-visible download output; do not request MANAGE_EXTERNAL_STORAGE for V1.

**Consequence:** V1 may use an app-controlled working area before exporting completed large files; storage remains abstract for future optimization.
