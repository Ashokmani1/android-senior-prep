# Core Android Deep Dive

A 12-phase, 41-module internals curriculum — the *"how does it actually work under the hood"*
questions that separate a Senior/Lead candidate from someone who only uses the APIs. Fully
[cross-referenced to the *Manifest Android Interview* book](../prep-strategy/manifest-coverage.md).

!!! tip "How this complements the Senior Rounds"
    The [Senior Rounds](../architecture/index.md) teach **judgment** (architecture, modularization,
    system design, leadership). This section teaches **depth** — `ActivityThread`, `ViewRootImpl`,
    Continuation-Passing-Style, the Retrofit dynamic proxy, Myers' diff, Binder IPC. Interviewers
    probe both: *"how would you design it"* **and** *"what happens internally when you call it."*

## Curriculum map

| Phase | Modules | Focus |
|---|---|---|
| **1 · Core Components** | M1 Activity · M2 Fragment | Lifecycle + framework internals |
| **2 · View & Rendering** | M3 View System · M4 RecyclerView | Measure/Layout/Draw, recycling |
| **3 · Architecture Components** | M5 ViewModel/LiveData · M6 Binding · M7 Room · M8 DataStore · M9 Paging | Jetpack data/state |
| **4 · Networking** | M10 Retrofit/OkHttp · M11 Advanced | Proxy, interceptors, caching |
| **5 · Coroutines & Flow** | M12 Coroutines · M13 Flow | CPS, dispatchers, StateFlow/SharedFlow |
| **6 · Dependency Injection** | M14 Dagger · M15 Hilt | Compile-time graph, components |
| **7 · Background** | M16 Services · M17 WorkManager · M18 Broadcast · M19 ContentProvider · M20 Alarm/Job | Execution limits, scheduling |
| **8 · Advanced Android** | M21 Navigation · M22 Permissions · M23 Storage · M24 Security · M25 Compose | Modern platform |
| **9 · System & Internals** | M26 Android Architecture · M27 Memory · M28 Rendering · M29 Context · M30 Manifest | ART, Zygote, Binder, GC |
| **10 · Library Internals** | M31 Coil · M32 JSON | Image pipeline, serialization |
| **11 · Performance & Testing** | M33 Performance · M34 Testing | Profiling, test pyramid |
| **12 · Build & Deploy** | M35 Gradle · M36 Distribution | Variants, signing, bundles |
| **Extras** | M37 Material · M38 Firebase · M39 Advanced · M40 View/Framework Extras · M41 Compose Internals | Breadth + book gap-fillers |

## How to study a module

Each module page follows the same rhythm so it stays interview-useful:

1. **What & why** — the concept and where it fits.
2. **Internals** — what the framework does under the hood (class names, flow, diagrams).
3. **Kotlin/code** — idiomatic usage + the gotchas.
4. **Interview angle** — the questions this module answers and the follow-ups.

!!! warning "Depth over breadth per session"
    Don't skim all 39. Interviews reward *depth*: pick the 8–10 modules on your target JD's stack
    and be able to whiteboard their internals. The [4-week plan](../prep-strategy/study-plan.md)
    sequences them.
