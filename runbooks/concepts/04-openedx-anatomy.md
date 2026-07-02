# C04 — Open edX anatomy: one platform, two generations

> **Goal:** know the parts of Open edX well enough to predict *where* any given feature
> or bug lives — and understand the historical reason the platform feels like two
> products glued together (because it is).

## 1. The core split: LMS vs CMS

One codebase, `edx-platform`, runs as **two separate processes**:

- **LMS** — the student-facing site: accounts, enrollment, courseware delivery, grades,
  certificates. Our plugin loads *here* (`lms.djangoapp` entry point).
- **CMS ("Studio")** — the author-facing site: course creation and editing.

They share databases but are configured independently. When you toggle a waffle switch or
install a plugin, always ask: *LMS, CMS, or both?* Ours: LMS only.

## 2. The two UI generations (and why you must always know which one you're in)

**Generation 1 — "legacy" pages.** Server-rendered: Django views + **Mako templates**
(an older template language) + SCSS compiled *into the openedx image*. Still serving: the
index page, course discovery, the old dashboard, course-about pages, wiki, certificates,
instructor dashboard.

**Generation 2 — MFEs.** Standalone React apps, one repo each (`frontend-app-profile`,
`frontend-app-account`, `frontend-app-learning`, our `frontend-app-sqa-payment`, …),
served as static bundles from the `mfe` pod under `apps.lms.…`. Already own: login/registration,
profile, account settings, the course player, discussions, the learner dashboard.

The migration from 1 → 2 has been running for years and will continue for years. **For us
that's not trivia — it's load-bearing:** every user-facing concern (theming, custom UI,
even "where do I look for this bug") has to be solved **twice**, once per generation.
That's why the theme has two pipelines (→ concepts/08) and why runbook 10 opens with a
"which surface is this?" table.

## 3. The MFE support system

Three shared pieces make ~15 separate React repos behave like one product:

- **frontend-platform** — the runtime every MFE boots on: authentication (reads the JWT
  cookie), configuration loading (the `mfe_config` API call from → concepts/02), logging,
  i18n.
- **Paragon** — the shared React component library (buttons, forms, modals) — *and* the
  design-token system that lets us restyle every MFE from one place (→ concepts/08).
- **Plugin slots** — named "holes" in MFE pages where a deployment can inject its own
  React components without forking the MFE. We use these for our dashboard widgets
  (→ runbook 09; the wiring guide is `MFE_COMPONENT_GUIDE.md`).

The customization ladder for MFEs, cheapest first: **runtime config** (colors, URLs, flags —
minutes) → **slot injection** (add UI — mfe image rebuild) → **custom MFE** (whole new app,
like sqa-payment) → **fork an existing MFE** (last resort: you now own its upgrades
forever; we have avoided this entirely).

## 4. Courses and content

- Courses are authored in Studio and stored in **MongoDB** (content) + **MySQL**
  (relational things: enrollments, grades, users).
- **OLX** is the XML export format — a course is a directory tree you can zip, version,
  and re-import (our Silly Bot course is one — imports via Studio, → runbook 05).
- **XBlocks** are pluggable course components (video, problem, HTML…). The codejail pods
  exist to run student-submitted Python from problem XBlocks *in a sandbox*, because
  running strangers' code inside the LMS process would be insane.
- A **course key** like `course-v1:SQA+BOT101+2026` is the universal ID every API and
  model uses — including our `CourseTierRequirement`.

## 5. The sanctioned extension surfaces (the map that prevents forking)

Open edX's whole modern philosophy: **extend without forking edx-platform.** The
surfaces, and when to reach for each:

| Surface | Use it to… | We use it for |
|---|---|---|
| Django plugin app | run backend code in-process | the entire membership/billing plugin |
| openedx-**events** (signals) | *react after* something happened | the enrollment safety net |
| openedx-**filters** | *intercept/veto before* something happens | (available; we chose signals — see C03 §4) |
| Waffle flags | turn behavior on/off at runtime | all 4 feature groups |
| Tutor plugin/patches | change deployment: settings, images, manifests | the tutor-indigo fork (theme URLs, slots, settings) |
| Comprehensive theme | restyle legacy pages | Meridian on gen-1 pages |
| Brand package + tokens | restyle all MFEs | Meridian on gen-2 pages |
| MFE plugin slots | inject UI into existing MFEs | dashboard widgets |
| Custom MFE | build a whole new user-facing app | sqa-payment |

Interview question for yourself: *"I want to award a badge when someone finishes a
course"* — which surface? (Events: subscribe to the completion event in a plugin app.
No fork, no filter, no theme.)

## You're ready when…

- Given any page URL you can name: generation (legacy/MFE), serving pod, and which repo
  styles it.
- You can explain to a newcomer why "just add a button to the dashboard" is a plugin-slot
  task, not a fork-the-dashboard task.
- You know why codejail exists and what OLX is.

**Next:** [C05 — containers, images, registries](05-containers-and-images.md) — the
packaging layer everything above ships through.
