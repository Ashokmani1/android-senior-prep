# How to Clear the Senior/Lead Loop

## 1. Know the shape of the loop

A typical Senior/Lead Android loop is 4–6 rounds:

| Round | Duration | What they *actually* grade | This kit |
|---|---|---|---|
| Core Android / Kotlin | 45–60m | Depth on lifecycle, Compose, coroutines, memory | *your existing PDF* |
| Coding / DSA | 45–60m | Problem solving, clean code, tests | LeetCode (easy–medium) + Kotlin idioms |
| **App Architecture** | 60m | Layering, UDF, testability, tradeoffs | [Architecture](../architecture/index.md) |
| **System Design (mobile)** | 60m | Ambiguity handling, data/sync/offline, scale | [System Design](../system-design/index.md) |
| **Behavioral / Leadership** | 45–60m | Ownership, influence, conflict, mentoring | [Leadership](../leadership/index.md) |
| Bar-raiser / Hiring manager | 45m | Consistency, seniority signals, red flags | all of the above |

**Modularization** rarely gets its own round but surfaces inside Architecture *and* System
Design ("how do you keep this buildable at 30 engineers?"). It's a force multiplier.

## 2. The answer structure that scores

At this level, interviewers listen for a repeatable shape. Train yourself to answer in it:

1. **Clarify** — restate the problem, ask 2–3 scoping questions, state assumptions out loud.
2. **Options** — name 2–3 viable approaches (not one).
3. **Tradeoff** — pick one and say *what you're giving up* and *why it's acceptable here*.
4. **Concrete** — sketch it: a diagram, a data class, a module graph, an interface.
5. **Failure modes** — "here's where this breaks, and how I'd detect it."

!!! example "Junior answer vs Senior answer"
    **Q: Should you use MVI?**

    *Junior:* "Yes, MVI is the modern pattern, it has a single state and intents."

    *Senior:* "Depends on screen complexity. For a form with lots of interdependent state and
    events, MVI's single immutable state + reduced-over-intents gives me replayable, testable
    logic and no impossible states. For a trivial screen it's boilerplate — I'd use plain
    MVVM with a state data class. The cost of MVI is verbosity and a learning curve for the
    team; I'd standardize it only if we have several complex screens."

## 3. Seniority signals to plant deliberately

- **Tradeoffs over absolutes.** "It depends, and here's the axis it depends on."
- **Cite failure.** Real leaks, ANRs, bad migrations you've handled. Specifics = credibility.
- **Testability as a first-class driver**, not an afterthought.
- **Team scale.** "This holds at 5 engineers; at 30 I'd add module boundaries + convention plugins."
- **Data & metrics.** "I'd measure with Macrobenchmark / Crashlytics before optimizing."
- **Own the decision.** Use "I decided / I drove / I aligned the team," not "we kind of."

## 4. Red flags that sink Senior candidates

- One true architecture for everything (dogma).
- Can't explain *why*, only *what* (memorized, not understood).
- No mention of testing, observability, or rollback.
- Over-engineering a trivial screen; under-engineering a hard one.
- "We did it because the lead said so" — no ownership.
- Talking only tech in the behavioral round; no people/impact.

## 5. Company-tailoring (do this per application)

Read the JD and the company's public eng blog, then bias your prep:

- **Product startups** → offline-first, velocity, pragmatic architecture, wearing many hats.
- **Big tech / platform** → system design rigor, scale, cross-team, coding bar.
- **Agencies / consultancies** → breadth, modularization for reuse, delivery under constraints.

> Don't read this kit linearly. Analyze the JD, find the two weakest rounds for *that* role,
> and go deep there first.
