# Android Senior & Lead Interview Kit

A personal, extensible knowledge base for cracking **Senior / Lead Android** interviews —
the rounds that a fundamentals-only resource (e.g. *Manifest Android Interview*) doesn't cover:
**app architecture, modularization, mobile system design, and technical leadership**.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## Run locally

```bash
pip install -r requirements.txt
mkdocs serve            # http://127.0.0.1:8000
```

## Build a static site

```bash
mkdocs build            # outputs to ./site
```

## Add a new topic (extensibility)

1. Create `docs/<topic>/index.md` (topic overview).
2. Add pages under `docs/<topic>/`.
3. Register the topic in the `nav:` block of `mkdocs.yml`.

Each topic folder is self-contained, so the kit grows without touching existing content.

## Content model

Every topic follows the same shape so it stays skimmable under interview pressure:

- **index.md** — why it matters, the mental model, what interviewers probe.
- **deep-dive pages** — the concepts, with diagrams and Kotlin.
- **questions.md** — real interview questions with strong (and weak) answers + follow-ups.

## Roadmap

- [x] Prep strategy + study plan
- [x] App Architecture
- [x] Modularization
- [x] System Design (mobile)
- [x] Leadership & Behavioral
- [ ] Build & Gradle performance at scale
- [ ] Testing strategy (pyramid, fakes, screenshot, macrobenchmark)
- [ ] Coroutines & Flow deep dive
- [ ] Performance: startup, jank, memory, baseline profiles
- [ ] Kotlin Multiplatform (KMP) for shared logic
