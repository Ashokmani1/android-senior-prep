# Driving Technical Decisions (ADRs & RFCs)

At Senior/Lead level, the interviewer isn't checking whether you *have opinions* about Room vs SQLDelight. They're checking whether you can **drive a decision through a team** — gather input, make a call under uncertainty, get people who disagreed to commit, write it down so it survives you, and revisit it with data instead of ego. This page is the playbook, and it doubles as fuel for your "hard technical decision" and "disagreed with a coworker" stories.

## The ADR: your seniority signal in one artifact

An **Architecture Decision Record** is a short, dated, immutable document capturing *one* significant decision and *why*. The format is deliberately tiny:

!!! note "ADR format"
    - **Title** — `ADR-014: Adopt MVI for complex screens` (numbered, so they form a timeline)
    - **Status** — Proposed / Accepted / Superseded by ADR-021 / Deprecated
    - **Context** — the forces at play: constraints, requirements, what hurts today. No solution yet.
    - **Decision** — the call, stated in the active voice: "We will…"
    - **Alternatives considered** — the *other* real options and why they lost. This is the section juniors skip and seniors obsess over.
    - **Consequences** — what becomes easier, what becomes harder, what we're now on the hook for. Include the negative ones.

**Why writing ADRs signals seniority:**

- It proves you think in **trade-offs, not verdicts.** "Alternatives considered" and honest "Consequences" show you saw the whole board.
- It creates **organizational memory.** Six months later when someone asks "why don't we just use LiveData here?" the answer is ADR-014, not a Slack archaeology dig or a hallway myth.
- It **decouples the decision from the person.** The team commits to a documented rationale, not to whoever argued loudest. That's the difference between a lead and a loud senior.
- It makes decisions **falsifiable.** A dated ADR with expected consequences is a bet you can grade later — which is how you revisit with data.

!!! tip "Interview move"
    When you tell a "hard decision" story, name the artifact: *"I wrote it up as an ADR so the two engineers who'd pushed for the other option could see their concerns captured in Consequences — that's what got them to commit."* That single sentence signals process maturity most candidates never demonstrate.

## RFCs: for the decisions too big for an ADR

An ADR records a decision. An **RFC (Request for Comments)** is the *process* you run to reach a big, cross-cutting one — a full modularization strategy, adopting KMP, replacing your analytics stack. RFC first (gather comment, iterate), ADR second (record the landed decision).

| | ADR | RFC |
|---|---|---|
| Scope | One decision, one team | Cross-cutting, multi-team, or high-risk |
| Length | Half a page | Several pages, problem statement + options + rollout |
| Lifecycle | Written *after* alignment, immutable | Written *to reach* alignment, commented on, revised |
| Audience | Future engineers | Everyone affected, right now |

RFC flow that works: **write the problem statement → circulate a draft with 2–3 real options → time-box comments (e.g. one week) → hold one synchronous meeting to close open threads → make the call → record it as an ADR.** The time-box is the lead skill: RFCs that stay open forever are a failure mode, not thoroughness.

## Running a decision with a team

The mechanical loop, and the one to describe in interviews:

1. **Gather input** — actively pull from the people who'll maintain it and the people who disagree. Silence is not consensus; go get the dissent on purpose.
2. **Disagree and commit** — you will not get unanimity, and waiting for it is a decision to stall. Make sure every objection was *heard and captured*, then ask for commitment to the chosen path even from those who preferred another. People commit to decisions they feel they influenced, even when they lost.
3. **Make the call** — own it explicitly. "I'm making the call: we go with X. Here's why, here's what would make me reverse it."
4. **Document it** — ADR. The rationale outlives the memory of the meeting.
5. **Revisit with data** — set a checkpoint ("in 6 weeks, if build time hasn't dropped 20%, we reassess"). Revisiting on evidence is strength, not flip-flopping. Superseding your own ADR-014 with ADR-021 because the data came back different is one of the most senior things you can show.

!!! warning "The two failure modes"
    **Analysis paralysis** — endless RFC, no decision, team blocked. And **the benevolent dictator** — you decide everything fast but nobody's bought in, so execution is half-hearted and every decision re-litigates itself. Leads live in the middle: decisive *and* inclusive. Say so.

## Tech debt as a portfolio, not a backlog

Seniors fix tech debt. Leads *manage a portfolio* of it and know which debt to **never** pay down.

Treat each piece of debt like a loan with an **interest rate** (how much it slows you *per unit time* — friction on every change, extra bugs, onboarding drag) and a **paydown cost** (effort to fix). Then:

- **High interest, low paydown** → fix now. (A flaky test everyone reruns 3× a day.)
- **High interest, high paydown** → schedule and sequence it; this is where migrations live.
- **Low interest, low paydown** → fix opportunistically, while you're in the file (boy-scout rule).
- **Low interest, high paydown** → **do not fix it.** Document why and move on.

!!! example "When NOT to fix it"
    A gnarly, untested legacy `SyncService` full of callbacks. Terrifying to touch — but it's stable, rarely changes, and the module is scheduled for deletion when the new sync engine ships in Q3. Paying it down is *lighting money on fire*. The senior move is to write one line in the ADR/backlog — "not fixing: low change frequency, module EOL Q3" — and walk away. Knowing what to *not* do is a top-tier judgment signal; volunteering it unprompted in an interview lands hard.

## Build-vs-buy and migration decisions

**Build vs buy** (in-app: build a shared module vs adopt a library/SDK): weigh it on *total cost of ownership*, not just today's effort. A library saves you now but costs you in version churn, size, policy risk (an ad/analytics SDK can get your app pulled), and lost control. Building costs now but you own the surface. State the axis you're optimizing: time-to-market? control? binary size? Frame the answer around which one the business needs *this quarter*.

**The big Android migrations** — XML→Compose, Java→Kotlin, monolith→modular — share one rule: **you almost never get a feature freeze, so you migrate incrementally via the strangler pattern.** Stand the new thing up *beside* the old, route new work through it, and starve the old one until it can be deleted.

!!! example "Sequencing a Compose migration without freezing features"
    1. **Beachhead** — pick one *new, low-risk, leaf* screen and build it in Compose. Prove the tooling, theming, and CI work. Don't start with checkout.
    2. **Interop, both directions** — `ComposeView` to drop Compose into existing XML screens; `AndroidView` for the few legacy custom views you need inside Compose. This is what lets old and new coexist so features keep shipping.
    3. **Convert on touch** — new screens are Compose by default; existing screens get converted *when a feature already requires opening them up*. You piggyback migration on planned work instead of asking for a migration quarter nobody will fund.
    4. **Shared foundation early** — design system / theme in Compose *first*, so every converted screen is consistent and you're not re-deciding spacing per screen.
    5. **Track and time-box** — a dashboard of "% screens on Compose" and a target date keeps a strangler migration from stalling at 60% forever (the classic failure — the half-migrated codebase where every dev must know *both* systems).

    Same shape for **Java→Kotlin** (convert file-by-file on touch; interop is free) and **modularization** (extract a `:core:*` module for the most-shared, least-coupled code first — usually design system or networking — validate the build-time win, then peel off feature modules behind an app-level nav graph).

## Good vs poor technical decision-making

| Signal | Poor (down-leveled) | Strong (Lead) |
|---|---|---|
| Basis | "It's the modern/popular way." | Tied to a constraint or metric: build time, crash rate, team velocity. |
| Alternatives | Only the chosen option discussed. | 2–3 real alternatives, each with why it lost. |
| Trade-offs | "It's just better." | Names what got *worse* and who's on the hook for it. |
| Dissent | Avoided or steamrolled. | Sought out, captured, converted to commitment. |
| Documentation | In someone's head / a Slack thread. | An ADR the next engineer can find. |
| Reversibility | "We decided, it's done." | Defined the data that would make them reverse it. |
| Scope of thinking | This feature. | This feature + maintenance + the team + next 2 quarters. |
| Debt | Fix everything, or ignore everything. | Prioritized by interest×paydown; explicitly declines some. |

!!! tip "The one-liner that reads as senior"
    "I try to make decisions *reversible and documented* — cheap to walk back if the data disagrees, and written down so we don't re-argue them. The expensive one-way-door decisions get an RFC; everything else gets an ADR and we move."
