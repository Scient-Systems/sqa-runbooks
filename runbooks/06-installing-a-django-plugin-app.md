# 06 — Installing a Django plugin app into the LMS

This is the heart of our custom backend. `sqa_django_app` is a **pip-installable Django app that
runs inside the LMS Python process** — not a separate service. This runbook explains how that
works, the anatomy of the app, the dev install loop, migrations, and how to scaffold a brand-new
plugin if you ever need one.

---

## 1. The one line that makes it a plugin

Open edX discovers Django plugins via setuptools entry points. In `setup.py`:

```python
entry_points={
    'lms.djangoapp': ['sqa_django_app = sqa_django_app.apps:SqaDjangoAppConfig'],
    'cms.djangoapp': ['sqa_django_app = sqa_django_app.apps:SqaDjangoAppConfig'],
},
```

Without that, no amount of correct Django code makes the LMS load the app. With it, simply
`pip install`-ing the package into the LMS makes Open edX add it to `INSTALLED_APPS`, mount its
URLs, load its settings, and connect its signals — automatically.

> Scaffolded from **`cookiecutter-django-app`** (the edX cookiecutter that produces installable
> Django apps), NOT `cookiecutter-django-ida` (which produces standalone services). That choice is
> *why* we can call LMS APIs and listen to LMS signals natively.

## 2. The big picture: the official ways to extend Open edX

Before the specifics, internalize this. Open edX is designed to be extended **without forking
`edx-platform`**. There are a small number of sanctioned extension surfaces; almost any modification
is "which surface do I reach for?" Knowing the whole map is what lets you build things these runbooks
don't spell out.

| You want to… | Use | Sync? | Runbook |
|---|---|---|---|
| Add models, REST APIs, admin, server pages, business logic | **Django plugin app** (entry point) | — | this one |
| **React to** something that happened (user registered, course published, cert issued) | **openedx-events** (a.k.a. "hooks/signals") — you *observe*, you can't change the outcome | async-ish, fire-and-forget | this one §4 |
| **Intercept / change / block** something mid-flow (prevent an enrollment, rewrite a grade) | **openedx-filters** (a "pipeline") — you *transform* the data or `raise` to stop it | synchronous | this one §4–5 |
| Inject Django settings, infra, pip deps, env vars | **Tutor plugin + patches** | — | 07 |
| Change how **legacy** (Mako/SCSS) LMS pages look | **comprehensive theme** | — | 10 |
| Add/replace a **React component** in a modern MFE page | **MFE plugin slot** | — | 09 |
| Add a new **course component** (a graded widget authors drop into a unit) | **XBlock** (separate pip package) | — | not used here |

> The single most-confused pair is **events vs filters**. Events = "tell me *after* it happened, I'll
> do something on the side." Filters = "let me sit *in the path* and approve/modify/abort it." Our
> enrollment gate is a **filter** (it must block); our auto-assign-on-registration is an **event** (it
> just reacts). Reach for a filter only when you need to change the outcome.

## 3. The wiring dict: `apps.py`

`pip install` makes Open edX *discover* the app (via the entry point, §1). The `plugin_app` dict on
`SqaDjangoAppConfig` tells it *how to wire it in*. This is the **real** structure (lightly trimmed):

```python
from edx_django_utils.plugins.constants import PluginSettings, PluginURLs, PluginSignals
try:
    from openedx.core.djangoapps.plugins.constants import ProjectType, SettingsType
except ImportError:                       # outside an LMS (tests / standalone) — define the strings
    class ProjectType:  LMS = 'lms.djangoapp';  CMS = 'cms.djangoapp'
    class SettingsType: PRODUCTION = 'production';  COMMON = 'common'

class SqaDjangoAppConfig(AppConfig):
    name = label = 'sqa_django_app'

    plugin_app = {
        PluginURLs.CONFIG: {                       # mount urls.py under a prefix + namespace…
            ProjectType.LMS: {                     # …only in the LMS process
                PluginURLs.NAMESPACE: 'sqa_django_app',
                PluginURLs.REGEX: r'^sqa/',
                PluginURLs.RELATIVE_PATH: 'urls',
            },
        },
        PluginSettings.CONFIG: {                   # call settings modules during settings load
            ProjectType.LMS: {
                SettingsType.PRODUCTION: {PluginSettings.RELATIVE_PATH: 'settings.production'},
                SettingsType.COMMON:     {PluginSettings.RELATIVE_PATH: 'settings.common'},
            },
        },
        PluginSignals.CONFIG: {                    # connect openedx-events receivers
            ProjectType.LMS: {
                PluginSignals.RELATIVE_PATH: 'signals',
                PluginSignals.RECEIVERS: [{
                    PluginSignals.RECEIVER_FUNC_NAME: 'on_student_registered',
                    PluginSignals.SIGNAL_PATH: 'openedx_events.learning.signals.STUDENT_REGISTRATION_COMPLETED',
                }],
            },
            ProjectType.CMS: {                     # a DIFFERENT handler runs in the Studio process
                PluginSignals.RELATIVE_PATH: 'signals',
                PluginSignals.RECEIVERS: [{
                    PluginSignals.RECEIVER_FUNC_NAME: 'on_course_catalog_info_changed',
                    PluginSignals.SIGNAL_PATH: 'openedx_events.content_authoring.signals.COURSE_CATALOG_INFO_CHANGED',
                }],
            },
        },
    }
```

What to actually take from this:

- 🚨 **`PluginURLs`/`PluginSettings`/`PluginSignals` come from `edx_django_utils`** — a normal pip
  dependency that's *always* importable. They are **not** the things behind the `try/except`. Only
  **`ProjectType`/`SettingsType`** are LMS-internal, so only those get the inline-fallback (which lets
  the package import in `pytest`/standalone with no LMS present). Keep that fallback; don't widen it.
- **`ProjectType.LMS == 'lms.djangoapp'`** — the dict keys are those exact strings. Same string as the
  entry-point group, deliberately.
- **LMS and CMS are the same codebase, different processes.** That's why one `plugin_app` registers a
  *different* signal receiver per process: `on_student_registered` only matters in the LMS;
  `on_course_catalog_info_changed` only fires in Studio (on course publish). URLs/settings here are
  LMS-only.
- **Settings register two modules:** `settings.common` (runs in every env) and `settings.production`
  (prod only). 🚨 But in `tutor dev`, Open edX's settings chain (`lms.envs.tutor.development`) does
  **not** call `plugin_settings()` reliably — so anything that *must* exist in dev is also set up in
  `AppConfig.ready()`. See §5 and runbook 04 §5.

## 4. The control design (how the pieces actually gate behavior)

1. **Waffle switches** (`waffle.py`) — entire URL groups and handler bodies no-op when their switch is
   OFF. All default OFF; created automatically by `waffle_init()` on startup; toggled in Django Admin.
   The 6 switches: `api_membership`, `api_enrollment`, `gate_enrollment`, `auto_assign`, `api_billing`,
   `api_proxy`. (Runbook 04 §4.)
2. **`check_enrollment_eligibility(user, course_key)`** (`utils.py`) — the single source of truth for
   the rank-comparison rule. Imported by **`api.py`** (the enroll view) and **`pipeline.py`** (the
   filter). 🚨 Never re-implement the rule anywhere else.
3. **Enrollment gating is two layers, BOTH calling that rule:**
   - **Layer 1 — `MembershipEnrollView`** (`api.py`): our own enroll endpoint returns **403** before
     enrolling. Proactive, but only covers enrollments that go *through our API*.
   - **Layer 2 — `MembershipEnrollmentCheck`** (`pipeline.py`): an **openedx-filters** step on
     `org.openedx.learning.course.enrollment.started.v1`. It runs for **every** enrollment source
     (LMS dashboard, admin, bulk import, our API) and `raise`s `CourseEnrollmentStarted.PreventEnrollment`
     **before the enrollment row is written** — so there's nothing to undo. Gated by `gate_enrollment`.
   > 🚨 **This replaced an older design** (a `COURSE_ENROLLMENT_CREATED` *event* handler that unenrolled
   > after the fact). If you read "unenroll signal / safety net" in older docs or `CLAUDE.md`, that's
   > stale — the current safety net is the **filter** (block-before-write), not an event. Reason: a
   > filter can *prevent*; an event can only react after the row exists. (Migration notes:
   > `LOCAL_DEV.md` → "openedx-filters migration".)
4. **What the two signal handlers do** (both openedx-events, both waffle-gated):
   - `on_student_registered` (LMS, `STUDENT_REGISTRATION_COMPLETED`) → assigns the **`free`** level to
     a brand-new user. Gated by `auto_assign`.
   - `on_course_catalog_info_changed` (CMS, `COURSE_CATALOG_INFO_CHANGED`, fires on every course
     publish) → reads the course's `other_course_settings['membership_required_level']` (the Studio
     Advanced Setting) and upserts a `CourseMembershipRequirement`. Gated by `gate_enrollment`. This is
     the bridge that lets a *course author* set a tier requirement without touching Django admin
     (runbook 05 §7).

## 5. File map of the package

```
sqa_django_app/
├── apps.py            ← AppConfig + plugin_app wiring + ready()
├── urls.py            ← routes, mounted under ^sqa/ (paths are RELATIVE to that prefix)
├── api.py             ← DRF views (membership, enroll, billing, proxy)
├── views.py           ← server-rendered plugin pages (e.g. /sqa/tokens/)
├── models.py          ← MembershipLevel, UserMembership, CourseMembershipRequirement,
│                        MembershipPayment, StripeEvent, ProxyToken, CourseIntegrationGrant
├── utils.py           ← check_enrollment_eligibility (the rule)
├── signals.py         ← openedx-events handlers: on_student_registered (LMS),
│                        on_course_catalog_info_changed (CMS publish → requirement sync)
├── waffle.py          ← the switches + waffle_init()
├── pipeline.py        ← openedx-filters enrollment pipeline step
├── stripe_service.py  ← Stripe SDK calls (billing)
├── proxy_service.py / gateway_client.py  ← AI-token broker
├── health.py          ← /sqa/health/
├── admin.py           ← Django admin registrations
├── settings/common.py ← plugin_settings()  + production.py
├── migrations/        ← ship inside the package; applied with manage.py migrate
└── management/commands/  ← sqa_django_app_init, sqa_proxy_reconcile
```

## 6. 🚨 Gotchas learned the hard way (each one cost real time)

- **URL prefix is added by the loader.** Routes in `urls.py` are relative to `^sqa/`. Write
  `path("health/", …)` → served at `/sqa/health/`. Writing `path("sqa/health/", …)` gives
  `/sqa/sqa/health/`.
- **Use the namespace in `reverse()`/`redirect()`/`{% url %}`.** Bare `redirect('sqa_tokens_page')`
  raises `NoReverseMatch`; use `redirect('sqa_django_app:sqa_tokens_page')`.
- **Waffle switches are read at import time** in `urls.py` → routes only (de)register at process
  start. Tests must use an unconditional `tests/api_urls.py` (switches are off when the test process
  starts). Restart the LMS after toggling.
- **`AppConfig.ready()` runs before DB tables exist.** `waffle_init()` hits the waffle table; on a
  fresh DB that table may not exist yet → `OperationalError`. The function wraps DB calls in
  try/except. Don't remove that guard.
- **`signals.py` must use lazy model imports.** It's imported at app-init, before the app registry
  is ready. Any top-level `from .models import …` crashes with `AppRegistryNotReady`. Put all model
  imports **inside** the handler bodies.
- **openedx-events handlers receive dataclasses, not Django models.** `STUDENT_REGISTRATION_COMPLETED`
  hands `on_student_registered` a `UserData` dataclass, not a Django `User`. Resolve the real user
  first: `user = User.objects.get(id=user.id)` — else `TypeError: Field 'id' expected a number but got
  UserData(...)`. (Same pattern for any event payload: pull the id off the dataclass, re-fetch the ORM
  object.)
- **URL route ordering:** specific paths before wildcards. `path('api/user/assign/')` must come
  before `path('api/user/<str:username>/')`, or `assign` is matched as a username (405 on POST).
- **openedx-filters `fail_silently` defaults to `True`** → a `PreventEnrollment` raised in a pipeline
  step is silently swallowed and enrollment proceeds. Always set `fail_silently: False` when the
  intent is to block.
- **Pipeline steps are plain classes** — do NOT subclass `OpenEdxPublicFilter` (that's for filter
  *definitions*). A plain class with a `run_filter` classmethod.
- **`PluginSettings.CONFIG` is unreliable in `tutor dev`** — its settings chain
  (`lms.envs.tutor.development`) doesn't call `plugin_settings()`, so `OPEN_EDX_FILTERS_CONFIG` stays
  unset. Fix: register filter config inside `AppConfig.ready()` via direct `settings` mutation, which
  fires regardless of the settings chain.
- **Django-template plugin pages** must extend `main_django.html` (NOT `main.html`, which is Mako),
  and use `{% block body %}` (not `content`). `main_django.html` doesn't load full Bootstrap — write
  self-contained `<style>`, set explicit `color` (the dark theme overrides inherited text colors).

## 7. The dev install + migrate loop

Covered in runbook 03 §4–5. Short version:
```bash
tutor dev exec lms pip install -e /openedx/sqa_django_app
tutor dev stop lms && tutor dev start -d lms        # discover entry_point
tutor dev exec lms pip install -e /openedx/sqa_django_app   # reinstall (recreate wiped it)
tutor dev exec lms python manage.py lms migrate sqa_django_app
curl http://local.openedx.io:8000/sqa/health/
```

### Make the install permanent (survive recreates / image builds)
Add to the image requirements so it's baked in:
```bash
echo "-e /openedx/sqa_django_app" >> "$(tutor config printroot)/env/build/openedx/requirements/private.txt"
```
🚨 For **production** the install path is different — `private.txt` is unreliable on Tutor v21; use
`OPENEDX_EXTRA_PIP_REQUIREMENTS` (pip from GitHub). See runbook 12.

## 8. Worked example — add a new endpoint end to end

Do this once and the whole model clicks. Goal: add `GET /sqa/api/ping/` that returns the caller's
membership level. It touches every layer: route → view → (model) → switch → restart → verify.

**1. Add the view** in `api.py`:
```python
from rest_framework.views import APIView
from rest_framework.response import Response
class PingView(APIView):
    def get(self, request):
        from .models import UserMembership
        m = UserMembership.objects.filter(user=request.user).first()
        return Response({"pong": True, "level": m.level.slug if m else None})
```
**2. Route it** in `urls.py` — inside the `api_membership` block, **specific path before any
wildcard** (gotcha in §6):
```python
if waffle_switches[API_MEMBERSHIP]:
    urlpatterns += [
        path('api/ping/', api.PingView.as_view(), name='sqa_ping'),
        # …existing routes…
    ]
```
**3. (If you'd added/changed a model)** generate + apply a migration — skip here, no model change:
```bash
tutor dev exec lms python manage.py lms makemigrations sqa_django_app
tutor dev exec lms python manage.py lms migrate sqa_django_app
```
**4. Make it live.** Code edit + the switch was already on → just reload. (If you'd *just* enabled the
switch, you need a restart because routes register at import time — §6.)
```bash
tutor dev restart lms
```
**5. Verify against the running server** (not via `exec` introspection — §9 in runbook 03):
```bash
curl -H "Cookie: sessionid=<…>" http://local.openedx.io:8000/sqa/api/ping/
# {"pong": true, "level": "basic"}
```
**6. Test it fast (Tier 1)** by adding the route to `tests/api_urls.py` and a test in `tests/`, then
`pytest tests/ -v` — no LMS needed.

That's the full inner loop. A model change adds step 3; a new setting adds a `plugin_settings()` edit
(runbook 07/04); shipping it adds runbook 12.

## 9. Iteration tiers (use the fastest that catches your change)

| Tier | Tool | Speed | Covers |
|---|---|---|---|
| 1 | `pytest tests/ -v` against `test_settings.py` | seconds | models, utils, view logic (stub enrollment_api). ~80% of changes. No LMS. |
| 2 | `tutor dev` | minutes | real LMS, real signals, real OAuth/JWT, real enrollment |
| 3 | k8s deploy | 10–25 min | the real production path (runbook 12) |

## 10. Scaffolding a brand-new plugin (rare)

```bash
pip install cookiecutter
cookiecutter https://github.com/openedx/edx-cookiecutters    # choose "cookiecutter-django-app"
```
Then add the `entry_points` block (§1) to its `setup.py` and the `plugin_app` dict (§3) to its
`apps.py`. Everything else (the control design in §4, the gotchas in §6) is project-specific.

---

## What's next

- Inject settings/secrets the plugin reads → **07-writing-tutor-plugins-and-patches.md**
- Ship it to production → **12-production-deployment-k8s.md**
