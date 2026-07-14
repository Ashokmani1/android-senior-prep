# Interview Questions: Architecture

Senior architecture rounds reward **judgment expressed as tradeoffs**. For each question below: the strong answer is what a lead would say, the red flag is the one-liner that ends the interview early, and the follow-ups are what the interviewer already has loaded to test whether your first answer was memorized or understood.

---

## 1. Why layer an app at all? Isn't it just more files?

!!! quote "Strong answer"
    Layers buy **independent rates of change**. UI churns constantly, business rules occasionally, data sources rarely — separating them means a Retrofit-to-Ktor migration never touches a screen and a redesign never touches a business rule. The second payoff is **testability**: pure domain logic runs on the JVM with no emulator. I don't layer reflexively though — a two-screen utility app doesn't need three modules. Layers are a response to *change pressure and team size*, not a default.

    ```kotlin
    // Unlayered: swapping Retrofit for Ktor touches the ViewModel AND the UI's error handling.
    class ProfileViewModel(private val api: RetrofitApi) : ViewModel() {
        fun load() = viewModelScope.launch {
            try { _state.value = Success(api.getProfile()) }        // Retrofit exception types leak here
            catch (e: HttpException) { _state.value = Error(e.code()) }
        }
    }

    // Layered: the ViewModel depends on an interface. Retrofit-to-Ktor is a :data-module-only change.
    interface ProfileRepository { suspend fun getProfile(): Result<Profile> }

    class ProfileViewModel(private val repo: ProfileRepository) : ViewModel() {
        fun load() = viewModelScope.launch {
            _state.value = repo.getProfile().fold(::Success, ::Error)  // no networking type in sight
        }
    }
    ```

!!! warning "Red flag"
    "Because that's clean architecture / best practice." — dogma with no cost awareness.

**Follow-ups:** When would you *not* layer? · How do layers help a five-person team specifically?

---

## 2. Do you always need UseCases?

!!! quote "Strong answer"
    No. A UseCase earns its place when logic is non-trivial (combines repositories, applies rules) or is reused across ViewModels, or when the team wants one enforced convention. A UseCase whose body is a single `repository.getX()` is pure ceremony — a file, an injection, and a test buying nothing. My default: ViewModels may call repositories directly, and I introduce a UseCase the moment logic exceeds one call or needs sharing. I optimize for the next reader deleting indirection, not for diagram symmetry.

    ```kotlin
    // Ceremony — buys nothing over calling the repository directly.
    class GetProfileUseCase @Inject constructor(private val repo: ProfileRepository) {
        suspend operator fun invoke() = repo.getProfile()
    }

    // Earns its place — combines two repositories AND applies a rule; reused by 3 ViewModels.
    class CheckoutUseCase @Inject constructor(
        private val cart: CartRepository,
        private val pricing: PricingRepository,
    ) {
        suspend operator fun invoke(cartId: String): CheckoutResult {
            val items = cart.getItems(cartId)
            val prices = pricing.currentPrices(items.map { it.sku })
            return if (items.any { it.qty > prices[it.sku]?.stock ?: 0 })
                CheckoutResult.OutOfStock
            else CheckoutResult.Ready(items, prices)
        }
    }
    ```

!!! warning "Red flag"
    "Every ViewModel must go through a UseCase for every call." — ceremony without judgment.

**Follow-ups:** Your team insists on UseCases everywhere for consistency — how do you argue it? · Where does the logic go if you skip the UseCase?

---

## 3. MVVM or MVI — which do you use?

!!! quote "Strong answer"
    I always want UDF; the question is how much structure I need to enforce it. MVVM with a single `StateFlow<UiState>` is UDF at low ceremony — my default for most screens. I reach for MVI when a screen is complex and stateful enough that impossible states become real bugs, or when a large team benefits from one enforced convention: single immutable state, intents, a reducer. MVI's cost is boilerplate on every screen; I don't pay it for a settings page with two toggles.

    ```kotlin
    // MVVM default — low ceremony, one state stream, direct mutation calls.
    class SettingsViewModel : ViewModel() {
        private val _state = MutableStateFlow(SettingsUiState())
        val state: StateFlow<SettingsUiState> = _state.asStateFlow()
        fun onDarkModeToggled(on: Boolean) { _state.update { it.copy(darkMode = on) } }
    }

    // MVI — explicit intent -> reducer, worth it once state transitions get complex.
    sealed interface CheckoutIntent { data class ApplyPromo(val code: String) : CheckoutIntent
                                       object ConfirmPayment : CheckoutIntent }

    class CheckoutViewModel : ViewModel() {
        private val _state = MutableStateFlow(CheckoutUiState())
        val state: StateFlow<CheckoutUiState> = _state.asStateFlow()

        fun dispatch(intent: CheckoutIntent) = _state.update { current -> reduce(current, intent) }

        private fun reduce(state: CheckoutUiState, intent: CheckoutIntent): CheckoutUiState =
            when (intent) {                                    // one place decides every legal transition
                is CheckoutIntent.ApplyPromo -> state.copy(promoCode = intent.code, discounted = true)
                CheckoutIntent.ConfirmPayment ->
                    if (state.discounted) state.copy(confirmed = true) else state  // impossible states blocked here
            }
    }
    ```

!!! warning "Red flag"
    "MVI is more modern so it's better." — treats a tradeoff as a fashion.

**Follow-ups:** What concrete bug does MVI's single state prevent? · What's the cost of MVI on a trivial screen?

---

## 4. Where does business logic live?

!!! quote "Strong answer"
    In the domain layer — UseCases or domain services — never in the ViewModel or the Composable. The ViewModel *orchestrates* (calls domain, shapes results into `UiState`); it doesn't *decide* (apply the discount rule, validate the transfer). The tell for a leak is a business rule you can't unit-test without Android, or the same rule copy-pasted across two screens. If logic is genuinely UI-only — a formatting choice — it belongs in presentation, and that's fine.

    ```kotlin
    // Leaked — the discount RULE lives in the ViewModel; untestable without duplicating it per screen.
    class CartViewModel : ViewModel() {
        fun total(items: List<CartItem>) =
            items.sumOf { it.price } * if (items.size >= 3) 0.9 else 1.0   // the rule, hiding here
    }

    // Correct — the rule is a pure, independently-testable domain function; the VM only orchestrates.
    class ApplyBulkDiscount @Inject constructor() {
        operator fun invoke(items: List<CartItem>): Money =
            items.sumOf { it.price } * if (items.size >= 3) 0.9 else 1.0
    }
    class CartViewModel(private val applyDiscount: ApplyBulkDiscount) : ViewModel() {
        fun total(items: List<CartItem>) = applyDiscount(items)   // orchestrates, decides nothing
    }
    ```

!!! warning "Red flag"
    "In the ViewModel." (as the unqualified default) — the classic God-ViewModel trajectory.

**Follow-ups:** How do you keep a discount rule out of the ViewModel? · What logic legitimately *does* belong in presentation?

---

## 5. How do you make a ViewModel testable?

!!! quote "Strong answer"
    Inject its collaborators as interfaces (UseCases or repository interfaces) so I substitute fakes — I prefer hand-written fakes over mocks for state-based tests. Keep business logic out of it so tests assert *state transitions*, not rules. Control coroutines with an injected dispatcher or `StandardTestDispatcher` + `runTest`, and assert on the `StateFlow` with Turbine. Crucially, no `Context`, no `Android` framework types in the ViewModel — the moment those appear, I need Robolectric and the tests slow down and get flaky.

    ```kotlin
    class ProfileViewModel(
        private val repo: ProfileRepository,          // interface — substitutable
        private val io: CoroutineDispatcher,           // injected — controllable in tests
    ) : ViewModel() { /* ... */ }

    class FakeProfileRepository : ProfileRepository {
        var result: Result<Profile> = Result.success(Profile("Ada"))
        override suspend fun getProfile() = result     // hand-written fake, no mocking framework
    }

    @Test
    fun `load emits Success on happy path`() = runTest {
        val vm = ProfileViewModel(FakeProfileRepository(), StandardTestDispatcher(testScheduler))
        vm.state.test {                                 // Turbine
            vm.load()
            assertEquals(ProfileUiState.Loading, awaitItem())
            assertEquals(ProfileUiState.Success(Profile("Ada")), awaitItem())
        }
    }
    ```

!!! warning "Red flag"
    "I test it with Espresso / on a device." — doesn't distinguish unit from UI testing, or reaches for the heavy tool first.

**Follow-ups:** How do you test a flow that uses `WhileSubscribed`? · Mocks or fakes, and why?

---

## 6. One UiState stream or several?

!!! quote "Strong answer"
    One `StateFlow<UiState>` per screen as the default — the UI observes a single consistent snapshot, and I use `combine` to derive it from multiple sources so the parts can never be momentarily inconsistent. I split into multiple streams only when sections are genuinely independent and have very different update frequencies (a high-frequency scrubber vs. static metadata) and merging them would cause wasteful recomposition. Multiple streams is an optimization I justify, not a starting point.

    ```kotlin
    // Scattered — two streams can be read at different points in time, momentarily inconsistent.
    val user: StateFlow<User?> = repo.observeUser().stateIn(scope, WhileSubscribed(5_000), null)
    val cart: StateFlow<List<CartItem>> = repo.observeCart().stateIn(scope, WhileSubscribed(5_000), emptyList())

    // One consistent snapshot — combine() guarantees the UI never sees a stale pairing.
    val state: StateFlow<CheckoutUiState> =
        combine(repo.observeUser(), repo.observeCart()) { user, cart -> CheckoutUiState(user, cart) }
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), CheckoutUiState.Loading)
    ```

!!! warning "Red flag"
    "One StateFlow per field." — recreates scattered-observable MVVM and its inconsistency bugs.

**Follow-ups:** How does `combine` keep the state consistent? · When would multiple streams actually help performance?

---

## 7. How do you handle one-off events like navigation or a toast?

!!! quote "Strong answer"
    State answers "what do I render" and must be safe to re-read; one-off events must not be re-read or they double-fire. Putting a `navigate` boolean in `UiState` re-navigates on the next rotation — the classic bug. I model events separately: a `Channel` exposed as `receiveAsFlow()` for single-consumer delivery that survives the config-change collector gap, collected in Compose via `flowWithLifecycle`. I use `SharedFlow` only when I truly need multiple collectors and can accept its replay/loss semantics.

    ```kotlin
    // Bug: re-navigates every time the collector re-subscribes (e.g. after rotation).
    data class UiState(val shouldNavigateHome: Boolean = false)

    // Correct: events are a separate, single-consumer Channel — never re-delivered.
    class LoginViewModel : ViewModel() {
        private val _events = Channel<LoginEvent>(Channel.BUFFERED)
        val events = _events.receiveAsFlow()

        fun onLoginSuccess() = viewModelScope.launch { _events.send(LoginEvent.NavigateHome) }
    }
    // Collector (Compose):
    LaunchedEffect(Unit) { viewModel.events.collect { event -> if (event is NavigateHome) navController.navigate("home") } }
    ```

!!! warning "Red flag"
    "I put a `showToast: Boolean` in the state." — the exact anti-pattern.

**Follow-ups:** Why not `SharedFlow(replay=0)` by default? · What happens to a `Channel` event if the UI is backgrounded when it's sent?

---

## 8. What is Single Source of Truth, and where do you enforce it?

!!! quote "Strong answer"
    Every piece of state has exactly one authoritative owner. For persisted data I make the *database* the SSOT: the UI observes Room, and network responses are written *through* the DB, never handed straight to the UI. That gives offline support, consistency, and automatic UI updates on refresh for free. Violating SSOT — UI reads from cache *and* network independently — guarantees they drift and you get the "pull to refresh shows different data than what's on screen" bug.

    ```kotlin
    // Violation — two sources the UI can observe independently; they WILL disagree.
    class FeedViewModel(private val api: Api, private val dao: FeedDao) : ViewModel() {
        fun refresh() = viewModelScope.launch { _networkResult.value = api.getFeed() }  // bypasses DB
        val cached: Flow<List<Item>> = dao.observeAll()                                  // second source
    }

    // SSOT — network writes THROUGH the DB; UI only ever reads one place.
    class FeedRepository(private val api: Api, private val dao: FeedDao) {
        fun observeFeed(): Flow<List<Item>> = dao.observeAll()          // the ONLY read path
        suspend fun refresh() { dao.insertAll(api.getFeed()) }          // write-through, DB re-emits
    }
    ```

!!! warning "Red flag"
    "The ViewModel holds the truth and syncs the DB and network manually." — reinvents cache-coherence bugs by hand.

**Follow-ups:** How does write-through-to-DB give offline support? · Where's the SSOT for ephemeral UI state like scroll position?

---

## 9. When is Clean Architecture over-engineering?

!!! quote "Strong answer"
    When the cost of the boundaries exceeds the change pressure they absorb. A small app with a backend you own, a solo dev, a short-lived project — full DTO/entity/domain/UI mapping and a separate domain module is carrying cost for decoupling nobody will exercise. I scale the architecture to the app: collapse DTO and domain into one model, let ViewModels call repositories directly, skip UseCases. The skill isn't applying Clean Architecture — it's knowing which parts to drop and being able to defend each cut.

    ```kotlin
    // Over-engineered for a 2-screen utility app: three models for one shape of data.
    data class UserDto(val id: String, val full_name: String)          // network
    data class UserEntity(val id: String, val name: String)            // domain
    data class UserUiModel(val id: String, val displayName: String)    // presentation

    // Right-sized: one model, @SerializedName absorbs the wire-format mismatch.
    data class User(val id: String, @SerializedName("full_name") val name: String)
    ```

!!! warning "Red flag"
    "Clean architecture is always correct." — no sense of carrying cost.

**Follow-ups:** Which layer would you collapse first on a small app? · How do you retrofit a boundary later if the app grows?

---

## 10. Architect a feature from scratch — walk me through it.

!!! quote "Strong answer"
    I start from the *outside in*: define the `UiState` and the user actions (intents/events) first, because that pins down what the screen must do. Then the ViewModel that owns that state. Then the domain contract — repository interface and any UseCase the ViewModel needs — as pure Kotlin. Then the data implementation: DTOs, Room entities, mappers, and the SSOT/caching policy. I wire dependencies through DI, keep `Context` and framework types at the presentation and data edges, and decide early on MVVM-single-state vs MVI based on screen complexity. Tests follow the domain first because it's pure.

    ```kotlin
    // 1. State + actions FIRST — this alone tells you what the screen must be able to do.
    data class OrderUiState(val order: Order? = null, val loading: Boolean = true, val error: String? = null)

    // 2. Domain contract, pure Kotlin, no Android/network types.
    interface OrderRepository { fun observeOrder(id: String): Flow<Order>; suspend fun refresh(id: String) }

    // 3. ViewModel orchestrates against the CONTRACT, written before any implementation exists.
    class OrderViewModel(private val repo: OrderRepository, id: String) : ViewModel() {
        val state: StateFlow<OrderUiState> = repo.observeOrder(id)
            .map { OrderUiState(order = it, loading = false) }
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), OrderUiState())
    }
    // 4. Data impl (Room + Retrofit) is written LAST — it only has to satisfy the interface above.
    ```

!!! warning "Red flag"
    "I create the Activity and start adding code." — no boundary thinking, no state-first design.

**Follow-ups:** Why state-first instead of data-first? · Where's the first test you write?

---

## 11. A `MainViewModel` has 1,200 lines and every screen's state. How do you fix it?

!!! quote "Strong answer"
    That's a God ViewModel — unmergeable, untestable, unreasonable. I don't make it bigger. I scope it down: one ViewModel per screen/feature, push business logic into UseCases, and hoist genuinely shared state into a scoped repository or a nav-graph-scoped holder rather than a shared ViewModel. I'd do it incrementally — carve out one screen at a time behind its own ViewModel and state — not a big-bang rewrite, so the app keeps shipping.

    ```kotlin
    // Before: one god object every team edits, every merge conflicts on.
    class MainViewModel : ViewModel() {
        val profileState: StateFlow<ProfileUiState> = /* ... */
        val cartState: StateFlow<CartUiState> = /* ... */
        val settingsState: StateFlow<SettingsUiState> = /* ... */
        // ...1200 lines, three teams, one file
    }

    // After: scoped per screen; genuinely shared state moves to a repository, not a shared VM.
    class ProfileViewModel(private val session: SessionRepository) : ViewModel() { /* profile only */ }
    class CartViewModel(private val session: SessionRepository) : ViewModel() { /* cart only */ }
    // SessionRepository is the single owner of "current user" — both VMs observe it, neither owns it.
    ```

!!! warning "Red flag"
    "Split it into a base ViewModel and subclasses." — inheritance to fix a scoping problem; usually makes coupling worse.

**Follow-ups:** Where does state that's genuinely shared across screens live? · How do you migrate without a feature freeze?

---

## 12. Your domain model needs a formatted, localized string. Where does the formatting happen?

!!! quote "Strong answer"
    Not in domain — the instant domain touches `Context`, `Resources`, or a `@StringRes Int` it's no longer pure or JVM-testable. Domain returns a *typed* value: `InsufficientFunds`, a `Money(amount, currency)`, an enum. The presentation layer maps that to a localized string via an injected resource provider. This keeps localization at the edge and lets me test the business decision without a device and without a locale.

    ```kotlin
    // Domain: typed result, zero Android, testable with a plain assertEquals.
    sealed interface TransferResult { object Success : TransferResult
                                       data class InsufficientFunds(val short: Money) : TransferResult }

    // Presentation: maps the typed result to a localized string, only here.
    fun TransferResult.toMessage(resources: Resources): String = when (this) {
        TransferResult.Success -> resources.getString(R.string.transfer_success)
        is TransferResult.InsufficientFunds ->
            resources.getString(R.string.transfer_short_by, short.formatted())
    }
    ```

!!! warning "Red flag"
    "I pass the `Context` into the UseCase to get the string." — leaks the framework into domain and kills testability.

**Follow-ups:** How do you inject string resources into the presentation layer testably? · What breaks if the `@StringRes Int` lives in the domain model?
