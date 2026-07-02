# C14 — Anatomy of a feature: how Membership 1.0 was actually built

> **Goal:** the war stories (→ C12) teach you to *diagnose*; this doc teaches you to
> *construct*. It walks the real build of our core feature — membership-gated
> enrollment — as a repeatable method you can apply to the next feature. Nothing here is
> hypothetical; this is the order it actually happened in, including why.

## 1. Start from the invariant, not the UI

The feature began as one sentence: *"a user whose membership rank is below a course's
required rank must not end up enrolled — no matter which door they came through."*

Notice what that sentence is: an **invariant** (a thing that must always be true), not a
feature list. Writing it first decided half the architecture — "no matter which door"
immediately implies you can't just build a nicer enroll button; you need something that
watches *all* enrollments (→ signals, C03 §4).

**Method step 1: write the invariant in one sentence. If you can't, you're not ready to
model data.**

## 2. Spec before code, tasks before spec-drift

The full design went into a spec doc (`SQA_PLUGIN_CONTEXT.md`: 3 models, 4 waffle
switches, ~9 endpoints, exact code templates), then was broken into numbered files in
`tasks/`, each with a `Status:` field, acceptance criteria, and a `Depends on:` graph.

Why bother, solo? Two reasons that paid off repeatedly: **(a)** a task file is a
contract with your future self — "done" is checkable, not vibes; **(b)** when reality
diverged from the spec (it did — several corrections now live in CLAUDE.md), the
divergence got *written down* instead of living in one person's head.

## 3. Build order: plumbing → data → truth → doors → net

The actual sequence, and the reasoning that generalizes:

1. **Plumbing first** (the `apps.py` + entry-point wiring, task-03). Until the LMS
   *loads* the plugin, nothing else is testable in context. Prove the skeleton mounts
   before decorating it.
2. **Models + migrations.** Data shape is the hardest thing to change later (→ C13 §3),
   so it gets designed while everything is still cheap: `MembershipLevel`,
   `UserMembership`, `CourseTierRequirement`.
3. **The single source of truth**: `check_enrollment_eligibility(user, course_key)` in
   `utils.py`. One pure-ish function holding the rank-comparison rule, written and
   unit-tested *before* any view or signal existed. Every later component **imports** it;
   none re-implements it. This is the load-bearing design decision of the whole plugin:
   rules that exist twice *will* drift.
4. **The doors** — REST endpoints (`api.py`), including the proactive enroll view that
   403s ineligible users politely.
5. **The safety net** — the enrollment-created signal handler that unenrolls anything
   ineligible that slipped through any other door. Built *last*, because it's the
   backstop: by the time it fires, three other layers already agreed.

**Method step 3 in one line: build from the inside out — data, then truth, then doors —
so each layer only depends on already-tested layers.**

## 4. Ship dark, enable gradually (the waffle discipline)

Every behavior group went behind a waffle switch, **all defaulting OFF**
(`api_membership`, `api_enrollment`, `gate_enrollment`, `auto_assign`). So the deploy
that shipped Membership to production changed *nothing* for users — the code was dark.

Then switches came on **one at a time, observing between each**: read-only APIs first
(worst case: a broken JSON endpoint), enrollment APIs, then actual gating (worst case:
students blocked — the risky one went last), then auto-assignment.

This converts one big scary launch into four small reversible ones, each with a
10-second, no-deploy kill switch. It also decouples "deploy day" from "launch day" —
pressure never stacks. (→ C03 §4 for the concept.)

## 5. Test at the cheapest tier that can catch the change

Three tiers (→ C06 §4), used deliberately:

- **Tier 1 — `pytest` + SQLite, no LMS** (seconds): the eligibility rule, models, view
  logic with the LMS's `enrollment_api` stubbed. ~80% of all changes never needed more
  than this — which is exactly why the eligibility logic was designed as an importable
  function instead of living inside a view: **testability at tier 1 was an architectural
  input, not an afterthought.**
- **Tier 2 — `tutor dev`** (minutes): the things tier 1 *cannot* see — does the entry
  point load? do real signals fire? does real OAuth work?
- **Tier 3 — k8s deploy** (tens of minutes): only what needs production truth — real
  Stripe webhooks over the public internet, the real image, real config flow.

The skill being trained: before testing anything, ask *"what is the cheapest tier that
can falsify this change?"* — and only escalate when the answer genuinely is "the next
tier up".

## 6. The leftovers that made it durable

- A **bootstrap management command** creating switches + default levels — so a fresh
  environment reaches a known state with one command instead of a wiki page of clicks.
- **Reference patterns before writing** (the `openedx-plugin-example` repo): each file
  was written *after* reading a working example of that file's job — including noting a
  known bug in the example so we didn't copy it. Cheap humility, big payoff.
- **Docs updated in the same session** as the code (CLAUDE.md corrections, runbooks,
  these concepts) — because a decision not written down within a day becomes archaeology
  within a month.

## 7. The reusable checklist (the whole doc in 10 lines)

For any new feature on this platform:

1. Write the invariant in one sentence.
2. Spec it; break it into checkable tasks with dependencies.
3. Wire the plumbing; prove it loads.
4. Design models + **additive** migrations (→ C13).
5. Put every rule in exactly one importable, tier-1-testable place.
6. Build the happy-path doors (API/UI).
7. Build the safety net (signals) for the doors you don't control.
8. Gate everything behind default-OFF switches; ship dark.
9. Test each change at the cheapest tier that can catch it.
10. Enable gradually, observe between steps, keep the kill switch warm.

## You're ready when…

- You can explain *why* the eligibility function came before the API, and what breaks
  (slowly, later) if the rule lives in two places.
- You can plan the next feature (say, membership expiry) as steps 1–10 on paper without
  opening an editor.
- "Deploy" and "launch" are two different words in your vocabulary now.
