# 4-Week Study Plan

Assumes ~1–1.5 hrs/weekday + a longer weekend block. Pairs this kit with your existing
fundamentals PDF. Compress to 2 weeks by doubling daily load if your loop is soon.

## Week 1 — Architecture + refresh fundamentals

| Day | Focus | Deliverable |
|---|---|---|
| Mon | [Clean Architecture](../architecture/clean-architecture.md) | Draw the layer diagram from memory |
| Tue | [MVVM · MVI · UDF](../architecture/patterns.md) | Convert one of your screens to MVI on paper |
| Wed | [State & error modeling](../architecture/state-and-errors.md) | Write a `UiState` sealed hierarchy |
| Thu | PDF: Compose runtime, state, recomposition | 5 practice Qs aloud |
| Fri | [Architecture Q&A](../architecture/questions.md) | Answer 5 aloud, record yourself |
| Wknd | Mock: architecture round with a friend/LLM | Feedback notes |

## Week 2 — Modularization + build

| Day | Focus | Deliverable |
|---|---|---|
| Mon | [Modularization strategy](../modularization/strategy.md) | Sketch a module graph for a real app |
| Tue | [Convention plugins](../modularization/convention-plugins.md) | Write a `build-logic` plugin skeleton |
| Wed | [DI across modules](../modularization/di-across-modules.md) | Diagram Hilt components across modules |
| Thu | PDF: Gradle, build variants, R8, KSP | Notes |
| Fri | [Modularization Q&A](../modularization/questions.md) | Answer 5 aloud |
| Wknd | Refactor a toy project into 3 modules | Working `:app` + `:core` + `:feature` |

## Week 3 — System design (the biggest differentiator)

| Day | Focus | Deliverable |
|---|---|---|
| Mon | [SD framework](../system-design/framework.md) | Memorize the 6-step flow |
| Tue | [Offline-first & sync](../system-design/case-offline-first.md) | Whiteboard it end to end |
| Wed | [Chat / feed](../system-design/case-chat-feed.md) | Whiteboard it end to end |
| Thu | [Image pipeline](../system-design/case-image-pipeline.md) | Whiteboard it end to end |
| Fri | [System Design Q&A](../system-design/questions.md) | 2 full mock prompts, timed 45m |
| Wknd | 2 more prompts (news reader, ride-share tracker) | Diagrams + tradeoff notes |

## Week 4 — Leadership + integration

| Day | Focus | Deliverable |
|---|---|---|
| Mon | [Decisions & ADRs](../leadership/decisions-adr.md) | Write one real ADR from your past |
| Tue | [Behavioral STAR](../leadership/behavioral-star.md) | Draft 6 STAR stories |
| Wed | [Leadership Q&A](../leadership/questions.md) | Rehearse stories aloud |
| Thu | Full mock loop, part 1 (arch + SD) | Feedback |
| Fri | Full mock loop, part 2 (behavioral + core) | Feedback |
| Wknd | Company research per JD; patch weak spots | Tailored notes |

## Daily habits

- [ ] One answer **spoken aloud** every day (interviews are verbal, not written).
- [ ] Keep a running list of *"stories"* — real things you built, broke, fixed, decided.
- [ ] After each mock, write the **one thing** you'd say better next time.

!!! warning "Don't skip mocks"
    Reading is necessary but not sufficient. Reserve at least 3 live mock rounds — verbalizing
    tradeoffs under time pressure is a separate skill from knowing them.
