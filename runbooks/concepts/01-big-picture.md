# C01 — The big picture: what we run, in layers

> **Goal of this doc:** after reading it you can draw the whole platform on a whiteboard
> from memory, and for any file in any of our repos you can say *where its code ends up
> running*.

## 1. Start from what a student sees

A teenager visits **stemquestacademy.com**, reads about the academy, clicks "join",
creates an account, buys a Premium membership with a card, and starts an AI-chatbot
course. Their browser touched **four completely different applications** without them
noticing:

1. The **marketing homepage** — a standalone Next.js site. Knows nothing about courses.
2. The **LMS** (Learning Management System) — the Open edX core. Accounts, courses,
   enrollment, the old-style server-rendered pages.
3. Several **MFEs** (micro-frontends) — modern React apps for login, profile, account,
   the course player, and our custom payment pages.
4. The **AI gateway** — a small separate service that lets the course's chatbot call a
   real AI model without the student ever holding an API key.

Behind those, invisible: databases, a payments provider (Stripe), a web server that does
HTTPS (Caddy), and background workers.

**First mental model: the platform is not one program.** It is a small fleet of programs
that trust each other. Most confusion on this project comes from working on program A
while the bug is in program B — or in the glue between them.

## 2. The five layers (top-down)

Think of the platform as five layers. Every task you'll ever do lives in exactly one:

```
┌──────────────────────────────────────────────────────────────────┐
│ 5. PRODUCT     what users experience: courses, memberships,     │
│                payments, themes, the chatbot                     │
├──────────────────────────────────────────────────────────────────┤
│ 4. APPLICATIONS  LMS, CMS/Studio, each MFE, homepage, gateway   │
│                  (each is a codebase someone can edit)           │
├──────────────────────────────────────────────────────────────────┤
│ 3. PACKAGING     Docker images — frozen, shippable copies of    │
│                  the applications (→ concepts/05)                │
├──────────────────────────────────────────────────────────────────┤
│ 2. ORCHESTRATION Kubernetes — keeps the right containers        │
│                  running, wired, and fed with config (→ 07)     │
├──────────────────────────────────────────────────────────────────┤
│ 1. MACHINE       one VPS: 8 GB RAM, one disk, one IP address    │
└──────────────────────────────────────────────────────────────────┘
```

And one tool that cuts across layers 2–4: **Tutor** (→ concepts/06). Tutor takes a config
file and *generates* the Dockerfiles, settings, and Kubernetes manifests, so we never
hand-write them. When you wonder "who created this file?", the answer is usually Tutor.

## 3. The repo map — what you edit, and where it runs

| Repo (all under `~/`) | What it is | Where the code ends up |
|---|---|---|
| `sqa_django_app` | **Our membership plugin**: tiers, enrollment gating, Stripe webhooks, AI-token control plane | *Inside* the LMS Python process — baked into the `openedx` Docker image |
| `frontend-app-sqa-payment` | Our custom React MFE: pricing / checkout / manage pages | Static JS bundle, baked into the `mfe` Docker image |
| `tutor-indigo` (fork) | Theme for legacy LMS pages + Tutor plugin that wires everything (settings, MFE slots, theme URLs) | Templates → rendered by Tutor → some baked into images, some become runtime config |
| `brand-openedx` (fork) | Design tokens + SCSS for all MFEs ("Meridian" look) | Two ways: baked into MFE bundles **and** served live from a CDN (→ concepts/08) |
| `sqa-homepage` | Marketing site (Next.js) | Deployed on its own (not Tutor, not k8s) |
| `sqa-proxytoken-service` | AI gateway (FastAPI) | Vercel + Neon Postgres — completely outside the VPS |
| `silly_bot_course_project` | Course content (OLX) + companion app | Imported into Studio; companion on Streamlit Cloud |

**Second mental model — the golden rule of "where things live":**

> Everything in production is one of three kinds:
> **code** (baked into an image at build time),
> **config** (injected into running containers at start time),
> or **data** (rows in a database, survives everything).

Why you should care: this rule decides how you *change* things and how *fast*.
Changing **code** = rebuild an image = 10–40 minutes. Changing **config** = re-render and
restart = ~2 minutes. Changing **data** = a database write = instant. A huge amount of our
engineering effort (like the runtime theme layer, → concepts/08) is about moving things
from the "code" column into the "config" column so changes get cheaper.

## 4. Production, concretely

One VPS runs Kubernetes ("k8s"), managed through `tutor k8s`. On it, in the `openedx`
namespace, live roughly these pods (a pod ≈ one running program, → concepts/07):

- `lms` — the big one: Django serving lms.stemquestacademy.com. **Our plugin runs inside
  this process.**
- `cms` — Studio (course authoring), same codebase as LMS, different process.
- `lms-worker`, `cms-worker` — Celery workers for background jobs (emails, grading…).
- `mfe` — a tiny nginx serving the *compiled* React bundles of every MFE at
  apps.lms.stemquestacademy.com.
- `caddy` — the front door: owns ports 80/443, does HTTPS certificates, routes each
  domain to the right pod.
- `mysql`, `mongodb`, `redis`, `meilisearch`, `minio` — the data layer: relational data,
  course content, cache/queues, search, file storage.
- `codejail…` — sandboxed Python execution for course problems.

Separate from the VPS entirely: Stripe (payments), GHCR (stores our Docker images),
jsDelivr (serves our theme CSS), Vercel (gateway + homepage), Google Fonts.

## 5. What we chose and what else existed

| Decision | We chose | Alternatives | Why / trade-off |
|---|---|---|---|
| How to customize Open edX | A **plugin** inside the LMS | Fork edx-platform; separate microservice | Fork = merge hell on every upgrade. Microservice = can't hear LMS signals natively, needs its own auth/DB sync. Plugin = native power; trade-off: our bugs run *inside* the LMS, and deploys ride the big image (→ concepts/03) |
| How to install/run Open edX | **Tutor** | Manual "native" install; the old devstack | Manual = weeks of pain, no upgrade path. Tutor is the official, supported way. Trade-off: you must learn Tutor's render-then-run model (→ concepts/06) |
| Production runtime | **k8s on one VPS** (`tutor k8s`) | `tutor local` (docker-compose) on the VPS; managed k8s (EKS/GKE) | docker-compose would honestly be simpler on one machine; k8s gives us declarative config, secrets, and a growth path. Managed k8s costs 3–5× more. Trade-off we accepted: k8s complexity + an 8 GB node that can't run two LMS pods at once (→ war story #2) |
| Where images live | **GHCR** (private) | Docker Hub; public images | Private keeps our plugin code out of public pull; trade-off: every puller (including the cluster itself!) needs credentials that can *expire* (→ concepts/09) |
| Homepage | Standalone Next.js | Theme the LMS index page | Marketing iterates weekly; decoupling means homepage deploys never touch the LMS |

## You're ready when…

- You can name the four applications a student's browser talks to, and the one they never
  talk to directly (the gateway is called by the course's tooling, not styled pages).
- Someone says "just change the pricing page text" and you know: repo
  `frontend-app-sqa-payment` → that's **code** → mfe image rebuild → ~30 min, not 30 sec.
- You can explain why our plugin crashing could take the whole LMS down with it (same
  process!), and why we accepted that.

**Next:** [C02 — a request's journey](02-request-journey.md), the doc that makes
debugging possible.
