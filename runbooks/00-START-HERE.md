# 00 — START HERE: The whole platform in one map

> **Read this first.** It explains what we are building, every moving part, where each
> part runs, and which runbook to open for which job. If you are brand new to Open edX,
> read this top to bottom once — it will make every other runbook make sense.

These runbooks are distilled from real build sessions (May–June 2026). Every command here
has actually been run on this project. Where something bit us, the gotcha is written down
so you never rediscover it. The deep "why" docs still live at the repo root
(`LOCAL_DEV.md`, `PROD_DEPLOYMENT.md`, `MFE_COMPONENT_GUIDE.md`, `LMS_THEMING_GUIDE.md`,
`SQA_PLUGIN_CONTEXT.md`) — the runbooks are the **operational** layer on top of them.

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
│   ~/tutor-indigo             Theme + MFE component injection (a Tutor plugin, forked)  │
│   ~/brand-openedx            MFE design tokens + SCSS (the @edx/brand package)         │
│   ~/sqa-homepage             Marketing homepage (Next.js, totally standalone)          │
│   ~/sqa-proxytoken-service   AI gateway (FastAPI) — deployed to Vercel, not Tutor      │
│   ~/silly_bot_course_project Course content (OLX) + companion Streamlit app           │
└───────────────────────────────────────────────────────────────────────────────────────┘
                                          │  git push
                                          ▼
┌──────────────────────────── PRODUCTION (VPS, Kubernetes via Tutor k8s) ───────────────┐
│  lms / cms / mfe pods (custom Docker images on GHCR)                                   │
│  k8s Secrets:  sqa-stripe (Stripe keys), sqa-gateway (broker URL+secret)              │
│  LMS host:  lms.stemquestacademy.com     MFE host: apps.lms.stemquestacademy.com      │
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
| Python in `sqa_django_app` | Rebuild **openedx** image → push → rollout LMS | 06, 12 |
| A custom MFE's React code (`frontend-app-sqa-payment`) | Rebuild **mfe** image → push → rollout MFE | 08, 12 |
| An MFE **slot widget / `env.config.jsx`** (`tutor-indigo` components) | Rebuild **mfe** image → push → rollout MFE | 09, 12 |
| Legacy LMS theme (`tutor-indigo` SCSS/templates) | Rebuild **openedx** image → push → rollout LMS | 10, 12 |
| Brand **design tokens only** (`brand-openedx` JSON/CSS) | Push brand → `tutor config save` → rollout LMS (no rebuild) | 10 |
| A Tutor plugin file (`tutor-plugins/*.py`) | `tutor config save` (+ maybe image rebuild) | 07 |
| The gateway service | `git push` → Vercel auto-deploy | 13 |
| A course | Studio import/export — no deploy at all | 05 |

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

- **First day:** 00 (this) → 01 (set up your machine) → 02 (understand Tutor) → 03 (run it).
- **First task:** whichever of 04–14 matches the job (see §4 table).
- **When stuck:** 15 (troubleshooting) — it's a symptom→fix index of every gotcha we hit.

> Conventions in these runbooks: shell blocks are copy-paste-ready. `<ANGLE_BRACKETS>` mean
> "replace this". 🚨 marks a footgun that has actually cost us hours. ✅ marks a verification
> step — never skip those; "it built" is not "it works".
