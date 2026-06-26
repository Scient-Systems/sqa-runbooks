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
| **Legacy LMS pages** (index, dashboard, course-about, wiki, certificates) | lms.stemquestacademy.com | comprehensive theme SCSS in `~/tutor-indigo` | **openedx image rebuild** |
| **MFEs** (learning, learner-dashboard, profile, account, discussions, authn) | apps.…/* | brand package `~/brand-openedx` (2 layers — §3) | mfe image rebuild OR just LMS restart (runtime layer) |
| Payment MFE | apps.…/sqa-payment | `~/frontend-app-sqa-payment` + brand runtime layer | mfe image rebuild |

Studio/CMS and the authoring MFE are deliberately left unthemed (tool-dense surfaces).

The repos (all under github.com/Scient-Systems; the user commits/pushes, no AI attribution):
| Clone | Branch | Notes |
|---|---|---|
| `~/tutor-indigo` | `release` | fork of overhangio/tutor-indigo. Deployed on VPS via editable pipx inject → `git pull` + `tutor config save` picks up changes. |
| `~/brand-openedx` | `ulmo/indigo` | fork of edly-io/brand-openedx. `dist/` is **committed** (pushing publishes the runtime CSS). Repo must stay **public** (browsers fetch its CSS by URL). |
| `~/frontend-app-sqa-payment` | `main` | mfe image pulls from GitHub `#main` — push before building. |
| `~/sqa-homepage` | `main` | standalone Next.js. |

🚨 Local WSL has **pip-installed tutor-indigo 21.1.3**, NOT the fork. Local `tutor config save`
reflects 21.1.3; only the VPS reflects the fork. **Theme changes can't be validated locally** — see
runbook 09 gotcha #4.

## 2. Legacy LMS theme (`~/tutor-indigo`)

A "comprehensive theme": Mako templates + SCSS that override stock LMS pages. They live in
`tutorindigo/templates/indigo/lms/...`, are **Jinja-rendered by `tutor config save`** into
`env/build/openedx/themes/indigo/`, then baked into the **openedx image** at
`tutor images build openedx` (live in pod at `/openedx/themes/indigo/`).

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
`RUN npm install '@edx/brand@github:Scient-Systems/brand-openedx#ulmo/indigo'` for learning,
learner-dashboard, profile, account, discussions, authn, sqa-payment. This carries **everything that
is NOT a CSS variable**: font `@import`s, header/footer/login chrome, `.btn-brand` styling, pill
buttons. Changing it = `tutor images build mfe` + push + restart.

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
branch name `ulmo/indigo` has a slash that jsDelivr's `@ref` can't parse.

Tokens: `paragon/tokens/src/themes/{light,dark}/**.json` → `npm ci && make build` regenerates
`paragon/build/` + `dist/`. 🚨 No `@font-face`/`@import` survives into the variant CSS — fonts ship
only via Layer A (or per-MFE CSS).

### sqa-payment specifics
Its `src/index.scss` does NOT import `@edx/brand`; page styling is **inline styles inside the page
components** (PricingPage/ManagePage/CheckoutPage). Inline styles beat stylesheet rules unless
`!important`. It does honor `PARAGON_THEME_URLS` (runtime theming).

## 4. Deploy pipelines (pick by what changed)

```bash
# A. Legacy LMS theme changed (tutor-indigo templates/sass):
cd /home/edx/tutor-indigo && git pull && tutor config save
tutor images build openedx && tutor images push openedx
kubectl -n openedx rollout restart deployment/lms deployment/cms

# B. Brand SCSS layer / sqa-payment changed → mfe image:
cd /home/edx/tutor-indigo && git pull && tutor config save   # if plugin.py changed too
tutor images build mfe && tutor images push mfe
kubectl -n openedx rollout restart deployment/mfe deployment/lms

# C. Brand TOKENS only (color.json etc.) — NO image rebuild:
# local: make build → commit+push ulmo/indigo → bump BRAND_DIST_REF SHA in plugin.py → push tutor-indigo
cd /home/edx/tutor-indigo && git pull && tutor config save
tutor k8s start                                       # updates ConfigMaps (restart alone isn't enough on k8s)
kubectl -n openedx rollout restart deployment/lms     # serves new MFE config; CSS cached ≤5min
```
🚨 The `npm install @edx/brand@…#ulmo/indigo` line is a **cached Docker layer** — the branch ref
never re-resolves on rebuild, so a 40-min `tutor images build mfe` can ship stale brand CSS. Bust it
with `--no-cache` (or bump the install line) when Layer A must change. This is exactly why structural
CSS moved to the runtime `brandOverride` (Layer B) where possible.

Verify chain when "nothing changed": grep the change in (1) VPS clone → (2)
`$(tutor config printroot)/env/...` → (3) `docker run --rm <image> grep …` → (4)
`kubectl -n openedx exec deploy/lms -- grep …`. First failure = the broken hop.

## 5. Redesign touchpoints (swap the design = edit exactly these)
1. `~/sqa-homepage`: `app/globals.css` (tokens), `app/layout.tsx` (fonts), `app/page.tsx` + components.
2. `~/tutor-indigo`: `_variables.scss` (palette + fonts + `$*-d` dark vars), `_extras.scss` head,
   `extra/_header.scss`, `extra/_footer.scss`, `home/_home.scss`, `courseware/_discover.scss`,
   `plugin.py` (`PRIMARY_COLOR`, `WELCOME_MESSAGE`), `components/AddDarkTheme.jsx` (iframe dark colors).
3. `~/brand-openedx`: `paragon/tokens/src/themes/{light,dark}/*.json`, `_fonts.scss`, `_overrides.scss`,
   `_header/_footer/_login/_profile/_dashboard.scss` → then `make build` and **commit `dist/` too**.
4. `~/frontend-app-sqa-payment`: `src/index.scss` + the inline-styled pages.

## 6. The logo constraint
`logo.png` = colorful bubble letters on navy. Looks great on dark chrome, poor on white. Both footers
display it — 🚨 **never `filter: invert()` the footer logo**. LMS serves it at
`/theming/asset/images/logo.png`.

---

## What's next

- The indigo plugin specifically (the fork, pipx inject, config) → **11-tutor-indigo-plugin.md**
- Deploy mechanics (image build/push/rollout) → **12-production-deployment-k8s.md**
