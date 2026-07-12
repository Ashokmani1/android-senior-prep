# Manifest Book — Coverage Map

Cross-reference of every question in ***Manifest Android Interview* (skydoves)** against this kit —
so you can confirm nothing in your fundamentals book is left unprepared.

- **Book 1 — Android:** Q0–Q69 (70 questions)
- **Book 2 — Jetpack Compose:** Q0–Q43 (44 questions)
- **Total: 114 core questions** → **all mapped to a module below.**

!!! success "Coverage: 114/114"
    Framework and Jetpack questions map to the [Deep Dive](../deep-dive/index.md) modules; the deeper
    View/legacy and Compose-runtime questions the book specializes in are covered by
    [M40 · View & Framework Extras](../deep-dive/40-framework-extras.md) and
    [M41 · Compose Runtime Internals](../deep-dive/41-compose-internals.md), added specifically to close those gaps.
    One item (Book 2 Q10, *Kotlin idioms in Compose*) is covered at basics level and goes deeper in the
    planned **Kotlin Fundamentals** module (see the README roadmap).

## Book 1 — Android (Q0–Q69)

| Q | Question | Module | Status |
|---|----------|--------|:--:|
| 0 | What is Android? | [M26 Android Architecture](../deep-dive/26-android-architecture.md) | ✅ |
| 1 | What is Intent? | [M1 Activity](../deep-dive/01-activity.md) | ✅ |
| 2 | Purpose of PendingIntent | [M1 Activity](../deep-dive/01-activity.md) | ✅ |
| 3 | Serializable vs Parcelable | [M26 Android Architecture](../deep-dive/26-android-architecture.md) | ✅ |
| 4 | What is Context & its types | [M29 Context](../deep-dive/29-context.md) | ✅ |
| 5 | Application class | [M29 Context](../deep-dive/29-context.md) · [M26](../deep-dive/26-android-architecture.md) | ✅ |
| 6 | Purpose of AndroidManifest | [M30 Manifest](../deep-dive/30-manifest.md) | ✅ |
| 7 | Activity lifecycle | [M1 Activity](../deep-dive/01-activity.md) | ✅ |
| 8 | Fragment lifecycle | [M2 Fragment](../deep-dive/02-fragment.md) | ✅ |
| 9 | What is Service? | [M16 Services](../deep-dive/16-services.md) | ✅ |
| 10 | BroadcastReceiver | [M18 BroadcastReceiver](../deep-dive/18-broadcast-receiver.md) | ✅ |
| 11 | ContentProvider | [M19 ContentProvider](../deep-dive/19-contentprovider.md) | ✅ |
| 12 | Handle configuration changes | [M1 Activity](../deep-dive/01-activity.md) | ✅ |
| 13 | Memory management & leaks | [M27 Memory](../deep-dive/27-memory.md) | ✅ |
| 14 | ANR causes & prevention | [M33 Performance](../deep-dive/33-performance.md) | ✅ |
| 15 | Deep links | [M21 Navigation](../deep-dive/21-navigation.md) · [M1](../deep-dive/01-activity.md) | ✅ |
| 16 | Tasks & back stack | [M1 Activity](../deep-dive/01-activity.md) | ✅ |
| 17 | Purpose of Bundle | [M1 Activity](../deep-dive/01-activity.md) · [M40](../deep-dive/40-framework-extras.md) | ✅ |
| 18 | Pass data between Activities/Fragments | [M1](../deep-dive/01-activity.md) · [M2](../deep-dive/02-fragment.md) | ✅ |
| 19 | Activity during config change | [M1](../deep-dive/01-activity.md) · [M5](../deep-dive/05-viewmodel-livedata.md) | ✅ |
| 20 | ActivityManager | [M26 Android Architecture](../deep-dive/26-android-architecture.md) | ✅ |
| 21 | Advantages of SparseArray | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 22 | Runtime permissions | [M22 Permissions](../deep-dive/22-permissions.md) | ✅ |
| 23 | Looper, Handler, HandlerThread | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 24 | Trace exceptions | [M40](../deep-dive/40-framework-extras.md) · [M38 Firebase](../deep-dive/38-firebase.md) | ✅ |
| 25 | Build variants & flavors | [M35 Gradle](../deep-dive/35-gradle.md) | ✅ |
| 26 | Accessibility | [M39 Advanced](../deep-dive/39-advanced.md) | ✅ |
| 27 | Android file system | [M23 Storage](../deep-dive/23-storage.md) | ✅ |
| 28 | ART, Dalvik, Dex | [M26 Android Architecture](../deep-dive/26-android-architecture.md) | ✅ |
| 29 | APK vs AAB | [M36 Distribution](../deep-dive/36-distribution.md) | ✅ |
| 30 | R8 optimization | [M24 Security](../deep-dive/24-security.md) · [M33](../deep-dive/33-performance.md) | ✅ |
| 31 | Reduce app size | [M33 Performance](../deep-dive/33-performance.md) | ✅ |
| 32 | Process & how Android manages it | [M26](../deep-dive/26-android-architecture.md) · [M27](../deep-dive/27-memory.md) | ✅ |
| 33 | View lifecycle | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 34 | View vs ViewGroup | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 35 | ViewStub & UI perf | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 36 | Custom views | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 37 | Canvas | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 38 | Invalidation in View system | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 39 | ConstraintLayout | [M3 View System](../deep-dive/03-view-system.md) | ✅ |
| 40 | SurfaceView vs TextureView | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 41 | RecyclerView internals | [M4 RecyclerView](../deep-dive/04-recyclerview.md) | ✅ |
| 42 | Dp vs Sp | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 43 | Nine-patch image | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 44 | Drawable | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 45 | Bitmap & large bitmaps | [M27 Memory](../deep-dive/27-memory.md) · [M31 Coil](../deep-dive/31-coil.md) | ✅ |
| 46 | Animations | [M39 Advanced](../deep-dive/39-advanced.md) | ✅ |
| 47 | What is the Window? | [M1 Activity](../deep-dive/01-activity.md) | ✅ |
| 48 | Render a web page (WebView) | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 49 | Core Jetpack Compose libraries | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 50 | Material 3 (Material You) | [M37 Material](../deep-dive/37-material.md) · [M25](../deep-dive/25-compose-basics.md) | ✅ |
| 51 | AndroidX Core (androidx.core) | [M40 Framework Extras](../deep-dive/40-framework-extras.md) | ✅ |
| 52 | Observe ViewModel state in Compose | [M25](../deep-dive/25-compose-basics.md) · [M5](../deep-dive/05-viewmodel-livedata.md) | ✅ |
| 53 | Adaptive UIs for screen sizes | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 54 | Jetpack ViewModel | [M5 ViewModel & LiveData](../deep-dive/05-viewmodel-livedata.md) | ✅ |
| 55 | Jetpack Navigation Library | [M21 Navigation](../deep-dive/21-navigation.md) | ✅ |
| 56 | Dagger 2 and Hilt | [M14 Dagger](../deep-dive/14-dagger.md) · [M15 Hilt](../deep-dive/15-hilt.md) | ✅ |
| 57 | Jetpack Paging library | [M9 Paging 3](../deep-dive/09-paging.md) | ✅ |
| 58 | Baseline Profile | [M33 Performance](../deep-dive/33-performance.md) | ✅ |
| 59 | Long-running background tasks | [M17 WorkManager](../deep-dive/17-workmanager.md) · [M16](../deep-dive/16-services.md) | ✅ |
| 60 | Serialize JSON to object | [M32 JSON Parsing](../deep-dive/32-json.md) | ✅ |
| 61 | Network requests & libraries | [M10 Retrofit & OkHttp](../deep-dive/10-retrofit.md) | ✅ |
| 62 | Paging for large datasets | [M9 Paging 3](../deep-dive/09-paging.md) | ✅ |
| 63 | Fetch & render network images | [M31 Coil](../deep-dive/31-coil.md) | ✅ |
| 64 | Store & persist data locally | [M7 Room](../deep-dive/07-room.md) · [M8 DataStore](../deep-dive/08-datastore.md) | ✅ |
| 65 | Offline-first features | [System Design: Offline-First](../system-design/case-offline-first.md) | ✅ |
| 66 | Where to launch initial data (LaunchedEffect) | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 67 | Predictive back gesture | [M39 Advanced](../deep-dive/39-advanced.md) | ✅ |
| 68 | KSP vs kapt | [M35 Gradle](../deep-dive/35-gradle.md) | ✅ |
| 69 | LiveEdit & Live Literals | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |

## Book 2 — Jetpack Compose (Q0–Q43)

| Q | Question | Module | Status |
|---|----------|--------|:--:|
| 0 | Structure of Jetpack Compose | [M25](../deep-dive/25-compose-basics.md) · [M41](../deep-dive/41-compose-internals.md) | ✅ |
| 1 | Compose phases | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 2 | Why Compose is declarative | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 3 | Recomposition & when it occurs | [M25](../deep-dive/25-compose-basics.md) · [M41](../deep-dive/41-compose-internals.md) | ✅ |
| 4 | How composable functions work internally | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 5 | Stability & performance | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 6 | Optimizing perf via stability | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 7 | Composition & how to create it | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 8 | Migrating XML → Compose | [M25](../deep-dive/25-compose-basics.md) · [ADRs](../leadership/decisions-adr.md) | ✅ |
| 9 | Testing Compose perf in release mode | [M41](../deep-dive/41-compose-internals.md) · [M34](../deep-dive/34-testing.md) | ✅ |
| 10 | Kotlin idioms used in Compose | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ◐ |
| 11 | State & APIs to manage it | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 12 | State hoisting | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 13 | remember vs rememberSaveable | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 14 | Coroutine scope in composables | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 15 | Side effects in composables | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 16 | rememberUpdatedState | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 17 | produceState | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 18 | snapshotFlow | [M25](../deep-dive/25-compose-basics.md) · [M41](../deep-dive/41-compose-internals.md) | ✅ |
| 19 | derivedStateOf | [M25](../deep-dive/25-compose-basics.md) · [M41](../deep-dive/41-compose-internals.md) | ✅ |
| 20 | Lifecycle of composables | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 21 | SaveableStateHolder | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 22 | The snapshot system | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 23 | mutableStateListOf / mutableStateMapOf | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 24 | Safely collect Flow in composables | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 25 | CompositionLocals | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 26 | Modifier | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 27 | Layout | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 28 | Box | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 29 | Arrangement vs Alignment | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 30 | Painter | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 31 | Load images from network | [M31 Coil](../deep-dive/31-coil.md) | ✅ |
| 32 | Efficiently render long lists | [M25 Compose Basics](../deep-dive/25-compose-basics.md) | ✅ |
| 33 | Pagination with lazy lists | [M9 Paging 3](../deep-dive/09-paging.md) | ✅ |
| 34 | Canvas (Compose) | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 35 | graphicsLayer Modifier | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 36 | Visual animations in Compose | [M41](../deep-dive/41-compose-internals.md) · [M39](../deep-dive/39-advanced.md) | ✅ |
| 37 | Navigate between screens | [M21 Navigation](../deep-dive/21-navigation.md) | ✅ |
| 38 | Preview & how to handle them | [M41 Compose Internals](../deep-dive/41-compose-internals.md) | ✅ |
| 39 | Unit tests for Compose UI | [M34 Testing](../deep-dive/34-testing.md) | ✅ |
| 40 | Screenshot testing | [M41](../deep-dive/41-compose-internals.md) · [M34](../deep-dive/34-testing.md) | ✅ |
| 41 | Accessibility in Compose | [M25](../deep-dive/25-compose-basics.md) · [M39](../deep-dive/39-advanced.md) | ✅ |
| 42 | Edge-to-edge & window insets | [M37 Material](../deep-dive/37-material.md) · [M39](../deep-dive/39-advanced.md) | ✅ |
| 43 | Benchmark key user journeys | [M33 Performance](../deep-dive/33-performance.md) | ✅ |

!!! note "Legend"
    ✅ Covered · ◐ Covered at basics level, deeper treatment planned (Kotlin Fundamentals module).
    The book's **172+ practical questions** are the "apply it yourself" prompts — rehearse them against
    the matching module above, then use each module's **Interview Q&A** section to self-test.
