# Architecture

At senior level, architecture is the axis the whole loop rotates around. Junior interviews test whether you can make a screen work. Senior/lead interviews test whether you can make a **team** ship features for three years without the codebase collapsing under its own coupling. Nobody cares that you can recite "MVVM." They care whether you can defend a boundary decision under pushback, know when a pattern is ceremony, and reason about the second-order costs of every abstraction you add.

## The mental model

Think in **boundaries and flows**, not in named patterns. Every architecture question reduces to two things:

1. **Where does each responsibility live, and what is allowed to depend on what?** (dependency direction, separation of concerns)
2. **How does state move through the system, and where is the one true copy?** (unidirectional data flow, single source of truth)

Everything else — MVVM vs MVI, whether you have a `domain` module, whether a UseCase exists — is an *implementation choice in service of those two questions*. A senior candidate reframes pattern questions ("do you use Clean Architecture?") into boundary questions ("what forces would justify that boundary here?"). That reframing is the signal interviewers are listening for.

## What interviewers actually probe

!!! note "The senior signal"
    They are not checking whether you know the pattern. They are checking whether you know **when the pattern is wrong**. The strongest answers name a tradeoff and pick a side with a reason. The weakest answers recite a diagram from a blog post.

- **Judgment over dogma.** "It depends" is only a good answer if you immediately say *what it depends on*. Can you name the specific force (team size, testability need, churn rate, feature volatility) that tips the decision?
- **Cost awareness.** Every layer, module, and abstraction has a carrying cost — more files, more mapping, slower onboarding. Do you charge for abstractions or add them reflexively?
- **Testability as a design driver.** Can you explain how a boundary makes a thing testable, and what you give up if you skip it?
- **Failure modes.** God ViewModels, boolean soup in UI state, `Context` leaking into domain, business logic smeared across the UI. Do you recognize the smells and know the refactor?
- **Scaling to a team.** Does your architecture let five engineers work in parallel without merge wars, or does it funnel everyone through one `MainViewModel`?

## Explore

<div class="grid cards" markdown>

-   **[Clean Architecture](clean-architecture.md)**

    Layers, the dependency rule, when UseCases earn their keep, repository pattern, model mapping, and the Android-specific traps.

-   **[Patterns: MVVM vs MVI vs UDF](patterns.md)**

    Precise definitions, the same screen in both styles, the side-effect gotcha, and when to pick which.

-   **[State & Errors](state-and-errors.md)**

    Modeling `UiState` without boolean soup, `Result`/`Either` for domain errors, combining flows, and why `WhileSubscribed(5000)`.

-   **[Interview Questions](questions.md)**

    10+ real senior questions with strong answers, red-flag answers, and the follow-ups the interviewer has loaded.

</div>

## Principles that survive every framework churn

Frameworks rot — AsyncTask, Loaders, RxJava, LiveData, now Flow and Compose. The principles underneath do not. **Separation of concerns** keeps UI, business rules, and data access independently changeable, so a Retrofit-to-Ktor swap never touches a screen. **Unidirectional data flow** makes state changes traceable in one direction — event down, state up — killing the "who mutated this?" class of bugs. **Single source of truth** means every piece of state has exactly one owner, so the disk, the network cache, and the UI can never silently disagree. **Testability** falls out for free when dependencies point inward toward pure logic you can exercise without a device. And the **dependency rule** — inner layers never know about outer ones — is what makes all of the above hold under pressure instead of eroding the first time someone is in a hurry. Learn these five and you can walk into any codebase, in any framework, five years from now, and know where things belong.
