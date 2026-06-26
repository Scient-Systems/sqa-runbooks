# 11 — The Tutor Indigo plugin

`tutor-indigo` is the Tutor plugin that does our theming and MFE-component injection. It's worth its
own runbook because (a) it's a *forked* plugin installed differently from our loose `tutor-plugins/`
files, and (b) it quietly renames your Docker image tags. This explains what it is, how it's
installed locally vs on the VPS, what knobs it exposes, and how changes reach production.

For the design content it delivers, see runbook 10. For component injection, runbook 09.

---

## 1. What Indigo is

Upstream `overhangio/tutor-indigo` is a popular community theme for Open edX. It is a normal Tutor
plugin (`tutormfe.hooks` + `tutor.hooks`) that ships:
- a **comprehensive legacy theme** (Mako templates + SCSS) baked into the openedx image;
- a set of **MFE components** (footer, theme toggle, themed logo) injected via `env.config.jsx`
  patches + `PLUGIN_SLOTS`;
- Tutor **config defaults** (`INDIGO_PRIMARY_COLOR`, `INDIGO_WELCOME_MESSAGE`, etc.);
- an init task that selects the theme automatically.

We run a **fork** (`Scient-Systems/tutor-indigo`, branch `release`) carrying the Meridian design and
our custom widgets (SqaDashboardHero, SqaMembershipInstrument, SqaDashboardEmptyState, SqaTokenCard,
MembershipLink wiring).

## 2. 🚨 It renames your image tags

When indigo is enabled it appends `-indigo` to the image names:
- `DOCKER_IMAGE_OPENEDX` → `…/openedx-indigo`
- `MFE_DOCKER_IMAGE` → `…/openedx-mfe-indigo`

So our production tags are `ghcr.io/scient-systems/openedx-indigo` and
`ghcr.io/scient-systems/openedx-mfe:21.0.0-indigo`. Always read the *effective* value before
build/push:
```bash
tutor config printvalue DOCKER_IMAGE_OPENEDX
tutor config printvalue MFE_DOCKER_IMAGE
```

## 3. How it's installed (local vs VPS — they differ!)

| | Local (WSL) | VPS (production) |
|---|---|---|
| Source | pip-installed `tutor-indigo 21.1.3` (NOT the fork) | the fork at `/home/edx/tutor-indigo` |
| Install method | `pip install tutor-indigo` (came with `tutor[full]`) | `pipx inject tutor /home/edx/tutor-indigo --editable` |
| Pick up changes | n/a — local can't see the fork | `git pull` + `tutor config save` (editable → no reinstall) |

🚨 The consequence (the single most important fact about indigo on this project): **a local
`tutor config save` does NOT reflect or validate fork changes** — it runs the pip 21.1.3 copy. Theme
and slot-widget changes can only be truly validated on the VPS. Before pushing a fork change, mentally
verify the wiring (runbook 09 §7) instead of relying on a local build.

To set the VPS up the same way on a fresh machine:
```bash
git clone https://github.com/Scient-Systems/tutor-indigo.git /home/edx/tutor-indigo
pipx inject tutor /home/edx/tutor-indigo --editable
tutor plugins enable indigo
tutor config save
```

## 4. The knobs (config defaults in `plugin.py`)

```python
"WELCOME_MESSAGE": "Where curious minds become builders",   # the index-hero <h1>
"PRIMARY_COLOR": "#101A33",                                  # Meridian ink (deep navy)
"ENABLE_DARK_TOGGLE": True,
"FOOTER_NAV_LINKS": [ {"title": "About Us", "url": "/about"}, ... ],
```
Override per-environment:
```bash
tutor config save --set INDIGO_WELCOME_MESSAGE="..."
tutor config save --set INDIGO_PRIMARY_COLOR="#101A33"
tutor config save --set INDIGO_FOOTER_NAV_LINKS='[]'   # remove all footer links
```
🚨 A `config.yml` override beats the plugin default — check `tutor config printvalue INDIGO_…` to see
what's actually in effect (defaults silently lose).

## 5. What lives in the fork (file map)

```
tutorindigo/
├── plugin.py                 ← config defaults, image rename, MFE_APPS deps, PLUGIN_SLOTS, brand npm patches
├── components/*.jsx          ← injected MFE widgets (Imports, IndigoFooter, Sqa*…). Runbook 09.
├── patches/
│   ├── mfe-env-config-buildtime-imports      ← ES imports (Imports.jsx)
│   ├── mfe-env-config-runtime-definitions     ← component DEFINITIONS ({{ patch("X.jsx") }})
│   └── mfe-env-config-runtime-final           ← footer dedupe logic
└── templates/indigo/
    ├── lms/templates/*.html  ← Mako overrides (index, footer, course, static pages)
    ├── lms/static/sass/...   ← the legacy theme SCSS (runbook 10 §2)
    └── tasks/init.sh         ← auto-selects the theme on `tutor … do init`
```

🚨 New files anywhere under `tutorindigo/` must be `git add`-ed — the VPS builds from the committed
tree; an untracked partial once broke an openedx image build.

## 6. Deploying an indigo change

Which pipeline depends on *what* you changed (this is the runbook-10 §4 table, restated):

- **Legacy theme (templates/sass)** → openedx image: `git pull` → `tutor config save` →
  `tutor images build openedx` → push → `rollout restart deployment/lms deployment/cms`.
- **MFE component / `plugin.py` slot / brand npm line** → mfe image: `git pull` → `tutor config save`
  → `tutor images build mfe` → push → `rollout restart deployment/mfe`.
- **Config default only** (e.g. WELCOME_MESSAGE) → it's rendered into templates, so it still needs the
  openedx image rebuild to change legacy pages; for MFE config it needs `tutor config save` + mfe
  rebuild.

Because indigo is an **editable** install on the VPS, the only "install" step is `git pull` — no
`pip install`/`pipx` re-run needed.

---

## What's next

- Full image build/push/rollout mechanics → **12-production-deployment-k8s.md**
- The CSS/design content indigo delivers → **10-theming-lms-and-mfes.md**
