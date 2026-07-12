# Modularization

Modularization is the single highest-leverage architectural decision on a large Android codebase. At senior/lead level you are not asked *whether* Gradle modules exist — you are asked to defend a module graph, quantify what it buys, and admit what it costs. This section gives you the opinionated version.

!!! quote "The one-liner to open the interview with"
    "Modules are a build-tooling and ownership boundary first, an architecture boundary second. I modularize to parallelize the build, enforce dependencies the compiler can check, and let teams own code without stepping on each other. Everything else is downstream of those three."

## Why modularize

There are five real payoffs. Rank them in this order when asked — build speed and enforced boundaries are the ones that survive scrutiny.

| Driver | What you actually get | The mechanism |
|---|---|---|
| **Build speed** | Faster CI and local iteration | Gradle runs independent modules **in parallel**, only **recompiles changed modules + their reverse deps** (incremental), and reuses outputs via the **build cache** (local + remote) |
| **Enforced boundaries** | Illegal dependencies become compile errors, not code-review comments | A module can only see what its `dependencies {}` block exposes. `implementation` hides transitives; the compiler enforces the layering |
| **Ownership** | Teams own modules end-to-end | `CODEOWNERS` maps module paths to teams; PRs auto-route; blast radius of a change is visible in the graph |
| **Reusability** | Share `:core:*` across app variants, wearables, and other apps | A `:core:designsystem` or `:core:network` module is consumed, not copy-pasted |
| **Dynamic delivery** | Ship less; install-on-demand | Play Feature Delivery serves `dynamic-feature` modules conditionally / on-demand, shrinking the base APK |

!!! tip "Build-speed nuance interviewers probe for"
    Parallelism only helps if the graph is **wide and shallow**, not a deep chain. If `:app → :feature:a → :feature:b → :core`, Gradle can't parallelize a linear chain and a change low in the chain rebuilds everything above it. Wide graphs (many leaf features depending on a stable core) parallelize well and localize rebuilds. **Graph shape, not module count, determines build speed.**

## When NOT to modularize

Modularization has real, ongoing cost. Say this out loud — it signals seniority more than reciting benefits.

!!! warning "The costs are permanent, not one-time"
    - **Boilerplate tax**: every module needs a `build.gradle.kts`, namespace, manifest, and DI wiring. Convention plugins amortize this but never eliminate it.
    - **Navigation & DI indirection**: cross-feature calls now go through abstractions (nav contracts, `@EntryPoint`) instead of a direct call.
    - **Refactor friction**: moving a class across a module boundary is a multi-file, multi-`build.gradle` operation, not a drag-and-drop.
    - **Cognitive overhead**: a new engineer must learn the graph before shipping one screen.
    - **Cold-build regression**: a *from-scratch* build of 60 modules can be **slower** than a monolith — you win on incremental, not clean, builds.

Do **not** modularize (beyond maybe `:app` + `:core`) when:

- The app is small (a few dozen screens) and will stay that way.
- The team is 1–3 engineers — there are no ownership boundaries to enforce and merge conflicts are rare.
- You are pre-product-market-fit and the architecture will be rewritten anyway.

## The signals it's time

Modularize reactively, driven by pain, not by blog posts:

- Incremental builds routinely exceed ~1–2 minutes and profiling blames a single giant `:app`.
- Merge conflicts in shared files (a god `AppModule`, a monolithic `nav_graph`, one `strings.xml`) are weekly.
- Team headcount crossed ~6–8 and you want independent ownership.
- You need on-demand delivery, an instant app, or to share code with a Wear/TV/other-app target.
- Code review can't stop layering violations by eyeball anymore.

## The golden rule

!!! note "Modularize by FEATURE, layer within the feature"
    Top-level modules are **features** (`:feature:search`, `:feature:profile`). *Inside or beneath* a feature you apply clean-architecture **layers** (ui → domain → data). Do **not** make your top-level split `:ui`, `:domain`, `:data` — that produces three giant modules that every team edits simultaneously, which defeats the ownership and parallelism goals. Feature-first localizes change; layer-first re-centralizes it.

## Explore this section

<div class="grid cards" markdown>

-   **[Strategy](strategy.md)**

    By-layer vs by-feature vs hybrid, the canonical module types, `api` vs `implementation`, the feature-to-feature problem, and a real module graph.

-   **[Convention Plugins](convention-plugins.md)**

    The `build-logic` included build, why it beats `buildSrc`, and a real `AndroidFeatureConventionPlugin.kt` that standardizes 30+ modules.

-   **[DI Across Modules](di-across-modules.md)**

    Hilt in a multi-module graph: components/scopes, where `@Module`s live, `@Binds` across `:domain`/`:data`, `@EntryPoint`, and Hilt vs Koin vs manual DI.

-   **[Interview Questions](questions.md)**

    10–12 senior questions with strong answers, red-flag weak answers, and the follow-ups interviewers actually ask.

</div>
