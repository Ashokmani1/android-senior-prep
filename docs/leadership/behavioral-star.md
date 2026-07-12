# The STAR Method, Done Right

STAR — **Situation, Task, Action, Result** — is the structure every behavioral answer should follow. Most engineers know the acronym and still bomb, because they spend 90% of the answer on Situation and Task (the easy, comfortable context) and almost nothing on **Action** (what *you specifically* did) and **Result** (the quantified outcome). This page fixes the ratio, gives you a story bank to build, and works three Android examples end to end.

## The right proportions

!!! tip "The 10 / 10 / 60 / 20 rule"
    - **Situation — ~10%.** One or two sentences of context. Where, when, what was at stake. Stop.
    - **Task — ~10%.** Your specific responsibility / the problem *you* had to solve. One sentence.
    - **Action — ~60%.** The heart of it. Concrete, sequential, first-person: *"I profiled… I found… I proposed… I built… I convinced…"* This is where your level shows.
    - **Result — ~20%.** Quantified outcome + what you learned + the lasting change. Numbers are non-negotiable.

## The three mistakes that get people down-leveled

!!! warning "Kill these"
    - **Too much context.** Three minutes of backstory about the org chart and the product. The interviewer's attention is gone before you did anything. Cut Situation to two sentences.
    - **"We" instead of "I".** "We decided, we built, we shipped." The interviewer literally cannot tell what *you* did — and their job is to assess *you*. Use "we" for context, then switch hard to "I" for your actions. It's not arrogant; it's the assignment. (When you genuinely led others, *that's* the multiplier signal — say "I got the team to…", not "we happened to…".)
    - **No result, or no metric.** "…and then it shipped and it was good." Useless. Every story ends in a number: crash-free rate, p90 latency, build time, adoption %, retention, revenue, hours saved per week, people leveled up. If you truly can't quantify, quantify the *effort or scale*: "used by all 40 engineers", "eliminated a class of bug we'd hit ~monthly".

## The story bank: 8 archetypes to prepare

Interview questions are infinite; the underlying stories are not. Prepare these **eight** written out once, and you can map almost any behavioral question onto one. Aim for stories from the last 2–3 years, ideally where you operated *above* your title.

1. **A hard technical decision** — a real trade-off you drove (and documented). *→ influence, judgment.*
2. **A conflict with a coworker or manager** — technical or interpersonal, resolved constructively. *→ conflict, communication.*
3. **A failure / incident you owned** — a production issue, a bad call, a missed thing — where you took responsibility and fixed the system, not just the symptom. *→ ownership, humility.*
4. **Mentoring someone** — a specific person you grew, with their trajectory as the result. *→ growing others.*
5. **Influencing without authority** — got people you don't manage to change course. *→ influence, the Lead multiplier.*
6. **A project you led end to end** — you owned scope, execution, and outcome. *→ ownership, delivery.*
7. **Missing a deadline / delivering under pressure** — how you handled the slip, communicated, and re-scoped. *→ judgment, communication.*
8. **Improving team process or quality** — a systemic fix (CI, review culture, testing, on-call) that outlived the immediate problem. *→ the multiplier, again.*

!!! note "Overlap is the point"
    One great story often covers several archetypes. A Compose migration you led can serve as *hard decision*, *influence without authority*, *led end to end*, and *process improvement* — you just emphasize a different slice depending on the question. Build 8; expect to use 4–5 heavily.

---

## Worked example 1 — Leading a Compose migration (influence + led end-to-end)

!!! example "\"Tell me about a time you drove a significant technical change.\""
    **(S)** Our app was ~120 XML screens, and Compose had just gone stable. New hires were slow to ramp because our custom-view layer was undocumented, and UI bugs were our #2 crash source.

    **(T)** I wasn't the manager — I was one of four senior Android engineers — but I believed we needed to move to Compose, and I took it on myself to make that happen without a feature freeze the PM would never approve.

    **(A)** First I built a proof of concept: I rewrote our *new* referral screen in Compose and measured it — 40% fewer lines, and I had it done in two days versus my estimate of four in XML. Then I wrote an **RFC** laying out three options: full rewrite (rejected — too risky, freezes features), stay on XML (rejected — bleeding new hires), and an **incremental strangler migration**, which I recommended. I circulated it, and the loudest skeptic was a staff engineer worried about interop cost. I met with him one-on-one, we spiked `ComposeView`/`AndroidView` interop together, and I captured his performance concerns directly in the RFC's Consequences section — that turned him into an advocate. Once we aligned, I set the rule: *new screens are Compose, old screens convert only when a feature already reopens them.* I built the shared Compose design-system module first so every converted screen was consistent, added a "% screens on Compose" dashboard to our CI, and ran a weekly 30-minute office hour to unblock people learning it.

    **(R)** Eighteen months in we went from 0 to 85% of screens on Compose **with zero feature-freeze weeks** — the PM never had to trade a roadmap item for it. UI-layer crashes dropped ~60%, and new-hire time-to-first-PR went from ~two weeks to four days because Compose + the design system were self-documenting. The dashboard and office-hours pattern got adopted by the iOS team for their SwiftUI migration. The biggest lesson: winning over the one credible skeptic early did more for adoption than any amount of broadcasting.

---

## Worked example 2 — Owning a production ANR / crash spike (ownership + failure)

!!! example "\"Tell me about a production incident you were responsible for.\""
    **(S)** After a release, our crash-free-users rate dropped from 99.4% to 97.1% overnight and Play Console flagged an ANR cluster on the checkout screen. Revenue-critical path, and it was *my* feature area.

    **(T)** I owned checkout reliability. I made myself incident lead — I didn't wait to be assigned — because I knew that code best.

    **(A)** I pulled the Play Console ANR traces and Firebase Crashlytics logs and saw the ANRs clustered on mid-tier Android 11 devices. The stack pointed into a third-party payments SDK we'd upgraded that release. I reproduced it by throttling I/O in the emulator: the new SDK version did a **synchronous disk read on the main thread** during init. I couldn't patch their code, so short-term I moved SDK initialization off the main thread and behind a loading state, and shipped a **hotfix within the day**. Then, so this couldn't silently recur, I did three systemic things: added a **StrictMode** main-thread-I/O check that fails in debug/CI, added a Crashlytics **alerting threshold** that pages us if crash-free drops below 99% (we'd been finding out from Play Console, too late), and wrote a **postmortem ADR** — blameless — documenting that we now smoke-test SDK upgrades on a low-end device before release. I also filed the bug with the SDK vendor with my repro.

    **(R)** Crash-free recovered to 99.5% within 48 hours of the hotfix. The StrictMode gate has since caught two more main-thread-I/O regressions *before* they shipped. The alerting cut our mean-time-to-detect on crash spikes from ~a day to under an hour. What I took away: the fix isn't done when the symptom is gone — it's done when the *class* of failure can't reach production silently again.

---

## Worked example 3 — Mentoring a junior through a modularization effort (growing others)

!!! example "\"Tell me about someone you developed.\""
    **(S)** We had a junior engineer, ~1 year in, strong coder but stuck on small tickets and hesitant to touch architecture. Meanwhile our single-module app had a 9-minute clean build that was killing everyone's flow.

    **(T)** As the senior on the team I wanted two outcomes at once: cut the build time *and* grow her into someone who owns architectural work — not just do the modularization myself, which would've been faster in the short run.

    **(A)** I scoped the modularization as *her* project and made myself the support, not the driver. I paired with her to write the first ADR together — extracting a `:core:designsystem` module — so she learned to reason about module boundaries and Gradle dependency direction. Then I deliberately **stepped back**: she drove the next extractions, I reviewed her PRs with questions ("what happens to build time if `:core:network` depends on `:feature:cart`?") instead of answers. When she got the Gradle convention-plugin setup wrong, I let her debug it with hints rather than fixing it, then had her present the fix at our team sync so she got the visibility. I checked in weekly, mostly to remove blockers and to tell her manager how well she was doing.

    **(R)** We modularized into ~12 modules; incremental builds dropped from 9 minutes to ~2, across a team of eight — so that's roughly 6 minutes × many builds × 8 people saved every day. But the real result is *her*: she now **owns our modularization standards and reviews other people's module PRs**, and was promoted to mid-level with this as the centerpiece of her packet. I learned that the slower path — coaching instead of doing — is what actually scales me, because now there are two of us who can drive this work.

---

## Fill-in template — build your own stories

Copy this per story. Write it out fully once; you'll internalize it and won't need the script live.

!!! note "Story template"
    ```
    STORY TITLE: ___________________  (archetypes it covers: __________)

    SITUATION (≤2 sentences — context + stakes):
      ...

    TASK (1 sentence — MY specific responsibility / the problem I owned):
      I ...

    ACTION (the 60% — bullet the concrete steps, all first-person "I"):
      - I ...  (what I investigated / measured)
      - I ...  (the decision I made and why)
      - I ...  (how I built / drove / convinced)
      - I ...  (how I handled the hard part / the pushback)
      - I ...  (the systemic thing I did so it stuck)

    RESULT (the 20% — quantified + lasting change + what I learned):
      - Metric moved: from ___ to ___ (crash-free / build time / adoption / retention / revenue / hours saved)
      - Lasting change: (a standard, a tool, a person leveled up, a doc that outlived me)
      - What I learned: one honest sentence.

    LIKELY FOLLOW-UPS I should be ready for:
      - "What would you do differently?"  ->
      - "What was the hardest part?"  ->
      - "How did the people who disagreed react?"  ->
    ```

!!! tip "Rehearse the follow-ups, not just the story"
    The main answer is scored, but the *follow-ups* are where level gets confirmed. "What would you do differently?" with a thoughtful, specific answer signals reflection and growth — a Lead trait. Have one real answer ready for each story; never say "honestly, nothing."
