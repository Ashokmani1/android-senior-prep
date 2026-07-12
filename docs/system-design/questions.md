# Practice Prompts

Twelve mobile system design prompts with rapid model outlines. For each: **how to approach** (mapped to the [6-step framework](framework.md)), the **key tradeoffs to raise**, and the **red flag** — what a weak candidate forgets, almost always *offline*, *cancellation*, or *failure modes*.

!!! tip "Drill method"
    Cover the answer, give yourself 60 seconds to outline aloud, then compare. You're training the *reflex* to clarify-before-designing and to always land on failure modes.

---

## 1. Design an offline-capable expense tracker

- **Approach:** Clarify — offline writes yes, single-user, multi-device sync, categories/receipts. Model — expenses with client UUID + `syncStatus` + `updatedAt`; receipt images by URL. Layer — Room SSOT, repo exposes `Flow`. Sync — outbox pattern, WorkManager on `CONNECTED`, delta pull. Cross-cutting — encrypt financial data at rest. Failure — retry vs rollback, tombstone deletes.
- **Tradeoffs:** LWW vs server-authoritative conflict; storing receipt images (disk quota + eviction) vs URLs; eager vs deferred sync (battery).
- **Red flag:** Treats it as online-only CRUD; no write queue; hard-deletes rows so deletions don't propagate across devices.

## 2. Design Instagram's feed

- **Approach:** Ranked infinite scroll, server-side ranking, works offline from cache. Paging 3 + `RemoteMediator` + Room SSOT; keyset cursor pagination; pull-to-refresh = transactional cache clear; prefetch via `prefetchDistance`; images via a dedicated pipeline. See [chat & feed case](case-chat-feed.md).
- **Tradeoffs:** Cache-then-network (stale flash) vs network-first (blank screen); how much to prefetch (data/battery vs smoothness); memory vs disk cache sizing for media.
- **Red flag:** Offset pagination; re-ranking on the client; loading full-res images (OOM); no cursor/`RemoteMediator` so no offline scroll.

## 3. Design a chat / messaging app

- **Approach:** Message states (SENDING→SENT→DELIVERED→READ→FAILED); client-generated IDs for local echo + idempotency; `serverSeq` for ordering; hybrid WebSocket (foreground) + FCM (background) + polling fallback; keyset history pagination; reconnect backfill via last `serverSeq`; DB-derived unread counts. See [chat & feed case](case-chat-feed.md).
- **Tradeoffs:** WebSocket latency vs FCM battery/reliability; ordering by server sequence vs client time; group fan-out on server vs client.
- **Red flag:** Ordering by device timestamp; no idempotency (duplicate messages on retry); assumes the socket never drops (no reconnect/backfill).

## 4. Design a file upload with resume

- **Approach:** Clarify — large files, flaky network, must survive process death. Chunked/multipart upload; server issues an upload session ID; track uploaded byte offset in Room; on resume query server offset (or send `Content-Range`) and continue; WorkManager foreground service for large uploads with a notification; verify checksum on completion.
- **Tradeoffs:** Chunk size (more chunks = more overhead but finer resume granularity); foreground service (reliable, user-visible) vs deferred WorkManager (battery-friendly, may delay); client vs server checksum.
- **Red flag:** Restarts the whole upload on any failure; holds progress only in memory (lost on process death); no cancellation; blocks the UI thread.

## 5. Design a location-tracking ride app

- **Approach:** Clarify — background tracking, accuracy vs battery, offline buffering. `FusedLocationProvider` with adaptive intervals; batch + buffer points in Room when offline; upload in batches via WorkManager; foreground service with notification for active trips; downsample points (distance/time filter) to save battery and bandwidth.
- **Tradeoffs:** GPS accuracy vs battery drain (high-accuracy vs balanced); update frequency; batch upload interval (freshness vs radio wakeups); background execution limits (Doze, foreground-service requirement).
- **Red flag:** Requests high-accuracy GPS continuously (kills battery); no offline buffering (points lost in tunnels); ignores background execution/permission limits; uploads every single point individually.

## 6. Design a client-side caching layer

- **Approach:** Two-tier — memory (LRU, bytes-bounded) + disk (SQLite/DiskLruCache); cache-then-network read policy; invalidation via TTL + `etag`/`updatedAt` conditional requests; explicit invalidation on mutation; eviction by size + age. DB as SSOT so the cache *is* the read source.
- **Tradeoffs:** TTL length (freshness vs hit rate); write-through vs write-back; memory footprint vs hit rate; per-entity vs global invalidation.
- **Red flag:** "I'll just cache it" with no eviction or invalidation story; unbounded cache; no staleness handling; cache and network as separate UI-visible sources.

## 7. Design a music/podcast streaming client

- **Approach:** Clarify — streaming vs offline downloads, gapless playback, background audio. `MediaSessionService` + ExoPlayer/Media3; adaptive bitrate; prefetch/buffer next track; downloaded episodes in disk cache with quota + eviction; download queue via WorkManager; resume playback position persisted to DB.
- **Tradeoffs:** Buffer size (startup latency vs rebuffering); download quality vs storage; prefetch aggressiveness (data cost vs gapless UX); DRM complexity.
- **Red flag:** No buffering/prefetch strategy; downloads not resumable; playback position lost on process death; no storage quota/eviction for downloads.

## 8. Design an e-commerce product catalog + cart

- **Approach:** Catalog — paged (keyset), cached in Room, images via pipeline. Cart — **optimistic** local writes synced to server; must survive process death and reconcile price/stock at checkout. Server-authoritative on price/inventory; client cart is a wish, server confirms.
- **Tradeoffs:** Optimistic cart vs server confirmation (stale price/stock); how long to cache catalog (freshness of price); guest cart (local) vs synced cart (account).
- **Red flag:** Trusts client-side price/stock; cart only in memory; no reconciliation at checkout when an item went out of stock or changed price.

## 9. Design a notes app with rich media

- **Approach:** Offline-first (see [offline-first case](case-offline-first.md)): Room SSOT, outbox, delta sync, tombstones, optimistic writes. Media — store blobs on disk keyed by note, upload separately with resume, reference by local path until synced then swap to URL.
- **Tradeoffs:** Conflict resolution (LWW vs versioning vs CRDT for concurrent edits); inline media storage (disk quota) vs lazy download; text sync vs media sync decoupling.
- **Red flag:** No offline write path; conflict resolution that silently loses an edit; blocking note save on media upload; hard deletes.

## 10. Design a news/article reader with offline reading

- **Approach:** Clarify — save-for-offline, prefetch on Wi-Fi. Feed paged + cached; "save" downloads full article + images to disk; background prefetch of top articles constrained to unmetered network + charging; TTL-based cache invalidation; reading position persisted.
- **Tradeoffs:** Prefetch scope (data/storage vs instant reads); TTL (freshness vs offline availability); text-only vs full-media offline save.
- **Red flag:** Prefetches on metered connection (burns data); no storage cap; no offline story despite "reader" implying commute use; ignores `NetworkType.UNMETERED` constraint.

## 11. Design analytics/event tracking SDK

- **Approach:** Clarify — must not block UI, must survive process death, batch to save battery. Enqueue events to a Room buffer synchronously-but-off-main; batch upload via WorkManager with backoff; dedupe with event IDs; cap buffer size with drop policy; flush on app background.
- **Tradeoffs:** Batch size/interval (freshness of data vs battery/network); at-least-once (dupes, needs idempotent IDs) vs at-most-once (data loss); buffer cap drop policy (oldest vs newest).
- **Red flag:** Sends an HTTP call per event (battery/network disaster); events only in memory (lost on crash — ironic for an analytics SDK); blocks the main thread; no retry/backoff.

## 12. Design a collaborative document editor (mobile client)

- **Approach:** Clarify — real-time multi-user editing, offline edits must merge. This is the one prompt that genuinely needs **CRDTs or OT** rather than LWW. Local model as CRDT, sync ops over WebSocket, buffer ops offline in an outbox and replay on reconnect, presence via ephemeral channel, DB persists the CRDT state.
- **Tradeoffs:** CRDT (convergent, no central authority, heavier state) vs OT (needs server transform, lighter payload); op granularity (character vs block); presence cost.
- **Red flag:** Proposes last-write-wins for concurrent text editing (data loss / clobbered paragraphs); no offline op buffering; assumes always-connected; ignores reconnect replay ordering.

## 13. Design a file downloader library

- **Approach:** Clarify — large file downloads, pause/resume, concurrency, priority queue, background downloading. Model — download tasks in Room (URL, local path, status: ENQUEUED/DOWNLOADING/PAUSED/COMPLETED/FAILED, progress bytes, checksum). Concurrency — ThreadPoolExecutor with bounds (max 3 concurrent downloads). Resume — use HTTP `Range: bytes=X-` header; download file to a temporary file (`.tmp`) and append bytes, rename to final path only on completion and checksum verify. Queue — priority queue sorted by user priority + queue time. WorkManager for background execution.
- **Tradeoffs:** Multi-connection chunked downloading (faster speed but higher socket/server overhead and tricky segment merging) vs single-connection resume (slower but robust and simple); database syncing frequency for progress (real-time slows UI; throttle progress updates to once per 500 ms).
- **Red flag:** Memory-buffers the entire file (causes OOM); starts from byte 0 on every connection drop; no connection limits (spawns 50 threads and crashes the app); blocks the UI thread.

## 14. Design Instagram Stories (mobile client)

- **Approach:** Clarify — 24h lifetime, multi-segmented slides (images/videos), low latency transition, viewing state sync. Cache — segmented LRU disk cache; prefetch metadata (JSON lists of story URLs) and proactively download the next 1-2 slides ahead of the current active story. Player — ExoPlayer instances pooled (2-3 instances) for instant video play; images loaded via a memory-cached pipeline. Sync — send view events (story ID, timestamp) to server via an optimistic batch queue (similar to analytics SDK).
- **Tradeoffs:** Video prefetching depth (too deep wastes network/battery if user exits early; too shallow causes buffering spinner); pre-rendering next slides vs memory consumption.
- **Red flag:** Loads and initializes a fresh Player per slide (causes jank/1-second lag during slide transitions); does not prefetch media; downloads stories on metered data without constraints.

## 15. Design a location-based "Nearby Friends" app

- **Approach:** Clarify — real-time updates, geofencing, battery efficiency, scalability. Client — periodic location updates using `FusedLocationProviderClient` with a dynamic interval (slower when stationary via accelerometer/activity recognition, faster when moving). Spatial indexing — geohashing (e.g. 6-character Geohash, ~1.2km precision) to represent user location. Networking — upload Geohash to server, query server for matching geohashes of active friends. Map rendering — cluster points locally to avoid layout overload.
- **Tradeoffs:** Location accuracy vs battery (GPS vs network triangulation); polling intervals vs WebSockets for real-time friend movements; client-side vs server-side distance calculation.
- **Red flag:** Continuously queries GPS at maximum frequency (battery dies in 2 hours); uploads precise GPS coordinates to the server every second (privacy leak and network overhead); uses O(N) distance checks for thousands of users on the client.

## 16. Design a client-side logging library

- **Approach:** Clarify — high throughput, minimal overhead, persistence, remote upload, secure. API — `Log.d`, `Log.e` routing. Persistence — enqueues logs to an in-memory ring buffer, flushed asynchronously in batches to a file using `BufferedWriter` on a single background worker thread to prevent disk write contention. Rotation — split files when they exceed size limit (e.g. 5 MB) or age (e.g. 24h), keeping a max of 5 files. Security — encrypt log contents on the fly using AES-GCM (key stored in Android Keystore). Upload — compress logs (GZIP) and upload via WorkManager during Wi-Fi + charging.
- **Tradeoffs:** SQLite DB (structured queryable logs but high write/CPU overhead) vs flat files (fast append, low overhead but harder to query); instant file syncing (no data loss but high disk wear) vs buffered flush.
- **Red flag:** Synchronous file writing on the logging thread (causes UI frames to drop on the main thread); unbounded file growth (fills up device storage); writes sensitive customer data in plaintext.

## 17. Real-time updates: HTTP Polling vs. Long-Polling vs. WebSockets vs. Server-Sent Events (SSE)

- **Approach:** Clarify the protocol characteristics for mobile:
    *   **Short Polling**: Client periodically sends HTTP requests. *Characteristics*: High battery and network overhead due to constant TCP handshakes/HTTP headers. *Use Case*: Low-frequency updates where delay is acceptable.
    *   **Long Polling**: Client sends request, server holds it open until new data is available. *Characteristics*: Reduces latency, but connection drops require constant reconnection overhead.
    *   **WebSockets**: Bi-directional, full-duplex TCP persistent connection. *Characteristics*: Lowest latency, low overhead, but requires keeping a TCP socket open (drains battery) and custom reconnect/heartbeat logic.
    *   **SSE**: Mono-directional (server-to-client) persistent HTTP connection. *Characteristics*: Reuses standard HTTP/2, auto-reconnects, but is read-only.
- **Tradeoffs:** WebSockets (great for chat, bi-directional, high battery footprint) vs SSE (great for live tickers/stocks, mono-directional, standard HTTP compatibility); WebSocket background battery drain vs FCM push notifications (wake radio only on data).
- **Red flag:** Leaves a WebSocket connection open continuously when the app is in the background (Google Play console battery warning); does not implement heartbeats/pings to detect silent connection drops.

## 18. How Voice & Video Calling works (WebRTC on Mobile)

- **Approach:** Clarify WebRTC architecture for mobile:
    *   **Signaling**: App exchanges SDP (Session Description Protocol) offer/answer and ICE candidates (IPs/ports) via a signaling channel (e.g. WebSocket or FCM push).
    *   **NAT Traversal**: Uses **STUN** (queries public IP/port) or **TURN** (relays media stream if direct peer-to-peer connection fails due to symmetric NAT).
    *   **Media Pipeline**: `AudioRecord` + `Camera2` capture raw frames → encoded via hardware codecs (H.264/VP8, Opus) → packetized and transmitted via RTP/SRTP over UDP.
    *   **State**: Exposes state updates (CONNECTING→CONNECTED→DISCONNECTED→FAILED).
- **Tradeoffs:** Peer-to-peer (no server media cost, low latency, but leaks client IP) vs SFU/MCU media servers (server-routed, scales to group calls, shields IPs, but high server cost).
- **Red flag:** Uses TCP for video/audio transmission (causes massive latency and lag due to head-of-line blocking); does not implement a TURN fallback (calls fail on most mobile carrier networks).

## 19. How "Where Is My Train" tracks train location without Internet

- **Approach:** Clarify off-grid tracking mechanisms:
    *   **Cell Tower Triangulation**: The app queries the telephony API (`TelephonyManager.getAllCellInfo()`) for the current Cell Tower ID (MCC, MNC, LAC, CID). It queries an **offline SQL database** packaged inside the APK containing the GPS coordinates of all railway-line cell towers.
    *   **GPS Satellites**: The mobile GPS receiver decodes signals from GPS satellites directly (requires no cellular network or internet) to obtain coordinates.
    *   **Offline Schedule Reconciliation**: Matches coordinates and speed against an offline timetable database (GTFS-like) to extrapolate the current station and delays.
- **Tradeoffs:** Cell ID tracking (extremely low battery, works deep inside the train compartment, but lower accuracy) vs GPS (high accuracy, but high battery drain and fails inside metallic train roofs).
- **Red flag:** Assumes location lookup requires a network geocoding API; queries GPS continuously at max frequency (burns battery); fails to package the location/cell databases offline.

---

!!! quote "The pattern across all nineteen"
    Every strong answer does the same four things: **clarifies offline/realtime/scale first**, makes the **local DB the single source of truth**, uses **cursors and optimistic writes with a sync queue**, and **closes on failure modes** (network loss mid-write, process death, cancellation). Every weak answer forgets offline, forgets cancellation, or designs only the happy path.
