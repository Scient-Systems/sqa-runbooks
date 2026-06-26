# 03 — Running Open edX locally (tutor dev daily workflow)

How to bring the platform up on your machine and the day-to-day loop. Assumes runbook 01 is done.

Local URLs once running:
- LMS: `http://local.openedx.io:8000`
- Studio/CMS: `http://studio.local.openedx.io:8001`
- MFEs: `http://apps.local.openedx.io:1996/…` (learner-dashboard), `:2003/sqa-payment`, etc.

---

## 1. First-time init (once per machine, ~1–2 hours, unattended)

This builds the full LMS Docker image from scratch (clones edx-platform, Python 3.11, Node, webpack)
and initializes databases.

```bash
tutor dev launch          # or: tutor dev do init  (see note)
```

- `tutor dev launch` = build images + start everything + run init (migrations, default admin user).
  **Note the admin credentials it prints.**
- If it hangs at the very first step (`docker compose ls`): Docker Desktop lost its WSL connection —
  restart Docker Desktop from the Windows tray, retry.
- If it fails mid-build with a GHCR/network timeout: transient — just run it again. Docker's layer
  cache resumes where it stopped.

## 2. The daily loop

### Start / stop
```bash
tutor dev start -d lms              # -d = detached (background)
tutor dev logs lms --tail 10        # look for: "Starting development server at http://0.0.0.0:8000/"
tutor dev stop                      # stop everything
tutor dev stop lms                  # stop one service
```

### 🚨 `exec` vs `run` — the difference that wastes hours
```bash
tutor dev exec lms <cmd>    # runs INSIDE the already-running LMS container ✅ use this
tutor dev run  lms <cmd>    # spins up a THROWAWAY --rm container ❌ plugin not installed there
```
A throwaway `run` container does **not** have our pip-installed plugin, so
`manage.py … migrate sqa_django_app` errors with *"No installed app with label 'sqa_django_app'"*,
`INSTALLED_APPS` won't list it, and settings look "NOT SET". **Always use `exec` to inspect or act
on the running LMS.**

### 🚨 `restart` vs `stop/start` — the other one
```bash
tutor dev restart lms                       # reload signal; Django StatReloader picks up FILE edits
tutor dev stop lms && tutor dev start -d lms # RECREATES the container (fresh filesystem)
```
- `restart` is enough after editing Python that's already installed editable.
- `restart` is **NOT** enough after: a `pip install`, adding/removing a mount, or env changes —
  those need a full stop/start (a recreate).
- 🚨 A recreate gives you a **fresh container** → your editable install is gone → you must
  `pip install -e /openedx/sqa_django_app` again (see §4).

`manage.py` note: Open edX's `manage.py` needs `lms` (or `cms`) as the first argument:
`tutor dev exec lms python manage.py lms migrate`.

## 3. Create a usable superuser (with a profile!)

```bash
tutor dev exec lms python manage.py lms createsuperuser   # interactive
```

🚨 `createsuperuser` makes a Django `User` but **no `UserProfile`** — Open edX login then fails with
*"User has no profile"*. Fix it:
```bash
tutor dev exec lms ./manage.py lms shell -c "
from django.contrib.auth import get_user_model
from common.djangoapps.student.models import UserProfile
u = get_user_model().objects.get(username='<YOURUSER>')
UserProfile.objects.get_or_create(user=u, defaults={'name': '<Your Name>'})
"
```

## 4. Mount + install our Django plugin into the running LMS

The plugin source is mounted from your host so edits hot-reload. One-time mount setup (already done
on this machine; verify with `tutor mounts list`):

```bash
# 🚨 The bare-path form (tutor mounts add ~/sqa_django_app) creates an EMPTY mount — wrong.
# Use the explicit service:host:container form:
tutor mounts add "lms,cms,lms-worker,cms-worker:$HOME/sqa_django_app:/openedx/sqa_django_app"
tutor mounts list     # compose_mounts should list all 4 services

# Recreate so the volume binds, then editable-install:
tutor dev stop lms && tutor dev start -d lms
tutor dev exec lms pip install -e /openedx/sqa_django_app
# Recreate ONE more time so the entry_point is discovered, then reinstall (it was wiped):
tutor dev stop lms && tutor dev start -d lms
tutor dev exec lms pip install -e /openedx/sqa_django_app
```

After this, `tutor dev restart lms` is enough for subsequent Python edits — **until** the next
recreate, after which you reinstall again. (To make the install permanent across rebuilds, add
`-e /openedx/sqa_django_app` to the image requirements — see runbook 06/12.)

🚨 `pip install -e /openedx/sqa_django_app` respects the plugin's `constraints.txt`. If that file
pins `edx-opaque-keys` too low, it silently downgrades the LMS and breaks `django.setup()`. (We keep
that pin out of constraints on purpose — see runbook 01 §7.)

### Apply plugin migrations
```bash
tutor dev exec lms python manage.py lms migrate sqa_django_app
```
Re-run whenever you pull code that adds migrations (idempotent). If it errors with
`ModuleNotFoundError` for a new dep (e.g. `stripe`), the editable install predates it —
`tutor dev exec lms pip install -e /openedx/sqa_django_app` then retry.

### Verify the plugin is live
```bash
curl http://local.openedx.io:8000/sqa/health/
# Expected: {"status":"ok","plugin":"sqa_django_app"}
```

## 5. Bootstrap data + feature flags

```bash
# 4 membership levels + anything else the init command seeds
tutor dev exec lms python manage.py lms sqa_django_app_init

# Turn on the feature flags the plugin gates behind waffle switches (persists in DB):
tutor dev exec lms python manage.py lms shell -c "
from waffle.models import Switch
for n in ['sqa_django_app.api_membership','sqa_django_app.api_enrollment','sqa_django_app.gate_enrollment','sqa_django_app.auto_assign']:
    Switch.objects.update_or_create(name=n, defaults={'active': True}); print(n,'ON')
"
```
🚨 Waffle switches are read at **module import time** in `urls.py`, so after toggling one you must
`tutor dev restart lms` for routes to register. More on switches in runbook 04.

## 6. Import a demo course (so you have something to look at)

Easiest path is Studio UI (runbook 05). The classic demo course key is
`course-v1:OpenedX+DemoX+DemoCourse`.

## 7. Health/debug quick reference

```bash
# entry_point registered?
tutor dev exec lms python -c "import importlib.metadata as m; print(list(m.entry_points(group='lms.djangoapp')))"
# in INSTALLED_APPS?
tutor dev exec lms python -c "import django; django.setup(); from django.conf import settings; print([a for a in settings.INSTALLED_APPS if 'sqa' in a])"
# does the URL resolve in the running server?
tutor dev logs lms --tail 50
```
🚨 `resolve('/sqa/health/')` succeeding under `tutor dev exec` does NOT prove the running server has
it — `exec` spawns a fresh Python process reading current state; the server loaded its URLs at its
own startup. Trust `curl` against the running server, not `exec` introspection.

---

## What's next

- Manage users, roles, and feature flags → **04-admin-and-user-management.md**
- Build/import a course → **05-creating-and-managing-courses.md**
- Understand the plugin you just installed → **06-installing-a-django-plugin-app.md**
