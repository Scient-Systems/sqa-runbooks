# 04 — Admin mode & user management

Covers: the Django admin site, superusers, staff/instructor roles, and **waffle switches** (the
feature flags our plugin lives behind). Commands shown for `tutor dev`; for production swap
`tutor dev exec lms` → `tutor k8s exec lms --`.

---

## 1. The Django admin site

Open edX exposes Django's admin at:
- Local LMS: `http://local.openedx.io:8000/admin/`
- Prod LMS: `https://lms.stemquestacademy.com/admin/`

You need a **superuser** (or `is_staff`) account to log in. From the admin you can edit almost any
DB record: users, waffle switches, our membership levels, course requirements, Stripe events, proxy
tokens, course integration grants, etc.

Our plugin's models appear under the **SQA_DJANGO_APP** section of the admin **only if** the plugin
is installed (`pip install -e` done) and the container restarted — see runbook 03/06. If the section
is missing, the app isn't in `INSTALLED_APPS`.

## 2. Superusers

```bash
# Create
tutor dev exec lms python manage.py lms createsuperuser
```
🚨 Always also create the `UserProfile` or Open edX login fails with "User has no profile" — see
runbook 03 §3 for the snippet.

Promote an existing user instead:
```bash
tutor dev exec lms ./manage.py lms shell -c "
from django.contrib.auth import get_user_model
u = get_user_model().objects.get(username='<USER>')
u.is_staff = u.is_superuser = True; u.save(); print('promoted')
"
```

## 3. Course-level roles (staff, instructor, beta)

Open edX has *global* roles (superuser/staff) and *per-course* roles. Per-course roles control who
can edit a course in Studio and who can use the Instructor Dashboard in the LMS.

**Easiest (UI):** in the LMS, open the course → **Instructor → Membership** tab → add users as
*Staff* or *Admin*. In **Studio**, open the course → **Settings → Course Team** to grant authoring
access.

**CLI** (grant Studio author + course staff):
```bash
tutor dev exec lms ./manage.py lms manage_user <USER> <email> --superuser   # global, if needed
# Per-course staff role:
tutor dev exec lms ./manage.py lms shell -c "
from django.contrib.auth import get_user_model
from common.djangoapps.student.roles import CourseStaffRole, CourseInstructorRole
from opaque_keys.edx.keys import CourseKey
u = get_user_model().objects.get(username='<USER>')
ck = CourseKey.from_string('course-v1:SQA+SQA405+2026_T2')
CourseStaffRole(ck).add_users(u)
CourseInstructorRole(ck).add_users(u)
print('course roles granted')
"
```

## 4. Waffle switches — the feature flags (READ THIS)

Our plugin keeps entire feature groups OFF until you flip a switch. **All switches default OFF.**
They are normal DB rows you can edit in Django Admin → **Waffle → Switches**, or via shell.

The switches and what they gate:

| Switch | Gates |
|---|---|
| `sqa_django_app.api_membership` | the membership REST endpoints (`/sqa/api/levels/`, `/sqa/api/user/…`) |
| `sqa_django_app.api_enrollment` | the enrollment endpoint (`/sqa/api/enroll/`) |
| `sqa_django_app.gate_enrollment` | the enrollment gate: our API returns 403, AND an openedx-filters step blocks ineligible enrollments from ANY source *before* the row is written (also enables the CMS course-requirement sync) |
| `sqa_django_app.auto_assign` | auto-assigning a level on registration |
| `sqa_django_app.api_billing` | the Stripe billing endpoints (status/checkout/webhook) |
| `sqa_django_app.api_proxy` | the AI-token broker endpoints |

Flip them in the shell:
```bash
tutor dev exec lms python manage.py lms shell -c "
from waffle.models import Switch
for n in ['sqa_django_app.api_membership','sqa_django_app.api_billing']:
    Switch.objects.update_or_create(name=n, defaults={'active': True}); print(n,'ON')
"
```

### 🚨 Two switch gotchas that have bitten us repeatedly

1. **Switches are evaluated at module import time** (in `urls.py`). Toggling one in the DB does
   nothing until the LMS process restarts:
   - dev: `tutor dev restart lms`
   - k8s: `kubectl -n openedx rollout restart deployment/lms`
2. **`gate_enrollment` is disruptive when turned on.** Once on, the openedx-filters step **blocks
   any enrollment** (new enrollment or re-enrollment, from any source — dashboard, admin, bulk import,
   our API) by a user below the course's required level, and the CMS publish-sync starts writing
   `CourseMembershipRequirement` rows from Studio's Advanced Settings. So a course's requirement
   suddenly becomes load-bearing. **Never enable it in production until the end-to-end flow is
   verified** — confirm the requirements and your test users' levels are what you expect first.
   (Same "verify before enabling in prod" caution for `auto_assign`.)
   > Note: the gate intercepts enrollment *attempts*; it does not retroactively scan and unenroll
   > already-enrolled students. The real-world disruption is that any affected student's next
   > enrollment action fails — plan the rollout accordingly.

## 5. Where Django settings come from (and how to add one)

You don't edit `edx-platform` settings directly. Settings reach the LMS three ways on this project:

1. **From the plugin** — `sqa_django_app/settings/common.py` `plugin_settings()` and the app's
   `ready()`. (Note: in `tutor dev`, the `PluginSettings.CONFIG` chain is unreliable — we register
   the openedx-filters config inside `ready()` instead. See runbook 06.)
2. **From a Tutor plugin patch** — e.g. `sqa_stripe_dev.py` patches
   `openedx-lms-development-settings`. Runbook 07.
3. **From a k8s Secret as env vars** — the production way for secrets (Stripe keys, gateway secret).
   `common.py` reads them via `os.environ`. Runbook 12/13.

🚨 Precedence trap: `plugin_settings()` runs and may overwrite a Tutor patch with
`os.environ.get(..., '')`. That's exactly why gateway settings come from a **k8s Secret (env var)**,
not a Tutor `ENV_PATCHES` plugin — a patch would be silently clobbered.

Verify a setting landed in the running LMS:
```bash
tutor dev exec lms python -c "import django; django.setup(); from django.conf import settings; print(bool(settings.SQA_STRIPE_SECRET_KEY))"
```

## 6. Studio Advanced Settings (course authors, no Django needed)

Some plugin behavior is course-author-facing. Our `sqa_stripe_dev.py` enables
`FEATURES["ENABLE_OTHER_COURSE_SETTINGS"]` so an author can set `membership_required_level` on a
course in **Studio → Settings → Advanced Settings** without touching Django admin. Runbook 05 §6.

---

## What's next

- Build and import a course → **05-creating-and-managing-courses.md**
- The plugin's internals (where these models/switches are defined) → **06**
