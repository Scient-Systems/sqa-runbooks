# 07 — Writing Tutor plugins & patches

A **Tutor plugin** customizes *Tutor itself* — it injects Django settings, registers MFEs, adds
config keys, patches the Dockerfile. (Not to be confused with a Django plugin, runbook 06.) On this
project, Tutor plugins are single Python files in `~/.local/share/tutor-plugins/`. This is the
correct place for: injecting secrets/settings into the LMS, registering a custom MFE, and adding nav
links.

---

## 1. Anatomy of a Tutor plugin

A plugin is a `.py` file that imports `tutor.hooks` and adds items to filters. Dropping it in the
plugins dir makes it discoverable; you then `enable` it. Minimal example:

```python
# ~/.local/share/tutor-plugins/my_plugin.py
from tutor import hooks

# Inject Django settings into the LMS (dev environment):
hooks.Filters.ENV_PATCHES.add_item(("openedx-lms-development-settings", """
MY_FEATURE_FLAG = True
"""))

# Add a Tutor config key others can override with `tutor config save --set`:
hooks.Filters.CONFIG_DEFAULTS.add_item(("MY_PLUGIN_URL", "http://localhost:9000"))
```

Lifecycle:
```bash
tutor plugins list                  # see installed/enabled
tutor plugins enable my_plugin
tutor config save                   # 🚨 ALWAYS re-render env/ after enable/disable or edits
tutor dev stop && tutor dev start -d lms   # recreate so new settings load
tutor plugins disable my_plugin
```

## 2. The patches you'll actually use

A patch is a named hole in Tutor's generated files (full list: runbook 02 §5). The ones that matter
here:

| Patch | Use it to… |
|---|---|
| `openedx-lms-development-settings` | inject LMS settings **in `tutor dev` only** |
| `openedx-lms-production-settings` | inject LMS settings **in `local` + `k8s`** |
| `openedx-cms-development-settings` | inject CMS/Studio settings in dev (e.g. enable a FEATURES flag) |
| `openedx-common-settings` | settings for both LMS+CMS, all envs |
| `openedx-dockerfile-post-python-requirements` | add `RUN pip install …` to the openedx image |

🚨 **dev and production are different patches.** A `…-development-settings` patch is invisible in
production and vice-versa. Whenever you add a dev setting, ask: "does prod need the production-patch
twin?" (This asymmetry has caused "works in dev, broken in prod" more than once.)

## 3. Real example #1 — injecting dev secrets (`sqa_stripe_dev.py`)

```python
from tutor import hooks

STRIPE_DEV_SETTINGS = """
SQA_STRIPE_SECRET_KEY = "sk_test_REPLACE_ME"
SQA_STRIPE_PUBLISHABLE_KEY = "pk_test_REPLACE_ME"
SQA_STRIPE_WEBHOOK_SECRET = "whsec_REPLACE_ME"
SQA_STRIPE_PRICE_TO_LEVEL = { "price_xxx": "basic", "price_yyy": "premium" }
SQA_PAYMENT_MFE_URL = "http://apps.local.openedx.io:2003/sqa-payment"
"""
hooks.Filters.ENV_PATCHES.add_item(("openedx-lms-development-settings", STRIPE_DEV_SETTINGS))

# Also enable an author-facing Studio feature in dev:
hooks.Filters.ENV_PATCHES.add_item(("openedx-cms-development-settings", """
FEATURES["ENABLE_OTHER_COURSE_SETTINGS"] = True
"""))
```
Key points:
- `sk_…`/`whsec_…` live **only in this host-only file** — they never enter a Docker image.
- `pk_…` (publishable) is public and safe in the browser.
- 🚨 **Never enable a `*_dev` secrets plugin in production.** Production secrets go in a **k8s
  Secret** as env vars, not a Tutor patch (runbook 12) — partly because `plugin_settings()` can
  overwrite a Tutor patch with `os.environ.get(...)`, so a prod patch would be silently clobbered.

Verify the injection landed:
```bash
tutor dev exec lms python -c "import django; django.setup(); from django.conf import settings; print(bool(settings.SQA_STRIPE_SECRET_KEY))"   # True
```

## 4. Real example #2 — registering a custom MFE + nav link (`sqa_payment.py`)

```python
from tutor import hooks
from tutormfe.hooks import MFE_APPS, PLUGIN_SLOTS

@MFE_APPS.add()
def _add_sqa_payment_mfe(mfes):
    mfes["sqa-payment"] = {
        "repository": "https://github.com/Scient-Systems/frontend-app-sqa-payment.git",
        "port": 2003,           # outside the built-in MFE published range (1984,1993-2002,2025)
        "version": "main",
    }
    return mfes

# Config key for the public URL (override per-env with `tutor config save --set SQA_PAYMENT_MFE_URL=…`)
hooks.Filters.CONFIG_DEFAULTS.add_item(
    ("SQA_PAYMENT_MFE_URL", "http://apps.local.openedx.io:2003/sqa-payment"))

# Define a React component inside the learner-dashboard env.config block (Jinja {{ }} bakes the URL):
hooks.Filters.ENV_PATCHES.add_item((
    "mfe-env-config-runtime-definitions-learner-dashboard",
    """const MembershipLink = () => (
  <a className="nav-link" href="{{ SQA_PAYMENT_MFE_URL }}">Membership</a>
);""",
))

# Insert it into the learner-dashboard desktop top nav slot:
PLUGIN_SLOTS.add_item((
    "learner-dashboard",
    "org.openedx.frontend.layout.header_desktop_main_menu.v1",
    """
    { op: PLUGIN_OPERATIONS.Insert,
      widget: { id: 'sqa-membership-link', type: DIRECT_PLUGIN, RenderWidget: MembershipLink } },
    """,
))
```
This is the full pattern for "add a new MFE" (runbook 08) and "inject a component" (runbook 09).
🚨 Anything touching `env.config.jsx` / `PLUGIN_SLOTS` is **build-time** — it needs
`tutor images build mfe`, not just a restart.

## 5. Config defaults & overrides

```python
hooks.Filters.CONFIG_DEFAULTS.add_item(("KEY", "default"))   # a default everyone can override
hooks.Filters.CONFIG_UNIQUE.add_item(("KEY", "{{ 20|random_string }}"))  # generated once (secrets)
```
Override at any time: `tutor config save --set KEY=value`. Read: `tutor config printvalue KEY`.
🚨 A plugin default only applies if `config.yml` doesn't already pin the key — on a long-lived
machine, check `tutor config printvalue KEY` to see the *effective* value.

## 6. Where to put what (decision guide)

| You want to… | Mechanism | File |
|---|---|---|
| Set an LMS setting in dev | `ENV_PATCHES` → `openedx-lms-development-settings` | a tutor-plugin `.py` |
| Set an LMS setting in prod | `ENV_PATCHES` → `openedx-lms-production-settings`, OR k8s Secret env var | tutor-plugin / k8s |
| Store a production secret | **k8s Secret** (not a plugin file, not git, not an image) | runbook 12 |
| Register a custom MFE | `MFE_APPS` | tutor-plugin `.py` (runbook 08) |
| Inject a React widget into a slot | `tutor-indigo` 3-file wiring | runbook 09 |
| Add a pip dep to the image | `openedx-dockerfile-post-python-requirements` | tutor-plugin `.py` |

## 7. The fork case: `tutor-indigo` is a pip-installed plugin, not a loose file

The theme plugin isn't a loose `.py` — it's our fork installed as an editable pip package
(`pipx inject tutor … --editable` on the VPS). That lets the VPS pick up changes with just
`git pull` + `tutor config save`. Details in runbook 11.

---

## What's next

- Add or edit MFEs → **08-mfes-editing-and-adding-new.md**
- Inject a React component → **09-mfe-plugin-slots-react-components.md**
- Production secrets pattern → **12-production-deployment-k8s.md**
