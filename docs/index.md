# Android Senior & Lead Interview Kit

> The rounds that fundamentals don't win.

You already have a strong fundamentals resource (*Manifest Android Interview* and friends).
That gets you through the **core Android knowledge** round. This kit is built for the **other
four rounds** that decide Senior and Lead offers.

## The Senior/Lead loop — and where this kit fits

<div class="grid cards" markdown>

-   :material-layers-triple: **[App Architecture](architecture/index.md)**

    Clean Architecture, MVVM vs MVI, unidirectional data flow, modeling state and errors.
    *"Design the presentation layer for this screen."*

-   :material-graph-outline: **[Modularization](modularization/index.md)**

    Multi-module strategy, `build-logic` convention plugins, the module graph, DI across
    modules, dynamic features. *"How would you split a 400k-line app?"*

-   :material-sitemap: **[System Design](system-design/index.md)**

    Offline-first + sync, caching, image pipelines, chat/feed design, tradeoffs at scale.
    *"Design a WhatsApp-style messaging client."*

-   :material-account-group: **[Leadership & Behavioral](leadership/index.md)**

    ADRs, technical tradeoffs, mentoring, code review, incident handling — in STAR form.
    *"Tell me about a hard technical decision you owned."*

</div>

## Why fundamentals alone stall at "Senior"

| Signal interviewers grade | Junior | Senior | Lead |
|---|---|---|---|
| Recall APIs / lifecycle correctly | ✅ core | assumed | assumed |
| Justify **architecture tradeoffs** | — | ✅ core | ✅ |
| Design a **multi-module** codebase | — | ✅ | ✅ core |
| **System design** under ambiguity | — | ✅ | ✅ core |
| Drive **decisions & people** | — | partial | ✅ core |

A great answer at this level isn't "what" — it's **"what, why, the alternative you rejected,
and the tradeoff you accepted."** Every page here is written to that bar.

## How to use this kit

1. Start with **[Prep Strategy → How to Clear the Loop](prep-strategy/how-to-clear.md)** to map the process.
2. Work the four modules. Read the concept pages, then drill the **Interview Q&A** page in each — those are written as *model answers with follow-ups*.
3. Convert the behavioral prompts into **your own STAR stories** before the loop.

!!! tip "This kit is extensible by design"
    Add a topic by dropping `docs/<topic>/index.md` and registering it in `mkdocs.yml`.
    The roadmap (build/Gradle, testing, coroutines, performance, KMP) is already stubbed in the README.
