# State & Errors

State modeling is where architecture meets the compiler. The goal is to make **illegal states unrepresentable** — so a whole class of "how did the spinner and the error show at once?" bugs cannot compile — and to model failure as a value your logic must handle, not an exception that escapes to a crash.

## Modeling UiState: sealed interface vs data class

Two shapes, one decision.

### Data class with flags

```kotlin
data class FeedUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null,
)
```

### Sealed interface

```kotlin
sealed interface FeedUiState {
    data object Loading : FeedUiState
    data class Success(val items: List<Item>) : FeedUiState
    data class Error(val message: String, val retryable: Boolean) : FeedUiState
    data object Empty : FeedUiState
}
```

| | Data class + flags | Sealed interface |
|---|---|---|
| Impossible states | Allowed (`isLoading && error != null`) | Impossible by construction |
| Exhaustiveness | None — easy to forget a case | Compiler-enforced `when` |
| Partial states (refreshing *with* stale data visible) | Natural | Awkward — needs a nested field |
| Verbosity | Low | Higher |

!!! tip "How a senior decides"
    Use a **sealed interface** when the states are genuinely mutually exclusive — a screen is loading *or* showing content *or* errored. Use a **data class** when states legitimately coexist: pull-to-refresh where you show the old list *and* a refresh spinner *and* possibly a transient error banner, all at once. The mistake is dogma in either direction. A common, strong hybrid: a sealed `content` type for the mutually-exclusive part, wrapped in a data class that carries the orthogonal flags.

```kotlin
// Hybrid: mutually-exclusive content + orthogonal, coexisting flags.
data class FeedUiState(
    val content: Content,
    val isRefreshing: Boolean = false,
    val transientMessage: String? = null,
) {
    sealed interface Content {
        data object Loading : Content
        data class Loaded(val items: List<Item>) : Content
        data object Empty : Content
        data class Error(val message: String) : Content
    }
}
```

## Avoiding boolean soup

!!! warning "Boolean soup"
    `isLoading`, `isError`, `isEmpty`, `isRefreshing`, `hasMore`, `isRetrying` — six booleans create 64 combinations, of which maybe 5 are legal. Every reader now has to *know* which combinations are valid, the compiler enforces none of them, and the "loading spinner over an error message" bug is one careless `copy()` away. Collapse the mutually-exclusive booleans into a sealed type; keep only truly independent booleans as flags.

## Domain errors: Result / Either

Exceptions are for *exceptional*, unrecoverable conditions. **Expected** failures — validation rejected, item out of stock, auth expired — are part of your domain and should be values in the type system, so the compiler forces the caller to handle them.

```kotlin
// A domain-specific error type — no HTTP codes, no exceptions leaking upward.
sealed interface CheckoutError {
    data object OutOfStock : CheckoutError
    data object PaymentDeclined : CheckoutError
    data class Unknown(val cause: Throwable) : CheckoutError
}

// Kotlin's built-in Result works; a typed Either-style class is clearer for domain errors.
sealed interface Outcome<out T> {
    data class Ok<T>(val value: T) : Outcome<T>
    data class Err(val error: CheckoutError) : Outcome<Nothing>
}

interface CheckoutRepository {
    suspend fun submit(cart: Cart): Outcome<OrderId>
}
```

!!! note "kotlin.Result vs a sealed Either"
    `kotlin.Result<T>` carries a `Throwable`, so the failure type is untyped — the caller cannot exhaustively `when` over the possible errors. It is fine for "succeed or blow up" boundaries. For domain errors you want the caller to *branch on*, a sealed `Outcome`/`Either` with a typed error is stronger: the `when` is exhaustive and adding a new error case becomes a compile error at every call site. Also note `Result` as a function *return* type is fine; avoid it as a `suspend fun` return in some older tooling and never store it in a `StateFlow` without care.

## Mapping exceptions to user-facing errors

The rule: **exceptions die at the data layer.** The repository catches `IOException`, `HttpException`, `SerializationException` and maps them to typed domain errors. The ViewModel maps domain errors to localized strings. The domain layer never sees an `HttpException`, and the UI never sees an unmapped stack trace.

```kotlin
// data layer — catch framework exceptions, emit typed domain errors
override suspend fun submit(cart: Cart): Outcome<OrderId> = try {
    Outcome.Ok(api.checkout(cart.toDto()).toOrderId())
} catch (e: HttpException) {
    when (e.code()) {
        409 -> Outcome.Err(CheckoutError.OutOfStock)
        402 -> Outcome.Err(CheckoutError.PaymentDeclined)
        else -> Outcome.Err(CheckoutError.Unknown(e))
    }
} catch (e: IOException) {
    Outcome.Err(CheckoutError.Unknown(e))
}

// presentation layer — typed error -> localized, actionable message
fun CheckoutError.toUiMessage(res: ResourceProvider): String = when (this) {
    CheckoutError.OutOfStock -> res.string(R.string.err_out_of_stock)
    CheckoutError.PaymentDeclined -> res.string(R.string.err_payment_declined)
    is CheckoutError.Unknown -> res.string(R.string.err_generic)
}
```

## Retry and loading semantics

Loading and error are not single-shot; they interact with retry, and stale data changes the rules.

!!! tip "Retry that respects stale data"
    On a refresh failure, don't wipe the screen to a full-screen error if you already have content — show the stale list plus a dismissible error banner and a retry action. Full-screen `Error` is for the *initial* load with nothing to show. This is exactly why the hybrid state (content + `isRefreshing` + `transientMessage`) beats a pure sealed type for list screens.

- **Initial load fails, no data** → `Content.Error` with a retry button.
- **Refresh fails, data present** → keep `Content.Loaded`, set `transientMessage`, keep a retry affordance.
- **Retry in flight** → `isRefreshing = true`, content untouched.

## Combining multiple flows

Real screens derive state from several sources — the user, their cart, a feature flag. Combine them into one `StateFlow<UiState>` so the UI has a single thing to observe.

```kotlin
val uiState: StateFlow<ProfileUiState> = combine(
    userRepository.observeUser(userId),
    cartRepository.observeCart(),
    settingsRepository.observeTheme(),
) { user, cart, theme ->
    ProfileUiState.Success(
        user = user,
        cartCount = cart.itemCount,
        theme = theme,
    )
}
    .catch { emit(ProfileUiState.Error(it.toUserMessage())) }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = ProfileUiState.Loading,
    )
```

`combine` re-emits whenever *any* upstream emits, so the derived state is always consistent. `stateIn` converts the cold `Flow` into a hot `StateFlow` with a cached current value the UI can read synchronously.

## Why `WhileSubscribed(5000)`

!!! note "The 5000 question — a favorite"
    `SharingStarted.WhileSubscribed(stopTimeoutMillis = 5000)` keeps the upstream flow **active for 5 seconds after the last collector disappears**, then stops it. The 5 seconds is a deliberate window that bridges Android configuration changes — a screen rotation tears down and rebuilds the UI in well under 5 seconds, so the upstream (DB query, network observation) is *not* cancelled and restarted; the ViewModel survives, the collector reattaches, and the cached value is served instantly. If the user actually leaves the screen for good, after 5 seconds the flow stops and you stop wasting a DB cursor or socket on an invisible screen.

Contrast the three `SharingStarted` policies:

| Policy | Upstream lifecycle | Problem it causes / solves |
|---|---|---|
| `Eagerly` | Starts immediately, never stops | Wastes resources on screens nobody is watching; runs before anyone subscribes |
| `Lazily` | Starts on first collector, never stops | Never releases the upstream even after the screen is gone |
| `WhileSubscribed(5000)` | Active while subscribed + 5s grace | Survives config change **and** releases resources when truly gone — the right default |

!!! warning "Don't forget initialValue"
    `stateIn` requires an `initialValue`, and that value **is** your `Loading` state — it is what the UI renders in the gap before the first upstream emission. Forgetting it (or making it `Success(empty)`) produces a flash of empty content instead of a spinner.
