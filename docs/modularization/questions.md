# Interview Questions

Senior/lead modularization questions are rarely "what is a module." They're "defend your graph," "quantify the tradeoff," "what breaks at scale." Each question below gives the **strong, tradeoff-driven answer**, the **weak answer / red flag** in one line, and the **follow-ups** interviewers actually chain.

---

## 1. How would you modularize a 400k-line monolith?

!!! example "Strong answer"
    Modularize **incrementally and driven by pain**, not with a big-bang rewrite. Steps: (1) Extract the leaf, dependency-free layers first — `:core:model` (pure Kotlin), `:core:common`, `:core:designsystem` — since nothing depends *up* into the monolith. (2) Extract `:core:network`/`:core:database` behind interfaces. (3) Carve **one vertical feature** out end-to-end (`:feature:x` + its slice of `:data`/`:domain`) to prove the pattern and set up convention plugins. (4) Repeat feature by feature, using the module graph and build times to prioritize the biggest hotspots. Enforce boundaries with a CI graph-assertion so the monolith can't leak back in. Measure incremental build time before/after to justify the work to leadership.

!!! warning "Weak answer / red flag"
    "Split it into `:ui`, `:domain`, `:data` and do it all in one branch over a sprint." — big-bang + layer-first = merge hell and no ownership win.

**Follow-ups**

- How do you keep the app shippable during the migration? *(strangler-fig: modules coexist with the monolith; move code, don't stop releases.)*
- How do you pick which feature to extract first? *(highest churn × worst build-time contribution; prove ROI early.)*

---

## 2. By-layer vs by-feature — which and why?

!!! example "Strong answer"
    **Feature-first at the top level, layer within the feature** (the hybrid). By-layer top-level (`:ui`/`:domain`/`:data`) creates three mega-modules every team edits simultaneously — merge conflicts, zero ownership, and no build parallelism because any `:data` change rebuilds the world. By-feature localizes change and gives each team a module, and shared `:core:*`/`:data`/`:domain` beneath prevents duplication. The graph ends up wide and shallow, which is what parallelizes and localizes rebuilds.

!!! warning "Weak answer / red flag"
    "By-layer, because clean architecture says three layers." — conflates architectural layering (a code concern) with module boundaries (a build/ownership concern).

**Follow-ups**

- Where do cross-cutting concerns like analytics or a design system live? *(shared `:core:*`.)*
- Doesn't feature-first duplicate networking code? *(no — that's what shared `:core:network`/`:data` are for.)*

---

## 3. How do feature modules communicate without depending on each other?

!!! example "Strong answer"
    They don't depend on each other — that would couple teams, break parallel builds, and risk a Gradle-rejected cycle. Instead, features depend on a **navigation contract**: each feature exposes only a route/typed destination and registers its own nav graph; `:app` assembles the nav host. To navigate to another feature, a feature calls `navController.navigate(route)` against a route it knows by contract, never referencing the target's screens/ViewModels. When a real compile-time contract is needed, put the **interface in a shared `:core`/`:domain` module**, the implementation in the owning feature, and let Hilt inject it — so the caller depends on the abstraction, not the feature.

!!! warning "Weak answer / red flag"
    "Feature A just imports Feature B's screen and calls it." — direct feature-to-feature dependency; the exact thing to forbid.

**Follow-ups**

- `:feature:api` + `:feature:impl` split — when is it worth the extra modules? *(when features need richer compile-time contracts than a route string.)*
- How does `:app` know about every feature's graph without features knowing each other? *(`:app` depends on all features and aggregates their registered graphs.)*

---

## 4. `api` vs `implementation` — what's the difference and the build impact?

!!! example "Strong answer"
    `implementation` keeps a dependency off consumers' *compile* classpath; `api` exposes it transitively. Default to `implementation` because it enables **compilation avoidance** — changing an `implementation` dep's internals (or bumping it without an ABI change) doesn't recompile consumers, and it shrinks the classpath the compiler resolves. Use `api` only when your module's **public signatures** return/accept types from that dependency (e.g. `:domain` exposing `:core:model` types). Overusing `api` recreates the monolith: one ABI change ripples through the whole graph and kills incremental-build gains.

!!! warning "Weak answer / red flag"
    "Use `api` so everything's available everywhere — less hassle." — leaks transitive deps and destroys compilation avoidance.

**Follow-ups**

- Give a case where `api` is correct. *(a module whose public API returns a type from that dependency.)*
- What's compilation avoidance and how does `implementation` enable it? *(ABI changes stop at the module boundary.)*

---

## 5. `buildSrc` vs convention plugins — which and why?

!!! example "Strong answer"
    Convention plugins in a **`build-logic` included build**. `buildSrc` is an implicit classpath dependency of the entire build, so any edit invalidates all build-logic compilation and busts the configuration cache — a slow feedback loop on a 30-module project. `build-logic` exposes discrete `Plugin<Project>` classes registered via `gradlePlugin { plugins { register(...) } }`; changing one plugin only reconfigures modules that apply it, and the config cache survives unrelated edits. Modules opt in with `plugins { id("myapp.android.feature") }`, which composes library + compose + hilt conventions — eliminating 40 lines of copy-pasted config per module and making `compileSdk` a one-file change.

!!! warning "Weak answer / red flag"
    "`buildSrc` is fine, it's the same thing." — misses the whole-build invalidation cost that motivates the switch.

**Follow-ups**

- How does a plugin read `libs.versions.toml`? *(`VersionCatalogsExtension` + `findLibrary("name").get()`.)*
- Why `compileOnly` for AGP/Kotlin in the convention module? *(match the app's build-time plugin versions, don't bundle them.)*

---

## 6. How does Hilt work across a multi-module app?

!!! example "Strong answer"
    There's **one graph**, rooted at the single `@HiltAndroidApp` in `:app`, which transitively depends on every module contributing bindings. Modules contribute `@Module @InstallIn(SingletonComponent::class)` classes: `:core:network` `@Provides` Retrofit/OkHttp (types it doesn't own); the repository **interface lives in `:domain`** and its impl + `@Binds` live in `:data`. Feature modules only **consume** — they inject use cases/repositories and host `@HiltViewModel`s, contributing no app-wide infrastructure. For classes Hilt can't inject (workers, content providers, non-Hilt modules) use the `@EntryPoint` accessor. The win: features depend on abstractions, so swapping an impl recompiles nothing downstream.

!!! warning "Weak answer / red flag"
    "Each feature module has its own `@HiltAndroidApp` / its own component." — there's exactly one, in `:app`.

**Follow-ups**

- `@Binds` vs `@Provides`? *(`@Binds` maps an `@Inject` impl to an interface with no runtime body; `@Provides` constructs types you don't own.)*
- Where does a `@HiltViewModel` get scoped? *(`ViewModelComponent`, retrieved with `hiltViewModel()`.)*

---

## 7. How do you keep build times down as modules grow?

!!! example "Strong answer"
    Several levers, ordered by impact: (1) **Graph shape** — keep it wide and shallow so Gradle parallelizes and a change rebuilds only its module + reverse deps; avoid deep chains. (2) **`implementation` over `api`** for compilation avoidance. (3) **Build cache** (local + remote) and the **configuration cache** enabled. (4) **`build-logic` not `buildSrc`** to avoid whole-build invalidation. (5) Keep `:core:model` a pure-JVM module so it compiles fast and doesn't drag Android in. (6) KSP over kapt for annotation processors. (7) Profile with `--scan` / the build analyzer and attack the actual bottleneck module, not guesses.

!!! warning "Weak answer / red flag"
    "Just add more modules — more modules is always faster." — ignores that a deep/wrong-shaped graph and cold builds can regress; module *count* isn't the metric.

**Follow-ups**

- Cold build got *slower* after modularizing — why? *(clean builds of many modules have per-module overhead; the win is incremental, not clean.)*
- kapt vs KSP? *(KSP is faster, no stub generation.)*

---

## 8. What's a dynamic feature module and what does it cost?

!!! example "Strong answer"
    A `dynamic-feature` module is delivered via Play Feature Delivery — on-demand, conditional, or instant — instead of being in the base APK, shrinking initial download. But the dependency direction **inverts**: the base `:app` doesn't depend on the dynamic feature; the dynamic feature depends on `:app`. That complicates DI (the feature installs at runtime, so Hilt bindings and navigation must be resolved reflectively/`@EntryPoint`-style), complicates testing, and adds `SplitInstallManager` handling plus failure/rollback UX. I'd only reach for it when there's a real payoff — a large, rarely-used feature (heavy AR module, one-time onboarding) or an instant-app entry — not as a default modularization tool.

!!! warning "Weak answer / red flag"
    "Make every feature a dynamic feature to save space." — inverts dependencies everywhere and adds runtime install complexity for no benefit on small features.

**Follow-ups**

- Why is DI harder in a dynamic feature? *(it's not present at base-graph assembly; resolve via reflection/`@EntryPoint`.)*
- On-demand vs install-time vs conditional delivery — differences?

---

## 9. How do you enforce module boundaries so they don't erode?

!!! example "Strong answer"
    Make violations fail **CI, not code review**. Options: a **module graph assertion** (NowInAndroid ships one) that fails the build when an illegal edge appears (e.g. `:core` depending on a `:feature`, or a feature-to-feature edge); **Konsist** or **custom Lint/ArchUnit-style rules** to assert package/dependency conventions; Kotlin **`internal` visibility** so only the intended surface crosses the boundary; and `api`/`implementation` discipline so transitive leakage is impossible by construction. Gradle already rejects cycles, but I design to avoid them rather than rely on the failure. Boundaries that depend on human vigilance always rot.

!!! warning "Weak answer / red flag"
    "We tell people in code review not to add bad dependencies." — un-enforced conventions decay; needs a machine gate.

**Follow-ups**

- How would you fail CI on a feature-to-feature dependency? *(graph assertion / custom check over the resolved configuration.)*
- How does `internal` help across modules? *(hides everything not part of the public API from other modules.)*

---

## 10. What are the downsides of over-modularizing?

!!! example "Strong answer"
    Modules aren't free. Costs: **boilerplate tax** (every module needs a build file, namespace, manifest, DI wiring — convention plugins amortize but don't erase it); **navigation/DI indirection** replacing what used to be a direct call; **refactor friction** (moving a class across a boundary is a multi-file operation); **cognitive load** (new engineers must learn the graph); and **cold-build regression** — a from-scratch build of 60 tiny modules can be slower than a monolith. The failure mode is a graph so granular that every trivial change touches five modules. Right-size it: modularize reactively when build time, merge conflicts, or team size demand it, and stop when marginal cost exceeds the parallelism/ownership benefit.

!!! warning "Weak answer / red flag"
    "There are no downsides, more modules is strictly better." — signals they've never paid the maintenance cost at scale.

**Follow-ups**

- How granular is too granular? *(when a typical change spans many modules and refactor friction dominates.)*
- When would you *merge* modules back together? *(when two always change together and the boundary buys nothing.)*

---

## 11. How would you share a design system across features and even other apps?

!!! example "Strong answer"
    A `:core:designsystem` module holding **pure design** — theme, typography, color, spacing, and atomic composables (buttons, icons, chips) with **no domain knowledge**. Components that need domain models (an item card) go in `:core:ui`, which depends on `:core:designsystem` + `:core:model`. Every feature depends on `:core:designsystem` via `implementation`. Because it's dependency-free of app logic, the same module can be consumed by a Wear/TV target or a second app. Enforce that no feature reimplements a button by keeping design tokens `internal` to the module's public API and reviewing new atoms centrally.

!!! warning "Weak answer / red flag"
    "Each feature styles its own components." — guarantees visual drift and duplicated code.

**Follow-ups**

- Why separate `:core:designsystem` from `:core:ui`? *(one is domain-free; the other knows about models.)*
- How do you version it if another app consumes it? *(publish as an artifact or share via included build.)*

---

## 12. Walk me through the dependency rule for modules.

!!! example "Strong answer"
    Dependencies point **downward/inward only**: `:app → :feature → :domain/:data → :core`. Concrete rules: `:core` never depends on a `:feature`; **no feature depends on another feature** (communicate via nav contracts / shared interfaces); `:domain` is pure and depends only on `:core:model` (+`:core:common`); `:data` depends on `:domain` (to implement its interfaces) and infrastructure `:core:*`; `:app` is a thin top that wires everything and hosts the Hilt root. No cycles — Gradle rejects them, but I design against them. This is the dependency-inversion principle at module granularity: high-level policy (`:domain`) doesn't depend on low-level detail (`:data`); both meet at an interface.

!!! warning "Weak answer / red flag"
    "Any module can depend on any other as long as it compiles." — no dependency rule means the graph degenerates into a monolith with extra steps.

**Follow-ups**

- Where does dependency inversion appear? *(`:domain` owns the repository interface; `:data` implements it.)*
- Why must `:app` be thin? *(it's the composition root; logic there can't be reused or tested in isolation.)*
