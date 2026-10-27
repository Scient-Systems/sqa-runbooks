# 11 — The Tutor Indigo plugin

`tutor-indigo` is the Tutor plugin that does our theming and MFE-component injection. It's worth its
own runbook because (a) it's a *forked* plugin installed differently from our loose plugin files,
and (b) one checkout of it is shared by every site on a server. This explains what it is, how it's
installed locally vs on the servers, what knobs it exposes, and how changes reach production.

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

We run a **fork** (`Scient-Systems/tutor-indigo`, branch **`verawood/indigo`**, ported onto
upstream v22) carrying the Meridian design and our custom widgets: the catalog homepage and course
about page (SqaCatalog, SqaCourseAboutEnroll), the header menu (SqaHeaderNav), the dashboard
(SqaDashboardHero, SqaMembershipInstrument, SqaDashboardEmptyState, SqaPlanRedirect), the
course-complete page (SqaCourseComplete), the Exit course link (SqaExitCourse), and SqaTokenCard.
It also has per-site settings (brand profile, footer links, membership, AI tokens) so several sites
share the same images. The old `release` branch is the retired Ulmo line.

## 2. Image names

Upstream indigo appends `-indigo` to the *default* image names. Our servers never see that: every
site's config sets both names explicitly to the CI-built images,
`ghcr.io/scient-systems/openedx:<tag>` and `ghcr.io/scient-systems/openedx-mfe:<tag>`. Read the
effective value with:
```bash
tutor22 config printvalue DOCKER_IMAGE_OPENEDX
tutor22 config printvalue MFE_DOCKER_IMAGE
```

## 3. How it's installed (local vs servers — they differ!)

| | Local (WSL) | Servers (both VPSes) |
|---|---|---|
| Source | upstream `tutor-indigo` (came with `tutor[full]`), unless you install your clone | the fork at `~/tutor-indigo-verawood` |
| Install method | `pip install tutor[full]==22.0.2`; optionally `pip install -e ~/tutor-indigo` | installed **editable** into the `tutor22` pipx venv |
| Pick up changes | `tutor config save` (only sees the fork if installed editable) | `git checkout <commit>` in the checkout + `tutor22 config save` |

🚨 **One checkout per box, shared by every site on it** (production + demo on VPS 1, staging +
k-12 on VPS 2). Moving it changes what the *next* `config save` of any site on that box renders.
The deploy script moves it to the commit pinned in `versions.yml`; if you move it by hand, the
neighbour picks it up on its next save. Fetch by URL, because the clone's `origin` may be a local
path: `git -C ~/tutor-indigo-verawood fetch https://github.com/Scient-Systems/tutor-indigo.git verawood/indigo`.

🚨 Without the fork installed locally, **a local `tutor config save` does NOT reflect fork changes**.
Install your clone editable, or validate on staging. Before pushing a fork change, verify the wiring
(runbook 09 §7) instead of relying on a local build.

To set up a fresh server the same way:
```bash
pipx install --suffix 22 "tutor==22.0.2"                 # installs the CLI as tutor22
pipx inject tutor22 tutor-mfe==22.1.0 tutor-minio==22.0.0 tutor-contrib-codejail==22.0.1   # versions: openedx-deploy versions.yml
git clone -b verawood/indigo https://github.com/Scient-Systems/tutor-indigo.git ~/tutor-indigo-verawood
pipx inject tutor22 --editable ~/tutor-indigo-verawood
TUTOR_ROOT=… TUTOR_PLUGINS_ROOT=… tutor22 plugins enable indigo
```

## 4. The knobs (config defaults in `plugin.py`)

```python
"WELCOME_MESSAGE": "Where curious minds become builders",   # the index-hero <h1>
"PRIMARY_COLOR": "#101A33",                                  # Meridian ink (deep navy)
"ENABLE_DARK_TOGGLE": True,
"FOOTER_NAV_LINKS": [ {"title": "Terms of Service", "url": "/tos"},
                      {"title": "Privacy Policy", "url": "/privacy"} ],
"BRAND_PROFILE": "",          # "scient" on the demo: second palette + logos
"SHOW_MEMBERSHIP": True,      # False on k-12 and the demo
"SHOW_AI_TOKENS": True,       # False on the demo
```
Override per-environment (on a server, with that site's `TUTOR_ROOT`/`TUTOR_PLUGINS_ROOT` set):
```bash
tutor22 config save --set INDIGO_WELCOME_MESSAGE="..."
tutor22 config save --set INDIGO_FOOTER_NAV_LINKS='[]'     # remove all footer links
tutor22 config save --set INDIGO_SHOW_MEMBERSHIP=false     # hide membership
tutor22 k8s start                                          # apply
```
🚨 A `config.yml` override beats the plugin default — check `tutor22 config printvalue INDIGO_…` to
see what's actually in effect (defaults silently lose). The per-site values each site uses are in
runbook **10** §1.

The policy pages behind the default footer links are `templates/indigo/lms/templates/static_templates/tos.html`
and `privacy.html`.

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

🚨 New files anywhere under `tutorindigo/` must be `git add`-ed — CI builds from the committed
tree; an untracked partial once broke an openedx image build.

`tutor-indigo` has its own tests on `verawood/indigo` (`.github/workflows/test.yml`). Its
`notify-deploy` job would open the pin PR in `openedx-deploy`, but it skips today because the repo
has no `DEPLOY_DISPATCH_TOKEN` secret — so you change its pin by hand (runbook 17).

## 6. Deploying an indigo change

Every kind of change goes through the pipeline (runbook **17**): push to `verawood/indigo`, set the
`tutor-indigo` ref in `openedx-deploy` `versions.yml`, merge to `main`. CI builds both images and
the deploy moves the servers' shared checkout to that commit.

What each kind of change actually needs, for when you are judging how risky it is:

- **Legacy theme (templates/sass)** → inside the openedx image.
- **MFE component / `plugin.py` slot / brand npm line** → inside the mfe image.
- **`BRAND_DIST_REF` or a runtime setting in `plugin.py`** → only a `config save` + `k8s start`, no
  image (runbook 10 §4 has the shortcut and its caveat).
- **A per-site setting** (`INDIGO_*` in one site's config) → `tutor22 config save --set …` on that
  site only; no commit, no build.

---

## What's next

- Full image build/push/rollout mechanics → **12-production-deployment-k8s.md**
- The CSS/design content indigo delivers → **10-theming-lms-and-mfes.md**
