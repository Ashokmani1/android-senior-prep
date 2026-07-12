# DI Across Modules

Hilt in a single module is trivial. Hilt across a 30-module graph is where seniors are separated from mid-levels: you must know *where* bindings live, *how* the interface/impl split maps onto `:domain`/`:data`, and the escape hatches for code Hilt can't reach. This page is the multi-module mental model.

## The core rule: `@HiltAndroidApp` lives only in `:app`

Hilt generates the dependency graph once, rooted at the `Application`. That means:

- **`@HiltAndroidApp` is applied to exactly one class, in `:app`.** Never in a library/feature module.
- Library and feature modules **contribute** `@Module`s and **consume** injected types. They don't own the root component.
- `:app` transitively depends on every module whose bindings it needs, so Hilt's annotation processor can see them all when it assembles `SingletonComponent`.

!!! warning "Common trap"
    A feature module cannot have its own independent Hilt graph. There's one graph, assembled in `:app`. Feature modules just add `@Module @InstallIn(...)` classes and `@AndroidEntryPoint` / `@HiltViewModel` consumers. If you catch yourself wanting a second `@HiltAndroidApp`, you've misunderstood the model.

## Components & scopes you must name

| Component | Lifetime | Typical residents |
|---|---|---|
| `SingletonComponent` | Application | Retrofit, OkHttp, Room DB, DataStore, repositories, `@Singleton` |
| `ActivityRetainedComponent` | Survives config change (ViewModel scope) | `@ActivityRetainedScoped` state shared across a ViewModel's life |
| `ViewModelComponent` | One ViewModel | `@ViewModelScoped` deps injected into a `@HiltViewModel` |
| `ActivityComponent` | One Activity | `@ActivityScoped`, activity-bound helpers |
| `FragmentComponent` / `ViewComponent` | Fragment / View | rarely needed in Compose-first apps |
| `ServiceComponent` | Service | service-scoped deps |

!!! tip "In a Compose + single-Activity app, you mostly touch two"
    `SingletonComponent` (everything app-wide) and `ViewModelComponent`/`@HiltViewModel` (per-screen state). `ActivityRetainedComponent` matters when multiple ViewModels or nav-scoped state must share one instance across config changes.

## Where bindings physically live

The layout of `@Module`s across modules mirrors the module taxonomy. Three canonical placements:

### 1. `:core:network` — provide concrete infrastructure with `@Provides`

Third-party types you don't own (Retrofit, OkHttp) can't be `@Inject`-constructed, so you `@Provides` them in the module that owns that concern.

```kotlin
// :core:network
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttp(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(
                Json.asConverterFactory("application/json".toMediaType())
            )
            .build()

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi =
        retrofit.create(UserApi::class.java)
}
```

### 2. Interface in `:domain`, `@Binds` implementation in `:data`

This is the single most important multi-module DI pattern. The **abstraction** lives where consumers depend on it; the **implementation** and its `@Binds` live where the infrastructure lives.

```kotlin
// :domain — pure Kotlin, no Android/Retrofit deps. Features depend on THIS.
interface UserRepository {
    suspend fun getUser(id: String): User
}
```

```kotlin
// :data — implementation depends on :core:network + :core:database, NOT exposed to features.
class DefaultUserRepository @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao,
) : UserRepository {
    override suspend fun getUser(id: String): User =
        dao.get(id)?.toDomain() ?: api.fetch(id).also { dao.insert(it.toEntity()) }.toDomain()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    @Binds
    abstract fun bindUserRepository(impl: DefaultUserRepository): UserRepository
}
```

!!! note "Why `@Binds` over `@Provides` here"
    `@Binds` is generated as a no-op factory that just returns the concrete type as the interface — no runtime method body, cheaper than `@Provides`. Use `@Binds` whenever you're mapping an `@Inject`-constructed impl to an interface. Use `@Provides` when you must *construct* something you don't own (like Retrofit above).

    The payoff: `:feature:profile` depends only on `:domain`'s `UserRepository` interface. It has **no idea** Retrofit or Room exist. Swap the impl (add caching, change transport) and no feature recompiles.

### 3. `@HiltViewModel` in the feature — the consumer

```kotlin
// :feature:profile
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val getUser: GetUserUseCase,   // from :domain
    savedStateHandle: SavedStateHandle,
) : ViewModel() {
    private val userId: String = savedStateHandle["userId"]!!

    val state: StateFlow<ProfileUiState> = /* ... */
}
```

```kotlin
@Composable
fun ProfileRoute(viewModel: ProfileViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    ProfileScreen(state)
}
```

The feature **only consumes** — it injects use cases/repositories bound elsewhere. It contributes no infrastructure `@Module`s. This keeps features thin and swappable.

## The `@EntryPoint` escape hatch

Hilt injects into a fixed set of Android classes (`Application`, `Activity`, `Fragment`, `View`, `Service`, `BroadcastReceiver`) and `@Inject` constructors. For **everything else** — a class you don't own, a `ContentProvider`, a manually-instantiated object, or code in a module that can't take a Hilt dependency — use `@EntryPoint` to reach into the graph manually.

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface AnalyticsEntryPoint {
    fun analytics(): AnalyticsClient
}

// Anywhere you have a Context but no @Inject:
val analytics = EntryPointAccessors
    .fromApplication(context, AnalyticsEntryPoint::class.java)
    .analytics()
```

!!! tip "When this actually shows up"
    WorkManager workers (pre-`hilt-work`), `ContentProvider`s (initialization, e.g. app-startup libraries), interop with a legacy module that can't apply Hilt, or exposing a Hilt-provided singleton to a pure-Kotlin module. Treat it as a deliberate seam, not a default — overusing `@EntryPoint` means you're fighting the graph.

## Feature modules only CONSUME bindings

Restating the discipline because interviewers press on it:

- Features **inject** repositories/use cases and **host** `@HiltViewModel`s.
- Features **do not** `@Provides` infrastructure — that would duplicate wiring and couple the feature to a transport/storage choice.
- The only `@Module` you'd reasonably put in a feature is a small `@ViewModelScoped` binding local to that feature's screens. Anything app-wide belongs in `:core:*` or `:data`.

## Hilt vs manual DI vs Koin — the senior comparison

You will be asked to defend the choice. Lead with the **compile-time vs runtime** axis; that's the real tradeoff.

| Dimension | **Hilt** (Dagger) | **Manual DI** (constructor + a container) | **Koin** |
|---|---|---|---|
| Graph resolution | **Compile-time** — missing binding = build error | Compile-time (it's just Kotlin) | **Runtime** — missing binding = crash on resolve |
| Annotation processing | KSP/kapt (build-time cost) | None | None (no KSP) |
| Build speed | Slower (codegen) | Fastest | Fast (no codegen) |
| Boilerplate | Low, but annotation-heavy | High at scale (you wire everything) | Low (DSL modules) |
| Android integration | First-class (`@HiltViewModel`, `hiltViewModel()`, WorkManager, nav) | You build it | Good, via `koin-androidx-*` |
| Multi-module story | Excellent — `@InstallIn` + `@Binds` across modules | You manage aggregation manually | DSL modules per lib, loaded in `Application` |
| Failure mode | Fails the **build** | Fails the build | Fails at **runtime** |
| Best fit | Large multi-module apps, teams wanting compile-safety | Tiny apps / libraries; DI-as-a-library authors | Small–mid apps, KMP, teams avoiding codegen |

!!! quote "The soundbite"
    "For a large multi-module app I default to Hilt: the graph is verified at compile time, so a broken injection fails CI, not a user's device, and its multi-module `@InstallIn` model is exactly what a feature-first graph needs. Koin trades that compile-time safety for faster builds and no KSP — reasonable for a small app or KMP, but I don't want DI errors surfacing as runtime crashes at scale. Manual DI is cleanest for a library with a couple of dependencies where pulling in Dagger is overkill."

!!! note "KMP caveat, if asked"
    Hilt/Dagger is JVM/Android-only. For Kotlin Multiplatform shared code, manual DI or Koin (which supports KMP) is the realistic choice — a point worth raising if the role involves shared iOS/Android code.
