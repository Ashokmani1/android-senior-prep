# Kotlin

The language round is where fundamentals interviews are quietly won or lost — Senior/Lead
candidates are expected to know *why* a Kotlin feature exists and what it compiles to, not just
its syntax.

<div class="grid cards" markdown>

-   :material-language-kotlin: **[Language Core](fundamentals.md)**

    Null-safety, `val`/`var`, data/sealed/enum/value classes, objects & companions, generics &
    variance (`in`/`out`/`reified`), visibility, equality, smart casts.

-   :material-function-variant: **[Idioms & Functional](idioms.md)**

    Scope functions, delegation & delegated properties, `lazy`, collections vs sequences, inline/
    `reified`/`crossinline`, extensions, operators, destructuring, DSLs, Java interop.

</div>

## Why interviewers probe Kotlin deeply

| They ask | They're really checking |
|---|---|
| "Why `sealed` over `enum`?" | Do you model state with the type system? |
| "What does `inline` actually do?" | Do you understand the cost of lambdas / reification? |
| "`List` vs `Sequence` here?" | Do you reason about allocation and laziness? |
| "How does `by lazy` work?" | Do you know delegation & thread-safety modes? |
| "Data class `copy` + `equals` gotchas?" | Do you know what the compiler generates? |

!!! tip "Pairs with the Coroutines & Flow modules"
    Coroutine *language* mechanics (suspend/CPS, structured concurrency) live in
    [M12 Coroutines](../deep-dive/12-coroutines.md) and [M13 Flow](../deep-dive/13-flow.md).
    This section is the rest of the language.

!!! note "Extensible"
    A **DSA / coding-round** section can slot in here later — say the word and it's added
    the same way.
