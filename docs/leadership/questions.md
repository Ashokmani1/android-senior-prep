# Practice Questions & Answer Shapes

Real behavioral/leadership questions, grouped by the dimension they probe. For each: the **question**, **what they're really assessing**, and a **strong answer shape** — a bullet outline of what a great answer *hits*, not a canned script. Answers must be personal and specific to *your* stories (build them in [behavioral-star.md](behavioral-star.md)); a memorized script dies on the first follow-up.

!!! tip "How to practice"
    Take each question, pick which of your 8 story-bank stories fits, and speak it out loud with the 10/10/60/20 timing. Time yourself — a strong answer is 2–3 minutes, not 6. If you can't land it in 3, your Situation is too long.

## Ownership

!!! note "Q1. Tell me about a project you owned end to end."
    **Really assessing:** scope of ownership — did you own an *outcome/metric* or just *tasks*? Do you think past the code (rollout, monitoring, follow-through)?

    **Strong answer shape:**

    - Frame it as owning an *outcome or metric*, not "I was assigned X."
    - Show the full arc: problem definition → decision/design → execution → **launch → monitoring → iteration**. Juniors stop at "it shipped."
    - Name where you pulled others in and unblocked them.
    - Quantified result + the thing that outlived the project.

!!! note "Q2. Describe a time you saw a problem nobody asked you to fix, and fixed it."
    **Really assessing:** proactivity vs. ticket-taking. This is a pure Lead-vs-Senior discriminator.

    **Strong answer shape:**

    - The problem was *systemic* (flaky CI, a recurring crash class, slow builds), not just a bug you happened to notice.
    - You *chose* to take it on and made the case for the time.
    - You fixed the class, not the instance.
    - Result in team-level terms: hours saved across N people, a regression class eliminated.

!!! note "Q3. Tell me about a time you had to make a decision with incomplete information."
    **Really assessing:** judgment under uncertainty, and comfort being accountable for a call that might be wrong.

    **Strong answer shape:**

    - Name the uncertainty honestly and the deadline that forced a call.
    - Show you gathered what signal you *could* cheaply (a spike, a metric, one expert).
    - You made a **reversible** decision where possible and defined what would make you change course.
    - Result + whether the bet paid off *and* what you'd revisit.

## Conflict

!!! note "Q4. Tell me about a technical disagreement with a coworker. How did it resolve?"
    **Really assessing:** can you disagree productively, separate ego from the decision, and reach commitment without a manager refereeing?

    **Strong answer shape:**

    - The disagreement was *technical and substantive* (e.g. Room vs SQLDelight, MVVM vs MVI), stated fairly from *both* sides — steel-man theirs.
    - You resolved it with **data or a spike**, not seniority or volume.
    - **Disagree-and-commit**: whoever "lost" still committed, because their concern was heard/captured.
    - Result: the decision held, the relationship stayed intact. Bonus: it was documented (ADR).

!!! note "Q5. Tell me about a time you disagreed with your manager."
    **Really assessing:** do you have a spine *and* judgment about when/how to push — and do you commit gracefully when overruled?

    **Strong answer shape:**

    - You disagreed on substance and raised it **directly and privately**, with reasoning and data.
    - You picked the battle — this mattered (a risky architecture call, a burnout-inducing deadline), not a nitpick.
    - Show the outcome *either* way: they changed their mind (you influenced up), *or* they didn't and you **committed fully** without sandbagging.
    - Maturity note: you understood their constraints (business/political) you initially couldn't see.

!!! note "Q6. Tell me about a time you received difficult feedback."
    **Really assessing:** ego, coachability, self-awareness. Down-levels people who get defensive or have no real example.

    **Strong answer shape:**

    - Real, slightly uncomfortable feedback (e.g. "you solve it yourself instead of growing the team", "your reviews are too harsh").
    - Your genuine first reaction, then how you *acted* on it.
    - A concrete behavior change + evidence it stuck.
    - No humble-brag feedback ("I work too hard").

## Mentoring & Influence

!!! note "Q7. Tell me about someone you mentored or helped grow."
    **Really assessing:** the multiplier. Do you invest in others, and can you coach rather than just do-it-for-them?

    **Strong answer shape:**

    - A *specific person* and their starting point.
    - You coached — paired, asked questions, gave stretch work, stepped back — rather than doing it for them.
    - The result is **their trajectory**: promotion, now owns X, unblocks others.
    - What *you* got better at as a mentor.

!!! note "Q8. Tell me about a time you drove adoption of something across teams you don't manage."
    **Really assessing:** influence without authority — the core Lead skill.

    **Strong answer shape:**

    - You had no formal power over the other teams.
    - You led with **evidence and by making adoption easy** (a POC, migration guide, doing the first integration for them), not mandates.
    - You won over a key skeptic early.
    - Result: N teams/engineers adopted it; it became the default.

!!! note "Q9. How do you get a team aligned when opinions are split?"
    **Really assessing:** decision-facilitation — can you converge a group without steamrolling or stalling?

    **Strong answer shape:**

    - Your process: surface all options, **time-box** the debate, ensure every voice is heard.
    - Disagree-and-commit as the closing move; document the rationale.
    - You made or forced the call rather than letting it drift.
    - Balance: decisive *and* inclusive — name both failure modes you avoided.

## Judgment & Failure

!!! note "Q10. Tell me about a time you failed, or a decision you got wrong."
    **Really assessing:** ownership of failure, humility, and whether you extract systemic lessons. A no-real-failure answer is a red flag.

    **Strong answer shape:**

    - A *real* failure with real consequences — you own it, no blaming others or "the requirements changed."
    - What you did to contain it immediately.
    - The **systemic** fix so the *class* of failure can't recur (a check, a process, a test).
    - Honest, specific lesson. Blameless framing.

!!! note "Q11. Tell me about a production incident you handled."
    **Really assessing:** calm under fire, structured debugging, and follow-through beyond the hotfix.

    **Strong answer shape:**

    - You took (or shared) incident lead — didn't wait to be told.
    - Structured triage: mitigate first (stop the bleeding), root-cause second.
    - Android specifics land here: Crashlytics/Play Console traces, ANR vs crash, device/OS clustering, a hotfix, then **prevention** (StrictMode, alerting thresholds, a smoke test).
    - Result: metric recovered + MTTD/MTTR improved + a blameless postmortem.

!!! note "Q12. Tell me about a time you had to miss or move a deadline."
    **Really assessing:** communication under pressure, re-scoping judgment, and stakeholder management — not whether you're superhuman.

    **Strong answer shape:**

    - You saw the slip coming and **raised it early**, not the night before.
    - You came with options: cut scope, add help, move the date — with trade-offs, letting stakeholders choose.
    - You protected quality/the team from a death march where you could.
    - Result: what shipped, and a process change so estimates got better.

## Leadership

!!! note "Q13. How do you decide what your team works on / how do you prioritize?"
    **Really assessing:** business alignment and the ability to say no — the difference between a senior IC and someone who can run an area.

    **Strong answer shape:**

    - Tie work to user/business impact, not "interesting" or "loudest requester."
    - Show a framework: impact vs. effort, and **tech debt as a portfolio** (interest vs. paydown), including debt you *chose not* to fix.
    - You say **no** / defer, and communicate the why.
    - Example where prioritization visibly paid off.

!!! note "Q14. Tell me about a time you improved how your team works."
    **Really assessing:** the multiplier again — do you invest in leverage (process/tooling/culture) over just output?

    **Strong answer shape:**

    - A systemic pain (slow reviews, flaky tests, painful releases, no on-call runbook).
    - You diagnosed root cause, proposed a change, and got buy-in (you didn't just decree it).
    - Adoption + a metric: review turnaround, build time, release frequency, on-call load.
    - It outlived you / became the norm.

!!! note "Q15. What kind of engineer do you want to be, and how do you lead technically?"
    **Really assessing:** self-awareness and whether your self-image matches Lead behavior — leverage, mentorship, decision ownership.

    **Strong answer shape:**

    - Articulate a philosophy centered on **leverage and growing others**, not personal heroics.
    - Concrete habits: writing ADRs, mentoring, seeking dissent, making reversible decisions.
    - Honest about a growth edge you're working on.
    - Grounded in a real example, not aspiration.

---

## Questions to ASK your interviewer

Senior candidates interview the company back. Good questions signal the altitude you operate at and surface whether this is a place a Lead can actually succeed. Pick 3–4, tuned to who's in the room (eng vs. manager vs. skip).

!!! tip "Ask about ownership & architecture"
    - "How are architectural decisions made here — is there an RFC/ADR culture, or is it more ad hoc? Can I see an example?"
    - "Who owns the app's architecture today? Is it centralized in a platform team or owned per feature squad?"
    - "How much of the codebase is on Compose / KMP / modularized, and where's the migration on that journey?"

!!! tip "Ask about on-call & operational health"
    - "What does on-call look like for mobile here? Rotation size, pager volume, and is there a blameless postmortem culture?"
    - "What's your current crash-free rate and how do you monitor and alert on it?"
    - "When there's a production incident, who leads it and how does the team learn from it?"

!!! tip "Ask about decision-making culture & tech debt"
    - "How does the team decide between shipping features and paying down tech debt? Is there dedicated capacity, or is it a constant fight?"
    - "Tell me about a recent technical decision the team disagreed on — how did it get resolved?"
    - "What's the biggest piece of tech debt you know about, and why hasn't it been paid down yet?" *(A revealing question — how they answer tells you about honesty and prioritization maturity.)*

!!! tip "Ask about the role & growth (for a Lead seat)"
    - "What's the scope of this role — how much is hands-on-keyboard vs. leading/mentoring/decisions? And how do you expect that ratio to change over the first year?"
    - "What would success look like at 3, 6, and 12 months for the person in this role?"
    - "How do engineers grow from Senior to Lead/Staff here — what's the actual path and who's done it recently?"

!!! warning "Don't ask"
    Anything trivially Google-able (headcount, funding you can look up), or purely self-serving perks questions in a technical/leadership round — save comp and benefits for the recruiter. Your questions are part of the assessment.
