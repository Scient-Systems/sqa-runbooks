# C03 — Django & the plugin idea (how our code gets inside Open edX)

> **Goal:** understand just enough Django to read our plugin, and understand the *plugin
> mechanism* — the single most important architectural fact of this project.

## 1. Django in seven ideas

Django is a Python web framework. Everything in `edx-platform` and in our
`sqa_django_app` is built from these seven pieces:

1. **App** — a folder of related functionality (models + views + urls). A Django *project*
   (like the LMS) is a stack of dozens of apps. Our plugin is one more app.
2. **Settings** — one giant configuration namespace (`DEBUG`, database credentials,
   installed apps, our `SQA_*` values). Everything reads from it.
3. **URLs → Views** — a table mapping URL patterns to Python functions/classes. A view
   takes a request, returns a response. Our REST endpoints are views.
4. **Models → ORM → Migrations** — a model is a Python class describing a database table.
   You never write SQL. When you *change* a model, you generate a **migration** — a small
   script that alters the real table. Migrations run in order, once, at deploy time.
   *Golden rule: model change without migration = crash at runtime.*
5. **Admin** — the free auto-generated back-office UI at `/admin`. Waffle switches,
   memberships, Stripe events — all inspectable there (→ runbook 04).
6. **Signals** — see §3. The nervous system.
7. **Management commands** — CLI verbs you add to `manage.py` (we use them for
   bootstrapping switches and levels).

If you know these words, you can navigate 90% of our Python.

## 2. The plugin mechanism — code injection, the sanctioned way

Problem: we want our membership logic to run *inside* the LMS — same process, same
database connection, same signal bus — **without editing edx-platform's code**.

The mechanism (Python-standard, not Open edX magic): **entry points**. Our `setup.py`
declares:

```python
entry_points={'lms.djangoapp': ['sqa_django_app = sqa_django_app.apps:...']}
```

Think of it as a **business card in a public registry**: when the LMS boots, it asks
Python "who registered under `lms.djangoapp`?", finds our card, and installs our app into
itself — settings, URLs, signal handlers, everything. Delete that one line and the plugin
silently stops existing. (This is gotcha #1 in → runbook 06.)

Our `apps.py` then carries a `plugin_app` dict telling the LMS *what* to mount:
- `PluginURLs` — mount our REST endpoints under the LMS's URL space
- `PluginSettings` — let us add/modify Django settings
- `PluginSignals` — subscribe our handlers to LMS events

**The trade (know it cold):** in-process means we can call real LMS Python APIs
(`enrollment_api`), hear every signal, and share the user table with zero sync. The price:
our exceptions are LMS exceptions (we can 500 the platform), our deploys ride the big
openedx image (→ concepts/05), and we must stay compatible with the LMS's Django version.
The alternative — a separate microservice — flips every one of those: isolated and
independently deployable, but deaf to signals, needing its own auth, database
synchronization, and network error handling. For enrollment gating, deaf-to-signals is
disqualifying (see §4).

## 3. Signals — the platform's doorbell

A **signal** is a broadcast: "X just happened, whoever cares, react." The sender doesn't
know or care who's listening.

- Django's classic version: `@receiver` on ad-hoc signals. Legacy; loosely typed.
- Open edX's modern version: **openedx-events** — a curated catalog of named, versioned
  events (`ENROLLMENT_CREATED`, `CERTIFICATE_AWARDED`, …) with typed payloads. **We use
  these** (function-style handlers registered via `PluginSignals`), never the legacy kind.

Why events instead of calling us directly? Because enrollments happen from *everywhere*:
the student dashboard, the admin panel, bulk imports, other plugins. None of those code
paths know our plugin exists — but they all emit the same event. Subscribing once covers
every door into the building, including doors added later.

## 4. Case study: dual-layer enrollment gating (how we actually used all this)

Requirement: a Free user must not enroll in a Premium course.

- **Layer 1 (proactive):** our own enroll endpoint checks eligibility and returns 403
  *before* enrolling. Nice UX, but only protects OUR door.
- **Layer 2 (safety net):** a handler on the enrollment-created event re-checks
  eligibility and **unenrolls** if the enrollment came through any other door.

Both layers call **one** function: `check_enrollment_eligibility()` in `utils.py`. That's
deliberate — the rank-comparison rule exists in exactly one place, so it cannot drift.
*Never re-implement it elsewhere; import it.*

And everything sits behind **waffle switches** — database-backed feature flags, all
defaulting OFF, toggled in Django Admin. Concept: **shipping code and activating behavior
are two separate decisions.** We can deploy gating code weeks before daring to turn it on,
and kill it in 10 seconds without a deploy if it misbehaves. (The four switches:
`api_membership`, `api_enrollment`, `gate_enrollment`, `auto_assign` — → runbook 06.)

Alternatives we didn't pick for gating: overriding LMS templates (cosmetic only — the API
would still enroll you), or **openedx-filters** (intercept-and-block *before* the action;
actually a very clean fit for blocking enrollment — the trade-off is that filters can only
veto what they intercept, while our signal layer also *repairs* — it catches paths that
predate or bypass any filter and unenrolls after the fact).

## You're ready when…

- You can explain the entry-point line to someone in one sentence, and what happens if
  it's missing (nothing loads, no error — silence).
- You know why a model edit isn't done until `makemigrations` ran and the migration is
  committed.
- You can argue both sides of "plugin vs microservice" and say why signals settled it.
- You know where the eligibility rule lives, and that the answer has to be "one place".

**Next:** [C04 — Open edX anatomy](04-openedx-anatomy.md).
