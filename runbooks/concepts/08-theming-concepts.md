# C08 — Theming concepts: tokens, two delivery pipelines, dark mode, CDNs

> **Goal:** understand *why* the Meridian theme is delivered the way it is, so that when
> a style change doesn't show up you know which of the ~6 caches and 2 pipelines to
> interrogate — instead of re-deploying things at random.

## 1. The core problem

We restyle a product we don't own, spread across **two UI generations** (→ concepts/04):
legacy server-rendered pages and ~8 separate React MFEs. We are not allowed to fork them
all (maintenance suicide). So the theme must *inject* itself through sanctioned seams.

## 2. Concept: design tokens

A **design token** is a named design decision: `color-primary = #8CA12C`. Tokens live in
JSON in `brand-openedx`, get compiled to **CSS variables** (`--pgn-color-primary-base`),
and every Paragon component references the *variable*, never the hex.

Two payoffs:

1. **One decision, one place.** Change the token, every MFE follows.
2. **Variants are just different values for the same names.** Dark mode isn't a second
   stylesheet of overrides — it's the same variables resolving to night-palette values
   under `[data-paragon-theme-variant="dark"]`. Flip the attribute, the page re-skins.

> 🚨 Two hard-won lessons about variables and dark mode:
> **1. Hardcoded hex is a landmine.** Our brand partials compile TWICE — once normal,
> once nested inside the dark selector. A CSS *variable* resolves correctly both times; a
> hardcoded `#374151` follows you into dark mode and turns invisible-on-navy
> (→ war story #1).
> **2. CSS only styles what CSS reaches.** Native browser widgets — `<select>` popups,
> checkboxes, scrollbars, date pickers — are painted by the *browser*, not your
> stylesheet. They obey one CSS property: `color-scheme`. Forget to declare
> `color-scheme: dark` and the browser paints light-mode widgets under your dark theme:
> our white-on-white dropdowns. We had patched symptoms (`accent-color`,
> `scrollbar-color`, autofill) for weeks without knowing the one-line root cause.

## 3. The two delivery pipelines (the load-bearing design)

**Pipeline A — baked (build-time).** Tutor patches every MFE's Dockerfile to
`npm install` our brand package; the SCSS compiles *into each bundle*. Carries what CSS
variables can't: fonts, structural chrome (header/footer/login layouts).
Cost of change: **mfe image rebuild (~40 min) — and it's the layer-cache landmine**
(`--no-cache`, → concepts/05 §2).

**Pipeline B — runtime (the escape hatch we built on purpose).** The LMS hands every MFE
(via the `mfe_config` API, → concepts/02) a set of `PARAGON_THEME_URLS` — CSS files
served by the **jsDelivr CDN** straight from the brand repo's committed `dist/`.
Changing the theme = push CSS + point the URLs at the new commit + restart LMS.
**No image rebuild. ~2 minutes.**

This is the "move code → config" strategy from C01 §3 made real. We deliberately migrated
as much styling as possible from A to B after being burned by A's cost and cache traps.

**Pipeline legacy — the third world.** Gen-1 pages get SCSS from `tutor-indigo`,
Jinja-rendered by `tutor config save`, compiled into the **openedx** image. Dark mode
there is a `body.indigo-dark-theme` class + `$*-d` variables — same concepts, older
plumbing, openedx image rebuild to ship.

## 4. CDN + caching discipline (pipeline B's fine print)

Serving CSS from a CDN bought speed and rebuild-freedom, and cost us three rules written
in scar tissue:

1. **Pin by commit SHA, never branch** (`brand-openedx@27fc260…/dist/core.min.css`).
   A branch URL can be cached as yesterday's content forever; a SHA is immutable — cache
   it as hard as you like, it can never be *wrong*. Same philosophy as image digests
   (→ concepts/05 §3). (Also pragmatic: jsDelivr can't parse our branch name's slash.)
2. **The server must say `text/css`.** raw.githubusercontent.com serves `text/plain` +
   `nosniff`, and browsers **silently drop** the stylesheet — no error, MFEs just quietly
   fall back to baked-in CSS (→ war story #4). jsDelivr exists in our stack because of
   this single header.
3. **Know your cache chain**: brand `dist/` commit → jsDelivr (~5 min) → LMS config cache
   (Redis, 5 min) → browser cache (hard-refresh). Four places a "deployed" style can be
   time-traveling from.

## 5. What we chose and what else existed

| Option | Verdict |
|---|---|
| Fork every MFE and style directly | Rejected: ~8 repos to rebase forever |
| Everything in pipeline A (baked only) | Rejected after practice: 40-min rebuilds + cache landmine for a hex tweak |
| Everything in pipeline B (runtime only) | Impossible: fonts/structural chrome and gen-1 pages can't ride CSS-variable URLs |
| **A for skeleton, B for skin** (chosen) | Cheap iteration where it's frequent, baked where it must be. Cost: *you* must know which pipeline a given change rides — that's runbook 10's job |

## You're ready when…

- "Change the CTA color" vs "change the font" — you can route each to its pipeline and
  estimate cost (2 min vs 40 min) without looking.
- A style change isn't showing: you check the four caches **in order** before touching
  any pipeline.
- You can explain `color-scheme` to someone in two sentences and know which widgets it
  governs.

**Next:** [C09 — auth & secrets](09-auth-and-secrets.md).
