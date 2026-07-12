# Leadership & Behavioral

The technical rounds get you *considered*. The behavioral round decides whether you get the **Senior** offer, the **Lead** offer, or a "strong hire but let's bring them in one level down." Strong engineers routinely under-prepare here because it feels like the soft, unscored part of the loop. It is not. At most companies the behavioral/leadership interviewer is explicitly asked to calibrate your *level*, and their write-up carries as much weight as the system-design score.

!!! warning "The trap"
    You can ace the coroutines question, nail the offline-sync system design, and still get down-leveled because your stories were all "we" and no "I", had no metrics, and showed you *doing tasks* instead of *owning outcomes and multiplying a team*. Down-leveling from Lead to Senior is a real, common, and expensive outcome. This section exists to stop that.

## What they actually grade

Behavioral interviewers are listening for signal on seven dimensions. Every question maps to one or more of these.

| Dimension | What "strong" sounds like | What "weak" sounds like |
|---|---|---|
| **Ownership** | "I owned the crash rate for our module and drove it from 1.8% to 0.2%." | "My manager asked me to look into crashes." |
| **Influence without authority** | "I got three teams to adopt the new networking module without being anyone's manager." | "I filed a ticket asking them to switch." |
| **Conflict** | "The staff eng and I disagreed on Room vs SQLDelight; here's how we resolved it with data." | "I just went with what my lead wanted." |
| **Mentoring / growing others** | "I paired with the junior weekly; she now owns the whole payments module." | "I answered questions when people asked." |
| **Judgment / prioritization** | "We *chose not* to fix that tech debt because the module was being deleted in Q3." | "We fixed all the warnings because they were there." |
| **Communication** | Structured, quantified, tailored to the listener (exec vs. IC). | Rambling, context-heavy, no landing. |
| **Impact** | Tied to users, revenue, retention, crash-free rate, build time, team velocity. | Tied to "I finished the tickets." |

!!! tip "The meta-signal: altitude"
    Across all seven, the interviewer is triangulating one thing — **the altitude you naturally operate at.** Do you talk about *your tasks*, *your team's outcomes*, or *the org's health*? Senior candidates who instinctively talk one level up (and can still zoom down to a concrete Android detail on demand) read as Lead. Talk only about your own tickets and you read as a mid-level IC in a senior's clothing.

## The ladder, in behavioral terms

Forget the HR rubric. Here is the ladder the way a hiring committee reads it from your stories.

!!! quote "IC → Senior → Lead"
    **IC does the work.**
    "I was assigned the settings screen. I built it, it shipped, QA passed it." Output is *code*. Scope is *a task*. Success is *my thing works*.

    **Senior owns outcomes and unblocks others.**
    "I owned checkout reliability. I found the ANR was a main-thread disk read in a third-party SDK, worked around it, and cut checkout ANRs 80%. Along the way I unblocked two teammates stuck on the same threading model." Output is *a solved problem*. Scope is *a feature area / a metric*. Success is *the outcome moved, and the people around me moved faster*.

    **Lead multiplies the team.**
    "I saw three squads each hand-rolling their own networking + error handling, shipping inconsistent retry logic. I wrote the RFC, aligned the leads, built the shared module, and migrated us incrementally. Six months later new features ship a week faster and our network-error crash class is gone. I mentored two engineers into owning it so it doesn't depend on me." Output is *leverage* — systems, standards, and people that make everyone else more productive. Scope is *multiple teams / a quarter+ horizon*. Success is *the team is permanently better, with or without me in the room*.

The move from Senior to Lead is **not "does harder tasks."** It's the shift from *additive* (I ship more/better) to *multiplicative* (I make N people ship more/better). Your story bank has to demonstrate the multiplication.

## How to use this section

<div class="grid cards" markdown>

-   **[Driving Technical Decisions (ADRs & RFCs)](decisions-adr.md)**

    How a senior/lead makes and *documents* calls — ADR format, the RFC process, disagree-and-commit, tech debt as a portfolio, and sequencing big migrations (XML→Compose, modularization) without freezing feature work. Includes a full worked ADR.

-   **[The STAR Method, Done Right](behavioral-star.md)**

    Heavy-on-Action, quantified-Result storytelling. The 8 story archetypes to prepare, 3 fully-worked Android STAR answers (Compose migration, an ANR incident, mentoring through modularization), and a copy-paste template for your own stories.

-   **[Practice Questions & Answer Shapes](questions.md)**

    12–15 real behavioral/leadership questions grouped by dimension, each with what they're *really* assessing and the *shape* of a strong answer — plus the questions **you** should ask to interview the company back.

</div>

!!! note "Preparation is not memorization"
    Do not memorize scripts — you'll sound rehearsed and fall apart on follow-ups. Instead: build the **story bank** in [behavioral-star.md](behavioral-star.md) (8 stories, written out once, quantified), then practice *mapping* any question to the right story on the fly. One well-built story usually answers 3–4 different questions.
