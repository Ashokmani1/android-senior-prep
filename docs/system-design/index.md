# Mobile System Design

Mobile system design is **not** backend system design with a phone drawn on the whiteboard. If you spend the round sizing a load balancer, sharding a database, and computing QPS for a fleet of servers, you have failed the assignment. In a mobile/client system design round **you own everything from the socket to the pixel** — and the server is a black box you negotiate a contract with.

The senior signal is this: you treat the **device as a hostile, resource-starved, frequently-disconnected environment** and design a client that stays correct and fast anyway.

!!! quote "The one-sentence framing"
    Backend SD optimizes for *throughput and consistency across machines*. Mobile SD optimizes for *correctness and responsiveness across network partitions, on a single battery-powered device with a few hundred MB of usable RAM.*

## What makes the mobile round different

The constraints are the whole game. Every design decision traces back to one of these:

| Constraint | Why it dominates | What it forces |
|---|---|---|
| **Flaky / offline network** | Trains, elevators, rural data, airplane mode | Local source of truth, retry, sync queues, offline-first |
| **Battery** | Radio wakeups and wakelocks drain % fast | Batch requests, coalesce, respect Doze/WorkManager constraints |
| **Limited RAM** | ~100–300 MB before the OS kills you | Bounded caches, downsampling, paging, no loading full lists |
| **Limited storage** | Users uninstall apps that hog GB | Cache eviction, quotas, cleanup policies |
| **Cold start** | First frame budget is ~1–2s or users bounce | Lazy init, cached data on screen before network returns |
| **Process death** | OS kills backgrounded apps anytime | Persist state to disk, restore on recreate, no in-memory-only state |
| **Untrusted client** | The device is in the attacker's hands | No secrets in the binary, cert pinning, encrypted storage |

Contrast with backend, where the hard problems are horizontal scale, consistency models across nodes, and cross-service latency. On the client you have exactly one node — but it can vanish mid-write and come back three days later with a stale copy.

## What the interviewer is actually grading

They are not grading whether you can name Retrofit. They watch for:

- **Handling ambiguity** — do you scope before you draw? A weak candidate starts coding endpoints; a strong one asks "does this need to work offline?" first.
- **Clean data flow** — is there a single source of truth, or does the UI read from three places and get three answers?
- **Offline & sync reasoning** — the highest-signal topic. Read path, write path, conflict resolution, deletes.
- **Caching strategy** — tiered, bounded, invalidated. Not "I'll cache it" with no eviction story.
- **Explicit tradeoffs** — every choice has a cost. Naming the cost is the senior move.
- **Failure modes** — what happens when the network dies mid-write, the disk is full, the token expires, the process is killed? Juniors design the happy path.

!!! warning "The three things weak candidates forget"
    **Offline**, **cancellation**, and **failure modes**. If you only remember one lesson from this section: whatever you design, immediately ask "what happens when the network is gone, the user leaves the screen, and the write half-completed?"

## How to use this section

<div class="grid cards" markdown>

-   **[The 6-Step Framework](framework.md)**

    A repeatable structure for any mobile SD prompt — clarify, model, layer, sync, cross-cutting, failure modes. Includes the canonical single-source-of-truth architecture.

-   **[Case: Offline-First App](case-offline-first.md)**

    Full worked example. Room as SSOT, outbox pattern, delta sync, optimistic writes with rollback, conflict resolution, tombstone deletes.

-   **[Case: Chat & Feed](case-chat-feed.md)**

    Message states, local echo, idempotency, WebSocket vs FCM vs polling, keyset pagination, and a Paging 3 + RemoteMediator feed.

-   **[Case: Image Loading Pipeline](case-image-pipeline.md)**

    Two-tier cache, request coalescing, downsampling to avoid OOM, lifecycle-tied cancellation, threading — and how Coil/Glide solve it.

-   **[Practice Prompts](questions.md)**

    10+ prompts with rapid model outlines: how to approach, key tradeoffs, and the red flag that sinks weak candidates.

</div>
