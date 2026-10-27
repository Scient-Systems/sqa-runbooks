# 00 — START HERE: The whole platform in one map

> **Read this first.** It explains what we are building, every moving part, where each
> part runs, and which runbook to open for which job. If you are brand new to Open edX,
> read this top to bottom once — it will make every other runbook make sense.

These runbooks are distilled from real build sessions (May–October 2026). Every command here
has actually been run on this project.

> 🚨 **Since 2026-10-27 every site runs Open edX Verawood on Tutor 22.** On the servers the CLI is
> `tutor22`, never plain `tutor`. The theme and brand repos live on `verawood/indigo`, and
> `openedx-deploy` `main` is the Verawood line. Anything you read elsewhere about Ulmo, Tutor 21,
> the `release` / `ulmo/indigo` branches or `openedx:21.0.x` image tags is history. Where something bit us, the gotcha is written down
so you never rediscover it. The deep "why" docs still live at the repo root
(`LOCAL_DEV.md`, `PROD_DEPLOYMENT.md`, `MFE_COMPONENT_GUIDE.md`, `LMS_THEMING_GUIDE.md`,
`SQA_PLUGIN_CONTEXT.md`) — the runbooks are the **operational** layer on top of them.

> 🧠 **New here, or lost in the jargon?** There is now a **concept-first layer** under
> [`concepts/`](concepts/README.md): 14 short docs that explain the platform from the big
> picture down — Django & plugins, Open edX anatomy, Docker, Tutor, Kubernetes, theming,
> auth/secrets, payments, data & backups, the debugging method, war stories of real
> incidents with the alternatives and trade-offs we weighed, and the anatomy of how a
> real feature was built end-to-end. **If a runbook ever feels like information
> overload, the missing background lives there.** Runbooks = what to type; concepts = why
> it works and what to do when it doesn't.

---

## 1. What is Open edX (the 30-second version)

Open edX is the open-source learning platform that powers edx.org. For our purposes it is
**three big pieces** plus a swarm of supporting services:

- **LMS** ("Learning Management System") — what *students* use. Serves courses, enrollment,
  the dashboard, login, the API. Python/Django monolith called `edx-platform`.
- **CMS / Studio** — what *course authors* use to build courses. Same `edx-platform`
  codebase, different process.
- **MFEs** ("Micro-Frontends") — modern React single-page apps that have replaced many old
  server-rendered LMS pages (the learner dashboard, profile, account, the learning/courseware
  view, login, discussions). Each MFE is its own repo and its own little web app.

Supporting services: MySQL, MongoDB, Redis, Elasticsearch/Meilisearch, Caddy (web server/TLS),
Celery workers, and optional plugins (forum, notes, discovery, credentials…).

**You never install all that by hand.** We use **Tutor** — the official Docker-based
installer/manager for Open edX — which packages every service into containers and gives you
one CLI to run them. Tutor is the single most important tool in this project. Read runbook
**02** to understand it.

---

## 2. What WE are building (Stem Quest Academy)

A membership-gated Open edX site. The custom value we add on top of stock Open edX:

1. **Membership tiers** (Free → Basic → Premium → Enterprise). A Django plugin gates course
   enrollment by tier. (`sqa_django_app`)
2. **Stripe payments** so students can buy a tier — a custom React MFE + Stripe webhooks
   in the plugin. (`frontend-app-sqa-payment` + `sqa_django_app`)
3. **A custom theme** ("Meridian") across the legacy LMS pages and the MFEs.
   (`tutor-indigo` fork + `brand-openedx` fork)
4. **A marketing homepage** outside Open edX entirely. (`sqa-homepage`, Next.js)
5. **An AI-token broker** so minors can use AI (Gemini) inside a course without ever holding
   a real API key — a control plane in the plugin plus a standalone gateway service.
   (`sqa_django_app` + `sqa-proxytoken-service`)
6. **Courses**, e.g. the "Silly Bot" teen AI-chatbot course. (`silly_bot_course_project`)

---

## 3. The architecture map — every part and where it lives

```
┌─────────────────────────── YOUR DEV MACHINE (WSL2 Ubuntu) ───────────────────────────┐
│                                                                                       │
│  Tutor (the manager CLI)  ──drives──▶  Docker containers (lms, cms, mfe, mysql, …)    │
│      └ config:  ~/.local/share/tutor/config.yml                                       │
│      └ plugins: ~/.local/share/tutor-plugins/*.py   (our custom Tutor plugins)        │
│                                                                                       │
│  Repos you edit (all under ~/ , inside the Linux filesystem — NOT /mnt/c):            │
│   ~/sqa_django_app           Django plugin: membership, billing, AI-token broker      │
│   ~/frontend-app-sqa-payment Custom React MFE: pricing/checkout/manage pages          │
│   ~/tutor-indigo             Theme + MFE widgets (a Tutor plugin, forked)  verawood/indigo │
│   ~/brand-openedx            MFE design tokens + SCSS (the @edx/brand package)  verawood/indigo │
│   ~/openedx-deploy           versions.yml, CI builds, deploy scripts, per-env config   │
│   ~/sqa-homepage             Marketing homepage (Next.js, totally standalone)          │
│   ~/sqa-proxytoken-service   AI gateway (FastAPI) — deployed to Vercel, not Tutor      │
│   ~/silly_bot_course_project Course content (OLX) + companion Streamlit app           │
└───────────────────────────────────────────────────────────────────────────────────────┘
                                          │  git push → pin in openedx-deploy → CI build
                                          ▼
┌──────────── TWO VPSes, Kubernetes (k3s) via Tutor 22 (`tutor22`), images on GHCR ──────┐
│  VPS 1  Production  lms.stemquestacademy.com / apps.lms.…        ns openedx           │
│         Demo        training-demo.stemquestacademy.com / apps.…  ns openedx-demo      │
│  VPS 2  Staging     staging.stemquestacademy.com / apps.staging.… ns openedx          │
│         k-12        k-12.stemquestacademy.com / apps.k-12.…      ns openedx-verawood  │
│  k8s Secrets per namespace: sqa-stripe (Stripe keys), sqa-gateway, ghcr-pull           │
└───────────────────────────────────────────────────────────────────────────────────────┘
        │                                   │
        ▼ (separate deploys, NOT Tutor)     ▼
   Vercel: sqa-proxytoken-service      Vercel/Netlify: sqa-homepage
   + Neon Postgres                     Streamlit Cloud: silly_bot companion app
```

**The single most important architectural fact:** `sqa_django_app` is **not a microservice**.
It is a pip-installable Django app that loads *inside the LMS Python process* via an
`entry_points={'lms.djangoapp': ...}` line in `setup.py`. That is why it can call LMS APIs
and listen to LMS signals natively. See runbook **06**.

---

## 4. The four ways code reaches production (know which one you're in)

Different changes deploy through completely different pipelines. Picking the wrong one is the
#1 source of "I changed it but nothing happened":

| You changed… | Pipeline | Runbook |
|---|---|---|
| Python in `sqa_django_app` | push `main` → pin PR opens itself in `openedx-deploy` → merge → CI builds → staging auto → production by button | 06, 17 |
| A custom MFE's React code (`frontend-app-sqa-payment`) | push `main` → **you** change its pin in `versions.yml` → same as above | 08, 17 |
| An MFE **slot widget / `env.config.jsx`** (`tutor-indigo` components) | push `verawood/indigo` → **you** change its pin → same as above | 09, 17 |
| Legacy LMS theme (`tutor-indigo` SCSS/templates) | push `verawood/indigo` → **you** change its pin → same as above | 10, 17 |
| Brand CSS (`brand-openedx`) | push `verawood/indigo` → bump `BRAND_DIST_REF` in `tutor-indigo` → pin **both** | 10, 17 §3 |
| A Tutor plugin file (`tutor-plugins/*.py`) | in `openedx-deploy`; build-time ones need a build, runtime ones a `tutor22 config save` on the box | 07 |
| A per-site setting (footer links, membership, name…) | `tutor22 config save --set …` on that site's root, then `k8s start` | 11, 12 |
| The gateway service | `git push` → Vercel auto-deploy | 13 |
| A course | Studio import/export — no deploy at all | 05 |

Every build gives **both** images one new tag (the hash of `versions.yml`). k-12 and the demo
are not in the pipeline: they are rolled forward by hand (17 §4b).

---

## 4b. The other half of "where do I make changes": the extension surfaces

The table above is about *deploying*. The deeper question is *which mechanism* to use at all. Open
edX is built to be extended **without forking `edx-platform`**, through a handful of sanctioned
surfaces — Django plugin apps, **openedx-events** (react to things), **openedx-filters** (intercept/
block things), Tutor plugins/patches, comprehensive themes, and MFE plugin slots. Understanding that
whole map is what lets you build things these runbooks don't spell out. The full decision table lives
in **runbook 06 §2** — read it once early; it's the conceptual spine.

---

## 5. Glossary (terms you'll hit everywhere)

- **Tutor** — the Docker-based Open edX manager CLI. Three run modes: `dev`, `local`, `k8s`.
  We run Tutor 22 (Verawood); on the servers it is installed as `tutor22`.
- **Verawood / Ulmo** — Open edX release names. Verawood is what runs; Ulmo is the retired line.
- **`versions.yml`** — the file in `openedx-deploy` that pins every input of an image build.
  Its hash is the image tag.
- **MFE** — Micro-Frontend; a standalone React app (learner-dashboard, profile, our sqa-payment…).
- **OLX** — Open Learning XML; the file format courses are authored/exported in.
- **Waffle switch** — a database-backed feature flag. Our plugin gates whole feature groups
  behind them. They default OFF and are toggled in Django Admin. See runbook **04**.
- **Plugin (two unrelated meanings!)**:
  - a **Django plugin** = a pip package the LMS loads (our `sqa_django_app`). Runbook 06.
  - a **Tutor plugin** = a Python file that customizes Tutor itself (`tutor-plugins/*.py`). Runbook 07.
- **Patch** — a named insertion point in Tutor's generated files. A Tutor plugin "patches"
  e.g. `openedx-lms-production-settings` to inject Django settings. Runbook 07.
- **Plugin slot** — a named hole in an MFE's React tree where you can inject a component
  (e.g. the dashboard sidebar). Runbook 09.
- **`env.config.jsx`** — the per-deployment MFE config file Tutor *assembles from patches at
  build time*. It is compiled into the bundle — NOT a runtime file. Runbook 09.
- **GHCR** — GitHub Container Registry; where our custom Docker images live.
- **Comprehensive theme** — Open edX's mechanism for overriding legacy (Mako/SCSS) LMS pages;
  what `tutor-indigo` ships. Runbook 10.

---

## 6. Recommended reading order

- **Day zero (don't know Docker/k8s/Django/Open edX yet?):** `concepts/` 01 → 02 first,
  then the rest of `concepts/` as each topic becomes relevant. They're written to be read
  *before* the runbook that uses them.
- **First day:** 00 (this) → 01 (set up your machine) → 02 (understand Tutor) → 03 (run it).
- **First task:** whichever of 04–14 matches the job (see §4 table).
- **When stuck:** 15 (troubleshooting) — a symptom→fix index — plus `concepts/11`
  (the debugging method) and `concepts/12` (war stories) for the *how to think* half.

> Conventions in these runbooks: shell blocks are copy-paste-ready. `<ANGLE_BRACKETS>` mean
> "replace this". 🚨 marks a footgun that has actually cost us hours. ✅ marks a verification
> step — never skip those; "it built" is not "it works".
