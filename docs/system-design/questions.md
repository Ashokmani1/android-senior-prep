# Practice Prompts

Twelve mobile system design prompts with rapid model outlines. For each: **how to approach** (mapped to the [6-step framework](framework.md)), a short **example** where no deeper case study covers it, the **key tradeoffs to raise**, the **red flag** — what a weak candidate forgets, almost always *offline*, *cancellation*, or *failure modes* — and the **follow-ups** an interviewer chains once you finish the outline.

!!! tip "Drill method"
    Cover the answer, give yourself 60 seconds to outline aloud, then compare. You're training the *reflex* to clarify-before-designing and to always land on failure modes. When a follow-up lands, answer it the same way you answered the prompt — approach, tradeoff, failure mode — not with a one-word guess.

---

## 1. Design an offline-capable expense tracker

- **Approach:** Clarify — offline writes yes, single-user, multi-device sync, categories/receipts. Model — expenses with client UUID + `syncStatus` + `updatedAt`; receipt images by URL. Layer — Room SSOT, repo exposes `Flow`. Sync — outbox pattern, WorkManager on `CONNECTED`, delta pull. Cross-cutting — encrypt financial data at rest. Failure — retry vs rollback, tombstone deletes. Full worked pattern (entity, outbox, `SyncWorker`, conflict table): [offline-first case study](case-offline-first.md).
- **Tradeoffs:** LWW vs server-authoritative conflict; storing receipt images (disk quota + eviction) vs URLs; eager vs deferred sync (battery).
- **Red flag:** Treats it as online-only CRUD; no write queue; hard-deletes rows so deletions don't propagate across devices.
- **Follow-ups**
    - Two devices add the same expense offline before either syncs — what happens? *(client UUID is the idempotency key; both upserts land as the same row server-side, no duplicate — this is exactly why IDs are generated client-side, not server-assigned.)*
    - How do you show the user "this expense failed to sync" without blocking the UI? *(a `syncStatus = FAILED` flag on the entity the list already observes — render a small badge, never a blocking dialog; see the rollback rule of thumb in the case study.)*

## 2. Design Instagram's feed

- **Approach:** Ranked infinite scroll, server-side ranking, works offline from cache. Paging 3 + `RemoteMediator` + Room SSOT; keyset cursor pagination; pull-to-refresh = transactional cache clear; prefetch via `prefetchDistance`; images via a dedicated pipeline. Full `RemoteMediator` implementation: [chat & feed case](case-chat-feed.md#part-2-social-feed).
- **Tradeoffs:** Cache-then-network (stale flash) vs network-first (blank screen); how much to prefetch (data/battery vs smoothness); memory vs disk cache sizing for media.
- **Red flag:** Offset pagination; re-ranking on the client; loading full-res images (OOM); no cursor/`RemoteMediator` so no offline scroll.
- **Follow-ups**
    - User pulls to refresh mid-scroll at item 40 — what happens to their scroll position? *(`REFRESH` clears the table transactionally and repopulates from page 1; the list jumps to top by design — if that's a bad UX for this product, the alternative is prepending new items above the current position instead of a full clear, which trades simplicity for a harder merge.)*
    - How do you avoid showing the same post twice if the server's ranking shifts between page loads? *(dedupe by a stable server-assigned post ID at the DB layer — `INSERT OR IGNORE`/upsert on primary key — never rely on position staying stable across a re-rank.)*

## 3. Design a chat / messaging app

- **Approach:** Message states (SENDING→SENT→DELIVERED→READ→FAILED); client-generated IDs for local echo + idempotency; `serverSeq` for ordering; hybrid WebSocket (foreground) + FCM (background) + polling fallback; keyset history pagination; reconnect backfill via last `serverSeq`; DB-derived unread counts. Full state machine + transport table: [chat & feed case](case-chat-feed.md#part-1-chat-client).
- **Tradeoffs:** WebSocket latency vs FCM battery/reliability; ordering by server sequence vs client time; group fan-out on server vs client.
- **Red flag:** Ordering by device timestamp; no idempotency (duplicate messages on retry); assumes the socket never drops (no reconnect/backfill).
- **Follow-ups**
    - The app is killed while a message is `SENDING` — what happens on next launch? *(it's already a row in Room with `state = SENDING`; on launch, re-check with the server by `clientId` — if the server has it, mark `SENT`; if not, retry the send. Never lose it, because it was never memory-only.)*
    - How do you know a group chat's 500 members all got a message without polling each one? *(you don't track per-recipient delivery client-side at that scale — server fan-out owns delivery/read receipts in aggregate, and the client only renders what the server reports for the conversation, e.g. "delivered to N of 500.")*

## 4. Design a file upload with resume

- **Approach:** Clarify — large files, flaky network, must survive process death. Chunked/multipart upload; server issues an upload session ID; track uploaded byte offset in Room; on resume query server offset (or send `Content-Range`) and continue; WorkManager foreground service for large uploads with a notification; verify checksum on completion.

    ```kotlin
    @Entity data class UploadEntity(
        @PrimaryKey val id: String,           // client UUID
        val filePath: String,
        val sessionId: String?,               // assigned by server once negotiated
        val bytesUploaded: Long = 0,
        val totalBytes: Long,
        val status: UploadStatus = UploadStatus.PENDING,
        val checksum: String,                 // computed once, verified server-side on completion
    )

    // Resume: ask the SERVER what it has, don't trust the local offset alone —
    // a previous attempt may have landed bytes the client never got an ack for.
    suspend fun resume(entity: UploadEntity) {
        val serverOffset = api.getUploadOffset(entity.sessionId!!)   // authoritative
        uploadFrom(entity, offset = maxOf(entity.bytesUploaded, serverOffset))
    }
    ```

- **Tradeoffs:** Chunk size (more chunks = more overhead but finer resume granularity); foreground service (reliable, user-visible) vs deferred WorkManager (battery-friendly, may delay); client vs server checksum.
- **Red flag:** Restarts the whole upload on any failure; holds progress only in memory (lost on process death); no cancellation; blocks the UI thread.
- **Follow-ups**
    - Why ask the server for the offset instead of trusting the local `bytesUploaded`? *(the client can crash after the network write succeeds but before it persists the new offset locally — trusting a stale local offset re-sends bytes the server already has, or worse, skips bytes it doesn't; the server's view is authoritative.)*
    - The user backgrounds the app mid-upload on Android 12+ — does the upload survive? *(only if it's running as a foreground service with a visible notification, or deferred to `WorkManager`; a plain background thread/coroutine tied to an Activity dies with the process under memory pressure.)*

## 5. Design a location-tracking ride app

- **Approach:** Clarify — background tracking, accuracy vs battery, offline buffering. `FusedLocationProvider` with adaptive intervals; batch + buffer points in Room when offline; upload in batches via WorkManager; foreground service with notification for active trips; downsample points (distance/time filter) to save battery and bandwidth.

    ```kotlin
    val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, /*intervalMs=*/4_000)
        .setMinUpdateDistanceMeters(10f)     // don't fire on sub-10m jitter
        .setMaxUpdateDelayMillis(20_000)     // allow batching — fewer radio wakeups
        .build()
    // Points land in a Room buffer; a periodic WorkManager job drains it in one batched PUT
    // rather than one HTTP call per point.
    ```

- **Tradeoffs:** GPS accuracy vs battery drain (high-accuracy vs balanced); update frequency; batch upload interval (freshness vs radio wakeups); background execution limits (Doze, foreground-service requirement).
- **Red flag:** Requests high-accuracy GPS continuously (kills battery); no offline buffering (points lost in tunnels); ignores background execution/permission limits; uploads every single point individually.
- **Follow-ups**
    - The rider goes through a tunnel for 3 minutes — what does the driver's app show? *(buffered last-known point + a "signal lost" state derived from update staleness, not a frozen pin presented as live; on reconnect, the buffered points backfill the route without implying teleportation.)*
    - Why `setMaxUpdateDelayMillis` instead of just a longer interval? *(it lets the OS batch location callbacks with other radio wakeups on the device, which is cheaper than the app dictating its own wake cadence — same data, fewer radio-on events.)*

## 6. Design a client-side caching layer

- **Approach:** Two-tier — memory (LRU, bytes-bounded) + disk (SQLite/DiskLruCache); cache-then-network read policy; invalidation via TTL + `etag`/`updatedAt` conditional requests; explicit invalidation on mutation; eviction by size + age. DB as SSOT so the cache *is* the read source. Concrete cross-platform/secure variant with the disk-log + index design: [Rakuten's disk-based cache constraints](../prep-strategy/coding-rounds.md#disk-based-cache-constraints).
- **Tradeoffs:** TTL length (freshness vs hit rate); write-through vs write-back; memory footprint vs hit rate; per-entity vs global invalidation.
- **Red flag:** "I'll just cache it" with no eviction or invalidation story; unbounded cache; no staleness handling; cache and network as separate UI-visible sources.
- **Follow-ups**
    - Two screens request the same key at once and it's not cached — do you fire two network calls? *(no — coalesce in-flight requests by key, the same pattern as the [image pipeline's request dedup](case-image-pipeline.md#request-dedup-coalescing): late callers `await` the same in-flight `Deferred` instead of triggering a duplicate fetch.)*
    - How do you invalidate one entity without wiping the whole cache? *(key invalidation by entity ID/`updatedAt`, not a global `clear()` — a mutation should only evict the keys it actually affects, or you pay a network round-trip for everything else that was still fresh.)*

## 7. Design a music/podcast streaming client

- **Approach:** Clarify — streaming vs offline downloads, gapless playback, background audio. `MediaSessionService` + ExoPlayer/Media3; adaptive bitrate; prefetch/buffer next track; downloaded episodes in disk cache with quota + eviction; download queue via WorkManager; resume playback position persisted to DB.

    ```kotlin
    // Persist position on a throttled interval, not per-frame — writing to disk every
    // frame is wasted I/O; losing a couple seconds of resume precision is fine.
    playerScope.launch {
        while (isActive) {
            delay(5_000)
            dao.updatePosition(trackId, player.currentPosition)
        }
    }
    ```

- **Tradeoffs:** Buffer size (startup latency vs rebuffering); download quality vs storage; prefetch aggressiveness (data cost vs gapless UX); DRM complexity.
- **Red flag:** No buffering/prefetch strategy; downloads not resumable; playback position lost on process death; no storage quota/eviction for downloads.
- **Follow-ups**
    - Why is `MediaSessionService` the right host, not just an ExoPlayer instance inside an Activity? *(a `Service` outlives the Activity, so playback survives navigation and backgrounding; `MediaSession` additionally exposes standardized transport controls to the lock screen, Bluetooth headsets, and Assistant without you hand-wiring each surface.)*
    - Storage fills up mid-download of a new episode — what's the eviction policy? *(LRU by last-played time among *downloaded* (not currently-playing) episodes, same bytes-bounded-cache discipline as the image pipeline — never evict the episode actively playing, and warn before evicting something the user explicitly pinned for offline.)*

## 8. Design an e-commerce product catalog + cart

- **Approach:** Catalog — paged (keyset), cached in Room, images via pipeline. Cart — **optimistic** local writes synced to server; must survive process death and reconcile price/stock at checkout. Server-authoritative on price/inventory; client cart is a wish, server confirms.

    ```kotlin
    // The client cart is a proposal, not a commitment — checkout re-validates every line.
    suspend fun checkout(cart: List<CartLine>): CheckoutResult {
        val validated = api.validateCart(cart.map { it.toRequest() })  // server re-prices/re-stocks
        return when {
            validated.allAvailableAtQuotedPrice -> api.placeOrder(validated)
            else -> CheckoutResult.NeedsConfirmation(validated.changes)  // surface diffs to the user
        }
    }
    ```

- **Tradeoffs:** Optimistic cart vs server confirmation (stale price/stock); how long to cache catalog (freshness of price); guest cart (local) vs synced cart (account).
- **Red flag:** Trusts client-side price/stock; cart only in memory; no reconciliation at checkout when an item went out of stock or changed price.
- **Follow-ups**
    - An item's price drops between "add to cart" and "checkout" — what does the user see? *(surface the diff explicitly — "price changed from $X to $Y, still add?" — rather than silently charging the new price or silently keeping the stale one; both silent options erode trust.)*
    - User is logged out with items in a guest cart, then logs in — how do you merge? *(merge by product ID with a deterministic rule, typically "union quantities, cap at stock," and let the user review the merged cart rather than silently overwriting one side.)*

## 9. Design a notes app with rich media

- **Approach:** Offline-first (see [offline-first case](case-offline-first.md)): Room SSOT, outbox, delta sync, tombstones, optimistic writes. Media — store blobs on disk keyed by note, upload separately with resume (see [file upload](#4-design-a-file-upload-with-resume)), reference by local path until synced then swap to URL.
- **Tradeoffs:** Conflict resolution (LWW vs versioning vs CRDT for concurrent edits); inline media storage (disk quota) vs lazy download; text sync vs media sync decoupling.
- **Red flag:** No offline write path; conflict resolution that silently loses an edit; blocking note save on media upload; hard deletes.
- **Follow-ups**
    - Why decouple text sync from media sync instead of syncing a note as one atomic blob? *(text is small and cheap to sync instantly; a photo attachment can take minutes on a bad connection — coupling them means a slow photo upload blocks the note's text from ever syncing, when the two have completely different size/latency profiles.)*
    - A note is edited on two devices while offline, one adds a photo, the other edits text — how do you merge? *(field-level merge where possible — the photo attachment and the text edit touch different fields, so both can survive; true conflict only exists if both edited the *same* field, which is where you fall back to the LWW/versioning table in the offline-first case study.)*

## 10. Design a news/article reader with offline reading

- **Approach:** Clarify — save-for-offline, prefetch on Wi-Fi. Feed paged + cached; "save" downloads full article + images to disk; background prefetch of top articles constrained to unmetered network + charging; TTL-based cache invalidation; reading position persisted.

    ```kotlin
    val prefetchConstraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.UNMETERED)   // never burn the user's data plan
        .setRequiresCharging(true)
        .build()
    ```

- **Tradeoffs:** Prefetch scope (data/storage vs instant reads); TTL (freshness vs offline availability); text-only vs full-media offline save.
- **Red flag:** Prefetches on metered connection (burns data); no storage cap; no offline story despite "reader" implying commute use; ignores `NetworkType.UNMETERED` constraint.
- **Follow-ups**
    - The user explicitly taps "save for offline" while on cellular data — does the `UNMETERED` constraint block it? *(no — that constraint governs *background, opportunistic* prefetch. An explicit user action is a foreground request the user consented to pay data for; conflating the two is the classic mistake here.)*
    - How do you cap total offline storage without surprising the user? *(a visible quota with an explicit "manage offline articles" screen, LRU-evicting *unpinned* saved articles first when the cap is hit — never silently delete something the user explicitly saved without telling them.)*

## 11. Design analytics/event tracking SDK

- **Approach:** Clarify — must not block UI, must survive process death, batch to save battery. Enqueue events to a Room buffer synchronously-but-off-main; batch upload via WorkManager with backoff; dedupe with event IDs; cap buffer size with drop policy; flush on app background.

    ```kotlin
    @Entity data class AnalyticsEvent(
        @PrimaryKey val eventId: String,      // client UUID → dedupe key server-side
        val name: String,
        val propsJson: String,
        val timestamp: Long,
        val uploaded: Boolean = false,
    )
    // enqueue() is a fast Room insert on a background dispatcher — never on the caller's thread,
    // and never an HTTP call per event.
    ```

- **Tradeoffs:** Batch size/interval (freshness of data vs battery/network); at-least-once (dupes, needs idempotent IDs) vs at-most-once (data loss); buffer cap drop policy (oldest vs newest).
- **Red flag:** Sends an HTTP call per event (battery/network disaster); events only in memory (lost on crash — ironic for an analytics SDK); blocks the main thread; no retry/backoff.
- **Follow-ups**
    - Why at-least-once with dedup instead of exactly-once? *(exactly-once delivery over an unreliable network is effectively impossible without a distributed consensus protocol you don't want in a mobile SDK; at-least-once + a client-generated `eventId` the server dedupes on gets you the same practical guarantee — no lost events, no double-counted events — for a fraction of the complexity.)*
    - The buffer hits its cap during an offline session — drop oldest or newest events? *(usually oldest — a burst of stale low-value events is less useful than the events closest to "now," but for funnel-critical events (purchase, signup) exempt them from the drop policy entirely rather than applying one blanket rule.)*

## 12. Design a collaborative document editor (mobile client)

- **Approach:** Clarify — real-time multi-user editing, offline edits must merge. This is the one prompt that genuinely needs **CRDTs or OT** rather than LWW. Local model as CRDT, sync ops over WebSocket, buffer ops offline in an outbox and replay on reconnect, presence via ephemeral channel, DB persists the CRDT state.

    ```kotlin
    // Ops, not snapshots, are what sync — this is the core mental shift from the other prompts.
    sealed interface DocOp { data class Insert(val pos: Int, val char: String, val opId: String) : DocOp
                             data class Delete(val opId: String) : DocOp }
    // Offline ops queue exactly like the outbox pattern elsewhere, but REPLAY must be
    // order-independent (that's what makes it a CRDT) — unlike the notes outbox, which
    // must stay FIFO per entity.
    ```

- **Tradeoffs:** CRDT (convergent, no central authority, heavier state) vs OT (needs server transform, lighter payload); op granularity (character vs block); presence cost.
- **Red flag:** Proposes last-write-wins for concurrent text editing (data loss / clobbered paragraphs); no offline op buffering; assumes always-connected; ignores reconnect replay ordering.
- **Follow-ups**
    - Why can't you reuse the same outbox pattern from the notes/offline-first case here? *(that outbox assumes FIFO-ordered whole-entity upserts converge to one correct state; a CRDT's whole point is that ops from *multiple independent authors* must converge regardless of arrival order — a fundamentally different consistency model, not just a bigger outbox.)*
    - Two users delete the same character concurrently — what happens? *(CRDTs are designed so this is a no-op, not a conflict: deleting an already-tombstoned op is idempotent by construction, which is exactly the property that makes convergence guaranteed without a central arbiter.)*

## 13. Design a file downloader library

- **Approach:** Clarify — large file downloads, pause/resume, concurrency, priority queue, background downloading. Model — download tasks in Room (URL, local path, status: ENQUEUED/DOWNLOADING/PAUSED/COMPLETED/FAILED, progress bytes, checksum). Concurrency — ThreadPoolExecutor with bounds (max 3 concurrent downloads). Resume — use HTTP `Range: bytes=X-` header; download file to a temporary file (`.tmp`) and append bytes, rename to final path only on completion and checksum verify. Queue — priority queue sorted by user priority + queue time. WorkManager for background execution.

    ```kotlin
    // Resume via Range — same "ask the source of truth" discipline as the upload prompt,
    // just in the read direction: trust the file's actual size on disk, not a stored counter.
    val existingBytes = tempFile.length()
    val request = Request.Builder().url(url)
        .header("Range", "bytes=$existingBytes-")
        .build()
    // server responds 206 Partial Content — append; a 200 means it ignored Range → restart.
    ```

- **Tradeoffs:** Multi-connection chunked downloading (faster speed but higher socket/server overhead and tricky segment merging) vs single-connection resume (slower but robust and simple); database syncing frequency for progress (real-time slows UI; throttle progress updates to once per 500 ms).
- **Red flag:** Memory-buffers the entire file (causes OOM); starts from byte 0 on every connection drop; no connection limits (spawns 50 threads and crashes the app); blocks the UI thread.
- **Follow-ups**
    - The server responds `200 OK` instead of `206 Partial Content` to a ranged request — what do you do? *(some servers/CDNs silently ignore `Range` and return the full file from byte 0 — you must detect this (check the response `Content-Range` header, not just the status) and restart the write from offset 0, or you'll corrupt the file by appending a full copy onto the partial one.)*
    - Why rename `.tmp` → final path only after checksum verification? *(atomic rename means a reader can never observe a partially-written file under the final name — any consumer polling for "does this file exist" sees either nothing or a complete, verified file, never a corrupt in-progress one.)*

## 14. Design Instagram Stories (mobile client)

- **Approach:** Clarify — 24h lifetime, multi-segmented slides (images/videos), low latency transition, viewing state sync. Cache — segmented LRU disk cache; prefetch metadata (JSON lists of story URLs) and proactively download the next 1-2 slides ahead of the current active story. Player — ExoPlayer instances pooled (2-3 instances) for instant video play; images loaded via a memory-cached pipeline (see [image pipeline case](case-image-pipeline.md)). Sync — send view events (story ID, timestamp) to server via an optimistic batch queue (same shape as the [analytics SDK](#11-design-analyticsevent-tracking-sdk)).
- **Tradeoffs:** Video prefetching depth (too deep wastes network/battery if user exits early; too shallow causes buffering spinner); pre-rendering next slides vs memory consumption.
- **Red flag:** Loads and initializes a fresh Player per slide (causes jank/1-second lag during slide transitions); does not prefetch media; downloads stories on metered data without constraints.
- **Follow-ups**
    - Why pool 2-3 ExoPlayer instances instead of one, and why not one per slide? *(one instance means the *next* slide can't pre-buffer while the current one plays, causing a visible stall at every transition; one per slide is wasteful — each `ExoPlayer` holds real decoder/surface resources — pooling 2-3 gives "current + next pre-buffering" without unbounded resource use.)*
    - User rapid-taps through 10 stories in 2 seconds — what happens to the 8 prefetches you kicked off for slides they skipped past? *(cancel prefetch for any slide the user has scrolled past before it started rendering — the same lifecycle-scoped cancellation discipline as the image pipeline; otherwise you're burning battery/data downloading content nobody will see.)*

## 15. Design a location-based "Nearby Friends" app

- **Approach:** Clarify — real-time updates, geofencing, battery efficiency, scalability. Client — periodic location updates using `FusedLocationProviderClient` with a dynamic interval (slower when stationary via accelerometer/activity recognition, faster when moving). Spatial indexing — geohashing (e.g. 6-character Geohash, ~1.2km precision) to represent user location. Networking — upload Geohash to server, query server for matching geohashes of active friends. Map rendering — cluster points locally to avoid layout overload.

    ```kotlin
    // Geohash precision is the privacy/utility knob: shorter = coarser = more private,
    // fewer server queries; longer = precise = better UX, worse privacy if leaked.
    val myGeohash = GeoHash.encode(lat, lng, precision = 6)   // ~1.2 km cell
    api.updatePresence(myGeohash)   // server matches against friends' cells, not raw lat/lng
    ```

- **Tradeoffs:** Location accuracy vs battery (GPS vs network triangulation); polling intervals vs WebSockets for real-time friend movements; client-side vs server-side distance calculation.
- **Red flag:** Continuously queries GPS at maximum frequency (battery dies in 2 hours); uploads precise GPS coordinates to the server every second (privacy leak and network overhead); uses O(N) distance checks for thousands of users on the client.
- **Follow-ups**
    - Why upload a geohash instead of raw lat/lng? *(precision is capped by design — a 6-character geohash cell is ~1.2 km, so even if the server or the wire is compromised, an attacker learns "which neighborhood," not "which house"; it's privacy-by-construction, not a policy you have to remember to enforce.)*
    - A friend is stationary for an hour — do you keep polling GPS at the same rate? *(no — this is exactly what activity-recognition-driven interval adjustment is for: detect "still" and drop to a much slower update rate, since polling a stationary point at high frequency burns battery for zero new information.)*

## 16. Design a client-side logging library

- **Approach:** Clarify — high throughput, minimal overhead, persistence, remote upload, secure. API — `Log.d`, `Log.e` routing. Persistence — enqueues logs to an in-memory ring buffer, flushed asynchronously in batches to a file using `BufferedWriter` on a single background worker thread to prevent disk write contention. Rotation — split files when they exceed size limit (e.g. 5 MB) or age (e.g. 24h), keeping a max of 5 files. Security — encrypt log contents on the fly using AES-GCM (key stored in Android Keystore, see [M24 Security](../deep-dive/24-security.md#android-keystore-system)). Upload — compress logs (GZIP) and upload via WorkManager during Wi-Fi + charging.

    ```kotlin
    // Single writer thread is the whole design: every caller's log() just offers to a
    // channel; one coroutine drains it sequentially, so there's no lock contention on disk I/O.
    private val logChannel = Channel<LogLine>(capacity = Channel.UNLIMITED)
    fun log(line: LogLine) { logChannel.trySend(line) }   // never blocks the caller
    // background: for (line in logChannel) bufferedWriter.appendLine(line.format())
    ```

- **Tradeoffs:** SQLite DB (structured queryable logs but high write/CPU overhead) vs flat files (fast append, low overhead but harder to query); instant file syncing (no data loss but high disk wear) vs buffered flush.
- **Red flag:** Synchronous file writing on the logging thread (causes UI frames to drop on the main thread); unbounded file growth (fills up device storage); writes sensitive customer data in plaintext.
- **Follow-ups**
    - The app crashes right after a `Log.e()` call — is that log line lost? *(if it's still sitting in the in-memory ring buffer/channel and hasn't been flushed to disk yet, yes — this is the real tradeoff of buffered writes: flush more aggressively for `ERROR`-severity lines specifically (immediate flush) while batching `DEBUG`/`INFO`, so the logs you need most for a crash are the ones least likely to be lost.)*
    - Why a single background writer thread instead of writing from whichever thread calls `log()`? *(concurrent writers to the same file need locking, which is exactly the contention you're trying to avoid; a single-writer/many-producers channel gives you lock-free enqueue from any caller and strictly ordered, contention-free disk writes.)*

## 17. Real-time updates: HTTP Polling vs. Long-Polling vs. WebSockets vs. Server-Sent Events (SSE)

- **Approach:** Clarify the protocol characteristics for mobile:
    *   **Short Polling**: Client periodically sends HTTP requests. *Characteristics*: High battery and network overhead due to constant TCP handshakes/HTTP headers. *Use Case*: Low-frequency updates where delay is acceptable.
    *   **Long Polling**: Client sends request, server holds it open until new data is available. *Characteristics*: Reduces latency, but connection drops require constant reconnection overhead.
    *   **WebSockets**: Bi-directional, full-duplex TCP persistent connection. *Characteristics*: Lowest latency, low overhead, but requires keeping a TCP socket open (drains battery) and custom reconnect/heartbeat logic.
    *   **SSE**: Mono-directional (server-to-client) persistent HTTP connection. *Characteristics*: Reuses standard HTTP/2, auto-reconnects, but is read-only.
- **Tradeoffs:** WebSockets (great for chat, bi-directional, high battery footprint) vs SSE (great for live tickers/stocks, mono-directional, standard HTTP compatibility); WebSocket background battery drain vs FCM push notifications (wake radio only on data).
- **Red flag:** Leaves a WebSocket connection open continuously when the app is in the background (Google Play console battery warning); does not implement heartbeats/pings to detect silent connection drops.
- **Follow-ups**
    - You need bi-directional updates but battery review flagged your background WebSocket — what's the fix? *(close the socket when backgrounded, rely on FCM data messages to wake the app and trigger a short-lived reconnect + sync when something actually changed — the exact hybrid pattern from the [chat client](#3-design-a-chat-messaging-app), which never holds a foreground-only resource open in the background.)*
    - Why do you need application-level heartbeats when TCP already has keepalives? *(TCP keepalive detects a dead *socket*, not a dead *application* — a mobile carrier's NAT/firewall can silently drop an idle connection without either side's OS noticing for minutes; an app-level ping/pong on a short interval detects that "silent death" fast enough to reconnect before the user notices stale data.)*

## 18. How Voice & Video Calling works (WebRTC on Mobile)

- **Approach:** Clarify WebRTC architecture for mobile:
    *   **Signaling**: App exchanges SDP (Session Description Protocol) offer/answer and ICE candidates (IPs/ports) via a signaling channel (e.g. WebSocket or FCM push).
    *   **NAT Traversal**: Uses **STUN** (queries public IP/port) or **TURN** (relays media stream if direct peer-to-peer connection fails due to symmetric NAT).
    *   **Media Pipeline**: `AudioRecord` + `Camera2` capture raw frames → encoded via hardware codecs (H.264/VP8, Opus) → packetized and transmitted via RTP/SRTP over UDP.
    *   **State**: Exposes state updates (CONNECTING→CONNECTED→DISCONNECTED→FAILED).
- **Tradeoffs:** Peer-to-peer (no server media cost, low latency, but leaks client IP) vs SFU/MCU media servers (server-routed, scales to group calls, shields IPs, but high server cost).
- **Red flag:** Uses TCP for video/audio transmission (causes massive latency and lag due to head-of-line blocking); does not implement a TURN fallback (calls fail on most mobile carrier networks).
- **Follow-ups**
    - Why is TCP specifically wrong for live media, when it's the safe default everywhere else in this doc? *(TCP guarantees in-order delivery via retransmission — great for a file, catastrophic for live audio: one dropped packet stalls *every* packet behind it (head-of-line blocking) waiting for a retransmit, and by the time it arrives the audio is already stale. RTP over UDP just drops the lost packet and keeps playing — a glitch beats a stall for real-time media.)*
    - Both peers are behind symmetric NAT and TURN is misconfigured — what does the user experience? *(the call fails to connect entirely, typically stuck in `CONNECTING` then `FAILED` — this is exactly why "no TURN fallback" is the red flag: STUN-only P2P works on maybe 80-90% of real-world mobile networks, and the remaining calls need a relay or they simply never connect.)*

## 19. How "Where Is My Train" tracks train location without Internet

- **Approach:** Clarify off-grid tracking mechanisms:
    *   **Cell Tower Triangulation**: The app queries the telephony API (`TelephonyManager.getAllCellInfo()`) for the current Cell Tower ID (MCC, MNC, LAC, CID). It queries an **offline SQL database** packaged inside the APK containing the GPS coordinates of all railway-line cell towers.
    *   **GPS Satellites**: The mobile GPS receiver decodes signals from GPS satellites directly (requires no cellular network or internet) to obtain coordinates.
    *   **Offline Schedule Reconciliation**: Matches coordinates and speed against an offline timetable database (GTFS-like) to extrapolate the current station and delays.
- **Tradeoffs:** Cell ID tracking (extremely low battery, works deep inside the train compartment, but lower accuracy) vs GPS (high accuracy, but high battery drain and fails inside metallic train roofs).
- **Red flag:** Assumes location lookup requires a network geocoding API; queries GPS continuously at max frequency (burns battery); fails to package the location/cell databases offline.
- **Follow-ups**
    - How does the app know when to switch from cell-ID tracking to GPS, or vice versa? *(a simple confidence/availability heuristic — try GPS first with a short timeout; if no fix (common inside a metal train car), fall back to cell-ID triangulation against the bundled offline DB; neither is "the" answer, the design is choosing the cheaper signal that's actually available.)*
    - The bundled offline cell-tower database is 6 months out of date — what breaks, and how would you keep it fresh without requiring internet on every launch? *(new/moved towers cause mismatches → wrong-station guesses; the fix is an opportunistic background update — download the latest DB delta only when the app *does* have connectivity, e.g. at a station with Wi-Fi — the same "sync when you can, work without it" pattern as every offline-first prompt above.)*

---

!!! quote "The pattern across all nineteen"
    Every strong answer does the same four things: **clarifies offline/realtime/scale first**, makes the **local DB the single source of truth**, uses **cursors and optimistic writes with a sync queue**, and **closes on failure modes** (network loss mid-write, process death, cancellation). Every weak answer forgets offline, forgets cancellation, or designs only the happy path. And every strong answer survives the follow-up: it doesn't stop at the outline, it can defend *why* each piece is shaped the way it is.
