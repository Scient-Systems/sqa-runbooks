# 10 — Theming the LMS & MFEs

How the visual design ("Meridian") is delivered. There are **four UI surfaces**, each styled by a
different repo through a different pipeline. Getting the surface→repo→pipeline mapping right is the
whole job; the design values are just data you swap at known touchpoints.

This runbook is the *plumbing*. Current design values live in `sqa-homepage/DESIGN_SYSTEM.md`
(canonical) and `tutor-indigo/MERIDIAN_THEME.md`. Component injection (React widgets, not CSS) is
runbook 09.

---

## 1. The four surfaces

| Surface | URL | Styled by | Deploy cost |
|---|---|---|---|
| Marketing homepage | stemquestacademy.com | `~/sqa-homepage` (Next.js, standalone) | its own deploy (runbook 13) |
| **Legacy LMS pages** (certificates, policy pages, instructor dashboard, the problem iframe inside a unit) | lms.stemquestacademy.com | comprehensive theme SCSS in `~/tutor-indigo` | **openedx image rebuild** (CI) |
| **MFEs** (catalog = homepage + course list + about, learning, learner-dashboard, profile, account, discussions, authn) | apps.lms.…/* | brand package `~/brand-openedx` (2 layers — §3) + widgets from `~/tutor-indigo` | runtime CSS: theme pin + config save; anything else: mfe image (CI) |
| Payment MFE | apps.lms.…/sqa-payment | `~/frontend-app-sqa-payment` + brand runtime layer | mfe image rebuild (CI) |

On Verawood the LMS homepage, `/courses` and course about pages **redirect to the Catalog MFE**,
and `/dashboard` to the learner-dashboard MFE, so their old theme templates no longer reach anyone.

Studio/CMS and the authoring MFE are deliberately left unthemed (tool-dense surfaces).

The repos (all under github.com/Scient-Systems; the user commits/pushes, no AI attribution):
| Clone | Branch | Notes |
|---|---|---|
| `~/tutor-indigo` | `verawood/indigo` | fork of overhangio/tutor-indigo, ported onto upstream v22. On the servers it is installed **editable** into the `tutor22` pipx venv from `~/tutor-indigo-verawood`, a checkout shared by every site on the box: `git checkout <commit>` there + `tutor22 config save` picks up changes. |
| `~/brand-openedx` | `verawood/indigo` | fork of edly-io/brand-openedx. `dist/` (SQA) and `dist-scient/` (the demo's palette) are **committed** (pushing publishes the runtime CSS). Repo must stay **public** (browsers fetch its CSS by URL). |
| `~/frontend-app-sqa-payment` | `main` | the build clones it at the commit `versions.yml` pins. |
| `~/sqa-homepage` | `main` | standalone Next.js. |

The old Ulmo branches (`release`, `ulmo/indigo`) are frozen; nothing deploys from them.

🚨 Locally, `pip install tutor[full]` brings the **upstream** `tutor-indigo`, not the fork. A local
`tutor config save` reflects upstream until you install your `verawood/indigo` clone editable
(`pip install -e ~/tutor-indigo`). Otherwise validate theme changes on staging — see runbook 09
gotcha #4.

### Per-site looks without a rebuild

Several sites share the same images; what differs is set in each site's Tutor config and read at
runtime (keys from `tutor-indigo` `plugin.py`, set with `tutor22 config save --set …`):

| Key | What it does | Who sets it |
|---|---|---|
| `INDIGO_BRAND_PROFILE` | swaps in a second palette and logo set (`dist-<profile>/`, `assets/<profile>/`) | demo: `scient` |
| `INDIGO_BRAND_NAME`, `INDIGO_CATALOG_HEADLINE`, `INDIGO_WELCOME_MESSAGE` | names and headline copy | demo |
| `INDIGO_FOOTER_NAV_LINKS` | footer links; `[]` = none | k-12 and demo: `[]`; default: Terms + Privacy |
| `INDIGO_SHOW_MEMBERSHIP` | header Membership link, dashboard plan panel, plan redirect | k-12 and demo: `false` |
| `INDIGO_SHOW_AI_TOKENS` | the AI token card on the profile | demo: `false` |
| `PLATFORM_NAME` | site name everywhere (course tour, emails, titles) | all: their own |

## 2. Legacy LMS theme (`~/tutor-indigo`)

A "comprehensive theme": Mako templates + SCSS that override stock LMS pages. They live in
`tutorindigo/templates/indigo/lms/...`, are **Jinja-rendered by `tutor config save`** into
`env/build/openedx/themes/indigo/`, then baked into the **openedx image** by CI
(live in pod at `/openedx/themes/indigo/`).

- 🚨 Beware Jinja *inside* SCSS/HTML: `{{ INDIGO_PRIMARY_COLOR }}`, `{% if INDIGO_ENABLE_DARK_TOGGLE %}`.
- SCSS entry points: `lms/static/sass/partials/lms/theme/_variables.scss` (palette + Google Fonts
  `@import`; dark theme is variable-driven via `$*-d` vars) and `_extras.scss` (global rules; imports
  every per-page partial at the bottom: header, footer, dashboard, course-about, discover, home,
  certificates…).
- Key selectors: `header.global-header`, `.wrapper-footer`/`footer.tutor-container`, the index hero
  `#main .home.style-logout header … .title`, course cards `.courses-container … .course`,
  `body.indigo-dark-theme` prefix for all dark overrides.
- `components/*.jsx` (IndigoFooter, ToggleThemeButton, AddDarkTheme, ThemedLogo…) are injected into
  MFEs via `tutorindigo/patches/mfe-env-config-*` + `PLUGIN_SLOTS` — same mechanism as runbook 09.
- Config defaults in `plugin.py`: `PRIMARY_COLOR`, `WELCOME_MESSAGE` (the index-hero `<h1>`),
  `ENABLE_DARK_TOGGLE`, `FOOTER_NAV_LINKS`. 🚨 Defaults only apply if `config.yml` doesn't override —
  check `tutor config printvalue INDIGO_WELCOME_MESSAGE`.
- SCSS compile-check without an LMS: concat a tiny stub (a `media-breakpoint-up` mixin + `$black`,
  `$static-path`) + `_variables.scss` (strip the googleapis/edx-bootstrap imports, substitute Jinja
  tokens) + the partials, then `npx sass`.

## 3. MFE theming — TWO delivery layers (the key insight)

### Layer A — build-time `@edx/brand` (SCSS compiled into each MFE bundle)
`tutor-indigo/plugin.py` emits `mfe-dockerfile-post-npm-install-<app>` patches:
`RUN npm install '@edx/brand@github:Scient-Systems/brand-openedx#verawood/indigo'` for the
styled MFEs (catalog, learning, learner-dashboard, profile, account, discussions, authn,
sqa-payment). This carries **everything that is NOT a CSS variable**: font `@import`s,
header/footer/login chrome, `.btn-brand` styling, pill buttons. Changing it = a new mfe image (CI).

Brand repo structure (`paragon/` dir): `core.scss` → `_variables`/`_fonts`/`_overrides`;
`_overrides.scss` imports every page partial (`_header _footer _login _profile _dashboard …`) then
`_dark.scss`, which **re-imports all the same partials inside `[data-paragon-theme-variant="dark"]`**
(each compiles twice; CSS vars resolve to dark token values the second time — so hardcoded hex in a
partial applies to BOTH variants; use that deliberately for chrome).
🚨 Partials use custom media queries like `@media (--pgn-size-breakpoint-min-width-md)` — these only
work through this repo's own build pipeline. In production MFEs the **dist ships raw** and browsers
**drop those blocks** → such responsive rules are silently dead. Use plain px queries (768px = paragon
md) in any new partial.

### Layer B — runtime CSS (`PARAGON_THEME_URLS`) — no image rebuild
`plugin.py` sets `MFE_CONFIG["PARAGON_THEME_URLS"]` (patch `mfe-lms-common-settings`) with a `core`
key (paragon core + our compiled `dist/core.min.css` as `brandOverride`) plus light/dark variants —
**all served from jsDelivr pinned to a commit SHA** (`BRAND_DIST_REF` in plugin.py):
`https://cdn.jsdelivr.net/gh/Scient-Systems/brand-openedx@<SHA>/dist/{core,light,dark}.min.css`.

🚨 **Never use `raw.githubusercontent.com` for these** — it serves `text/plain` +
`X-Content-Type-Options: nosniff`, so browsers refuse to apply the stylesheet (silently; MFEs fall
back to the baked-in CSS). jsDelivr serves proper `text/css`. SHA pinning is mandatory anyway — the
branch name `verawood/indigo` has a slash that jsDelivr's `@ref` can't parse.

A site with `INDIGO_BRAND_PROFILE` set (the demo) loads `dist-<profile>/` instead of `dist/` from
the same commit.

Tokens: `paragon/tokens/src/themes/{light,dark}/**.json` → `npm ci && make build` regenerates
`paragon/build/` + `dist/`. 🚨 No `@font-face`/`@import` survives into the variant CSS — fonts ship
only via Layer A (or per-MFE CSS).

### sqa-payment specifics
Its `src/index.scss` does NOT import `@edx/brand`; page styling is **inline styles inside the page
components** (PricingPage/ManagePage/CheckoutPage). Inline styles beat stylesheet rules unless
`!important`. It does honor `PARAGON_THEME_URLS` (runtime theming).

## 4. Deploy pipelines (pick by what changed)

The normal path for every kind of theme change is the pipeline in runbook **17**: push to
`verawood/indigo`, change the pins in `openedx-deploy` `versions.yml`, merge, and CI builds both
images and deploys staging (which also moves the shared theme checkout). For brand CSS that means
three steps: the brand commit, `BRAND_DIST_REF` in `tutor-indigo`, and both pins (17 §3).

The one shortcut, for **runtime brand CSS only** (Layer B) on a single site, without a build:

```bash
# brand pushed, BRAND_DIST_REF bumped and pushed in tutor-indigo → on the box:
export PATH="$HOME/.local/bin:$PATH" TUTOR_ROOT=~/.local/share/<root> TUTOR_PLUGINS_ROOT=~/.local/share/<root>-plugins
git -C ~/tutor-indigo-verawood fetch -q https://github.com/Scient-Systems/tutor-indigo.git verawood/indigo
git -C ~/tutor-indigo-verawood checkout <tutor-indigo-commit>
tutor22 config save && tutor22 k8s start     # updates ConfigMaps (a restart alone isn't enough on k8s)
```

🚨 The checkout is **shared by every site on the box**: the neighbour picks up that theme commit
the next time *its* config is saved. Afterwards record the new pins in `versions.yml`, or the next
pipeline deploy moves the checkout back.

🚨 The `npm install @edx/brand@…#verawood/indigo` line is a **cached Docker layer** — the branch ref
never re-resolves on its own, so an mfe build can ship stale baked-in brand CSS. This is exactly why
structural CSS lives in the runtime `brandOverride` (Layer B), which is pinned by SHA.

Verify chain when "nothing changed": runbook **15**, "Universal 'nothing changed' recipe".

## 5. Redesign touchpoints (swap the design = edit exactly these)
1. `~/sqa-homepage`: `app/globals.css` (tokens), `app/layout.tsx` (fonts), `app/page.tsx` + components.
2. `~/tutor-indigo`: `_variables.scss` (palette + fonts + `$*-d` dark vars), `_extras.scss` head,
   `extra/_header.scss`, `extra/_footer.scss`, `xblock/_xblock.scss` (problems inside units),
   `plugin.py` (`PRIMARY_COLOR`, `WELCOME_MESSAGE`), `components/AddDarkTheme.jsx` (iframe dark
   colors), `components/Sqa*.jsx` (catalog, dashboard, header, course-complete widgets).
3. `~/brand-openedx`: `paragon/tokens/src/themes/{light,dark}/*.json`, `_fonts.scss`, `_overrides.scss`,
   `_header/_footer/_login/_profile/_dashboard/_catalog/_course-exit/_phone.scss`, `palettes/`
   (the demo's second palette) → then `make build` and **commit `dist/` and `dist-scient/` too**.
4. `~/frontend-app-sqa-payment`: `src/index.scss` + the inline-styled pages.

## 6. The logo constraint
`logo.png` = colorful bubble letters on navy. Looks great on dark chrome, poor on white. Both footers
display it — 🚨 **never `filter: invert()` the footer logo**. LMS serves it at
`/theming/asset/images/logo.png`.

The MFE headers are a navy band in both themes; the header logo is **48px** tall on desktop and
36px on phones (`_header.scss`, `_phone.scss`). The learning MFE's own CSS sets
`#root header .logo img { height: 100% }`, which is why `_header.scss` pins the course header's
logo explicitly. Change the size in one place and check it on the dashboard and a course page, in
both themes.

---

## What's next

- The indigo plugin specifically (the fork, pipx inject, config) → **11-tutor-indigo-plugin.md**
- Deploy mechanics (image build/push/rollout) → **12-production-deployment-k8s.md**
