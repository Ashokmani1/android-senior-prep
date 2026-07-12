# Interview Questions: Architecture

Senior architecture rounds reward **judgment expressed as tradeoffs**. For each question below: the strong answer is what a lead would say, the red flag is the one-liner that ends the interview early, and the follow-ups are what the interviewer already has loaded to test whether your first answer was memorized or understood.

---

## 1. Why layer an app at all? Isn't it just more files?

!!! quote "Strong answer"
    Layers buy **independent rates of change**. UI churns constantly, business rules occasionally, data sources rarely — separating them means a Retrofit-to-Ktor migration never touches a screen and a redesign never touches a business rule. The second payoff is **testability**: pure domain logic runs on the JVM with no emulator. I don't layer reflexively though — a two-screen utility app doesn't need three modules. Layers are a response to *change pressure and team size*, not a default.

!!! warning "Red flag"
    "Because that's clean architecture / best practice." — dogma with no cost awareness.

**Follow-ups:** When would you *not* layer? · How do layers help a five-person team specifically?

---

## 2. Do you always need UseCases?

!!! quote "Strong answer"
    No. A UseCase earns its place when logic is non-trivial (combines repositories, applies rules) or is reused across ViewModels, or when the team wants one enforced convention. A UseCase whose body is a single `repository.getX()` is pure ceremony — a file, an injection, and a test buying nothing. My default: ViewModels may call repositories directly, and I introduce a UseCase the moment logic exceeds one call or needs sharing. I optimize for the next reader deleting indirection, not for diagram symmetry.

!!! warning "Red flag"
    "Every ViewModel must go through a UseCase for every call." — ceremony without judgment.

**Follow-ups:** Your team insists on UseCases everywhere for consistency — how do you argue it? · Where does the logic go if you skip the UseCase?

---

## 3. MVVM or MVI — which do you use?

!!! quote "Strong answer"
    I always want UDF; the question is how much structure I need to enforce it. MVVM with a single `StateFlow<UiState>` is UDF at low ceremony — my default for most screens. I reach for MVI when a screen is complex and stateful enough that impossible states become real bugs, or when a large team benefits from one enforced convention: single immutable state, intents, a reducer. MVI's cost is boilerplate on every screen; I don't pay it for a settings page with two toggles.

!!! warning "Red flag"
    "MVI is more modern so it's better." — treats a tradeoff as a fashion.

**Follow-ups:** What concrete bug does MVI's single state prevent? · What's the cost of MVI on a trivial screen?

---

## 4. Where does business logic live?

!!! quote "Strong answer"
    In the domain layer — UseCases or domain services — never in the ViewModel or the Composable. The ViewModel *orchestrates* (calls domain, shapes results into `UiState`); it doesn't *decide* (apply the discount rule, validate the transfer). The tell for a leak is a business rule you can't unit-test without Android, or the same rule copy-pasted across two screens. If logic is genuinely UI-only — a formatting choice — it belongs in presentation, and that's fine.

!!! warning "Red flag"
    "In the ViewModel." (as the unqualified default) — the classic God-ViewModel trajectory.

**Follow-ups:** How do you keep a discount rule out of the ViewModel? · What logic legitimately *does* belong in presentation?

---

## 5. How do you make a ViewModel testable?

!!! quote "Strong answer"
    Inject its collaborators as interfaces (UseCases or repository interfaces) so I substitute fakes — I prefer hand-written fakes over mocks for state-based tests. Keep business logic out of it so tests assert *state transitions*, not rules. Control coroutines with an injected dispatcher or `StandardTestDispatcher` + `runTest`, and assert on the `StateFlow` with Turbine. Crucially, no `Context`, no `Android` framework types in the ViewModel — the moment those appear, I need Robolectric and the tests slow down and get flaky.

!!! warning "Red flag"
    "I test it with Espresso / on a device." — doesn't distinguish unit from UI testing, or reaches for the heavy tool first.

**Follow-ups:** How do you test a flow that uses `WhileSubscribed`? · Mocks or fakes, and why?

---

## 6. One UiState stream or several?

!!! quote "Strong answer"
    One `StateFlow<UiState>` per screen as the default — the UI observes a single consistent snapshot, and I use `combine` to derive it from multiple sources so the parts can never be momentarily inconsistent. I split into multiple streams only when sections are genuinely independent and have very different update frequencies (a high-frequency scrubber vs. static metadata) and merging them would cause wasteful recomposition. Multiple streams is an optimization I justify, not a starting point.

!!! warning "Red flag"
    "One StateFlow per field." — recreates scattered-observable MVVM and its inconsistency bugs.

**Follow-ups:** How does `combine` keep the state consistent? · When would multiple streams actually help performance?

---

## 7. How do you handle one-off events like navigation or a toast?

!!! quote "Strong answer"
    State answers "what do I render" and must be safe to re-read; one-off events must not be re-read or they double-fire. Putting a `navigate` boolean in `UiState` re-navigates on the next rotation — the classic bug. I model events separately: a `Channel` exposed as `receiveAsFlow()` for single-consumer delivery that survives the config-change collector gap, collected in Compose via `flowWithLifecycle`. I use `SharedFlow` only when I truly need multiple collectors and can accept its replay/loss semantics.

!!! warning "Red flag"
    "I put a `showToast: Boolean` in the state." — the exact anti-pattern.

**Follow-ups:** Why not `SharedFlow(replay=0)` by default? · What happens to a `Channel` event if the UI is backgrounded when it's sent?

---

## 8. What is Single Source of Truth, and where do you enforce it?

!!! quote "Strong answer"
    Every piece of state has exactly one authoritative owner. For persisted data I make the *database* the SSOT: the UI observes Room, and network responses are written *through* the DB, never handed straight to the UI. That gives offline support, consistency, and automatic UI updates on refresh for free. Violating SSOT — UI reads from cache *and* network independently — guarantees they drift and you get the "pull to refresh shows different data than what's on screen" bug.

!!! warning "Red flag"
    "The ViewModel holds the truth and syncs the DB and network manually." — reinvents cache-coherence bugs by hand.

**Follow-ups:** How does write-through-to-DB give offline support? · Where's the SSOT for ephemeral UI state like scroll position?

---

## 9. When is Clean Architecture over-engineering?

!!! quote "Strong answer"
    When the cost of the boundaries exceeds the change pressure they absorb. A small app with a backend you own, a solo dev, a short-lived project — full DTO/entity/domain/UI mapping and a separate domain module is carrying cost for decoupling nobody will exercise. I scale the architecture to the app: collapse DTO and domain into one model, let ViewModels call repositories directly, skip UseCases. The skill isn't applying Clean Architecture — it's knowing which parts to drop and being able to defend each cut.

!!! warning "Red flag"
    "Clean architecture is always correct." — no sense of carrying cost.

**Follow-ups:** Which layer would you collapse first on a small app? · How do you retrofit a boundary later if the app grows?

---

## 10. Architect a feature from scratch — walk me through it.

!!! quote "Strong answer"
    I start from the *outside in*: define the `UiState` and the user actions (intents/events) first, because that pins down what the screen must do. Then the ViewModel that owns that state. Then the domain contract — repository interface and any UseCase the ViewModel needs — as pure Kotlin. Then the data implementation: DTOs, Room entities, mappers, and the SSOT/caching policy. I wire dependencies through DI, keep `Context` and framework types at the presentation and data edges, and decide early on MVVM-single-state vs MVI based on screen complexity. Tests follow the domain first because it's pure.

!!! warning "Red flag"
    "I create the Activity and start adding code." — no boundary thinking, no state-first design.

**Follow-ups:** Why state-first instead of data-first? · Where's the first test you write?

---

## 11. A `MainViewModel` has 1,200 lines and every screen's state. How do you fix it?

!!! quote "Strong answer"
    That's a God ViewModel — unmergeable, untestable, unreasonable. I don't make it bigger. I scope it down: one ViewModel per screen/feature, push business logic into UseCases, and hoist genuinely shared state into a scoped repository or a nav-graph-scoped holder rather than a shared ViewModel. I'd do it incrementally — carve out one screen at a time behind its own ViewModel and state — not a big-bang rewrite, so the app keeps shipping.

!!! warning "Red flag"
    "Split it into a base ViewModel and subclasses." — inheritance to fix a scoping problem; usually makes coupling worse.

**Follow-ups:** Where does state that's genuinely shared across screens live? · How do you migrate without a feature freeze?

---

## 12. Your domain model needs a formatted, localized string. Where does the formatting happen?

!!! quote "Strong answer"
    Not in domain — the instant domain touches `Context`, `Resources`, or a `@StringRes Int` it's no longer pure or JVM-testable. Domain returns a *typed* value: `InsufficientFunds`, a `Money(amount, currency)`, an enum. The presentation layer maps that to a localized string via an injected resource provider. This keeps localization at the edge and lets me test the business decision without a device and without a locale.

!!! warning "Red flag"
    "I pass the `Context` into the UseCase to get the string." — leaks the framework into domain and kills testability.

**Follow-ups:** How do you inject string resources into the presentation layer testably? · What breaks if the `@StringRes Int` lives in the domain model?
