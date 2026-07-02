# C06 — The Tutor mental model: render, build, run

> **Goal:** internalize Tutor's three-stage pipeline so you always know which stage a
> change lives in — that single skill prevents most "my change didn't apply" mysteries.

## 1. What Tutor is

Installing Open edX by hand means wiring ~15 services, hundreds of settings, TLS, static
assets… Tutor is the official installer/manager that turns all of that into one config
file and one CLI. But don't think of it as "an installer". Think of it as a
**code generator with a runner attached**:

```
 config.yml  ──(1) tutor config save──▶  env/  ──(2) tutor images build──▶  images
 (source of truth)                  (rendered files)                       (frozen code)
                                                                              │
                                        (3) tutor k8s start / tutor dev ◀────┘
                                            (run the rendered manifests + images)
```

**Stage 1 — render.** `tutor config save` takes `config.yml` + every enabled plugin and
*generates* a full deployment tree under `$(tutor config printroot)/env/`: Dockerfiles,
Django settings, Caddyfile, Kubernetes manifests. Pure text generation, nothing deployed.

**Stage 2 — build.** `tutor images build openedx|mfe` runs Docker on the *rendered*
Dockerfiles (→ concepts/05).

**Stage 3 — run.** `tutor dev start` (docker-compose, dev machine) or `tutor k8s start`
(apply manifests to the cluster, VPS).

**The iron law: never edit `env/` by hand.** It is build output — the next
`tutor config save` silently overwrites your edit. If you find yourself wanting to edit a
file in `env/`, what you actually want is a *plugin patch* (§2). Corollary: after editing
any plugin or config value, nothing whatsoever has changed until you re-render.

## 2. Tutor plugins and patches

A **Tutor plugin** is Python that hooks into stage 1 — adding config defaults, adding
template files, or injecting text at named **patch points** inside the generated files.

A patch point is a labeled hole. Tutor's templates contain markers like
`{{ patch("openedx-lms-production-settings") }}`; every plugin's contribution to that
name gets concatenated into that spot at render time. Our `tutor-indigo` fork uses
patches for nearly everything:

- Django settings for the LMS (Stripe keys wiring, MFE config additions)
- `PARAGON_THEME_URLS` (the runtime theme CSS pointers, → concepts/08)
- MFE Dockerfile lines (install our brand package into every MFE build)
- `env.config.jsx` fragments (MFE slot components, → runbook 09)
- Kubernetes manifest overrides (the `k8s-override` patch — we use it for
  single-node-safe rollout strategy, → war story #2 / `K8S_ROLLOUT_MEMORY_FIX.md`)

So the fork is not "a theme" — it's **our deployment's control panel**: one repo that
tells Tutor how our whole installation differs from stock.

## 3. Same input, three runners

| Mode | Runner | We use it for |
|---|---|---|
| `tutor dev` | docker-compose, dev conveniences (auto-reload, mounted source) | daily development in WSL2 |
| `tutor local` | docker-compose, production-ish | (not used) |
| `tutor k8s` | Kubernetes manifests | production on the VPS |

Same rendered `env/`, different runner. That's Tutor's quiet superpower: dev and prod
share one source of truth, so "works in dev" usually survives the trip.

**The iteration ladder** (a thinking tool, not just a fact): every change should be
verified at the **cheapest tier that can catch it being wrong**:

| Tier | Cost | Catches | Cannot catch |
|---|---|---|---|
| 1. `pytest` (SQLite, no LMS) | seconds | models, the eligibility rule, view logic with LMS APIs stubbed — ~80% of changes | plugin loading, real signals, real auth |
| 2. `tutor dev` (WSL2) | minutes | entry-point wiring, real signals, real OAuth, template rendering | image builds, k8s config flow, public webhooks |
| 3. k8s deploy (VPS) | tens of minutes | everything — production truth | nothing, but at maximum cost per attempt |

Escalate a tier only when the current one *genuinely cannot falsify* your change.
Debugging at tier 3 what tier 1 could have caught is the most common way to lose an
afternoon. (→ concepts/14 §5 shows this ladder used on a real feature.)

> 🚨 **The one place our dev/prod symmetry is broken (this has burned us):** the VPS has
> our *fork* of tutor-indigo injected editably (pipx), so `git pull` + `tutor config save`
> picks up fork changes. The WSL2 dev machine has the *stock pip release* of tutor-indigo
> — fork changes (theme, patches, plugin.py) **cannot be validated locally**; they're
> only real on the VPS. Details → runbook 10 §1.

## 4. Everyday inspection verbs

```bash
tutor config printroot            # where is everything?
tutor config printvalue LMS_HOST  # what value actually rendered?
grep -r "my-change" $(tutor config printroot)/env/   # did my change render at all?
```

That grep is hop #2 of the universal debugging chain (→ concepts/11): *repo → env/ →
image → pod → browser*.

## 5. Releases & upgrades — the future tax

Open edX ships **named releases** (~every 6 months), and **each Tutor major version is
married to one of them** — our Tutor 21.x runs release "Teak"-era code as image tag
`21.0.2`. Upgrading is therefore never "bump a number": it's new Tutor + new images +
**database migrations** (backup first! → concepts/13) + re-validating everything we've
customized. Budget the honest checklist now, not during the upgrade:

- our plugin against the new Django/edx-platform APIs (tier 1 tests, then tier 2),
- the tutor-indigo fork **merged with its upstream's new release branch**,
- brand-openedx against the new Paragon,
- every pinned thing (the `BRAND_DIST_REF` mechanics survive; the pins need review).

The trade-off we accepted by customizing at all: every seam we injected into (plugin,
fork, brand, slots) is a seam we must re-verify per upgrade. This is *vastly* cheaper
than a fork of edx-platform — sanctioned seams are stable-ish by design — but it is not
free, and skipping releases compounds it. Rule: never upgrade more than one named
release at a time, and treat the upgrade as a project (spec + tasks), not a command.

## 6. Trade-off corner

- **Tutor vs manual install:** manual gives total control and no framework to learn;
  cost: you own every wiring decision and every upgrade forever. Nobody does this
  anymore; Tutor is the official path.
- **Render-then-run vs edit-live:** you can't hot-patch production settings by editing a
  file on the server — everything must flow through config/plugin → render → apply. Feels
  slower the first week; it's the reason our production is *reproducible* (the entire
  deployment can be rebuilt from git repos + config.yml + secrets).
- **Fork tutor-indigo vs write our own plugin from scratch:** forking gave us a working
  theme + slot wiring to modify; cost: we inherit upstream's structure and must merge
  upstream changes occasionally.

## You're ready when…

- Someone reports "I changed config.yml and nothing happened" and you reply "did you
  `tutor config save`?" before they finish the sentence.
- You can say which stage (render/build/run) each of these lives in: a Django setting, a
  plugin Python file, a theme SCSS file, a waffle switch. (Trick: the waffle switch is
  *none* — it's data in MySQL, changeable live in Django Admin. That's the point of
  waffle, → concepts/03.)
- You treat `env/` as radioactive build output.

**Next:** [C07 — Kubernetes from zero](07-kubernetes-from-zero.md).
