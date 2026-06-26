# 15 — Troubleshooting & gotcha index

A symptom → cause → fix index of (nearly) every trap we've hit. `Ctrl-F` your symptom. Linked
runbook has the full context.

---

## Tutor / containers

| Symptom | Cause | Fix |
|---|---|---|
| `tutor: command not found` in a script | non-login shell; `~/.local/bin` not on PATH | use a **login** shell: `wsl -e bash -lc "..."` (01) |
| `tutor dev do init` hangs on `docker compose ls` | Docker Desktop lost the WSL connection | restart Docker Desktop, retry (03) |
| Init fails mid-build, GHCR timeout | transient network | just re-run; Docker layer cache resumes (03) |
| `No installed app with label 'sqa_django_app'` | you used `tutor dev run` (throwaway container) | use `tutor dev exec` (03, 06) |
| Settings/INSTALLED_APPS look "NOT SET" via CLI | same — `run` skips the installed plugin | inspect the running container with `exec` (06) |
| Code edit not reflected after `pip install` | `restart` only reloads file changes | full `stop && start -d` after any pip install/mount/env change (03) |
| Plugin gone after `stop/start` | editable install doesn't survive a recreate | `pip install -e /openedx/sqa_django_app` again (03) |
| `tutor mounts add ~/path` mounts nothing | bare path → empty `compose_mounts` | use `svc1,svc2:/host:/container` form; verify `tutor mounts list` (03) |
| MFE folder mount not binding | folder name ≠ MFE repo name | rename so it matches exactly (08) |

## Django plugin

| Symptom | Cause | Fix |
|---|---|---|
| URL resolves to `/sqa/sqa/health/` | put `sqa/` in `urls.py` paths | paths are relative to the `^sqa/` prefix — drop it (06) |
| `NoReverseMatch` on `redirect('sqa_…')` | missing namespace | `redirect('sqa_django_app:sqa_…')` (06) |
| Route 404s after enabling a waffle switch | switches read at import time | restart LMS (dev: `restart`; k8s: `rollout restart`) (04, 06) |
| `AppRegistryNotReady` at startup | top-level `from .models import` in signals.py | move model imports inside handler bodies (06) |
| `TypeError: Field 'id' expected a number but got UserData(...)` | event handler got a dataclass, not a User | `User.objects.get(id=user.id)` first (06) |
| 405 on `POST /api/user/assign/` | wildcard route registered before specific | put specific paths before `<str:username>` (06) |
| Enrollment filter does nothing | `fail_silently` defaults True | set `fail_silently: False` explicitly (06) |
| `OPEN_EDX_FILTERS_CONFIG` stays unset in dev | `PluginSettings.CONFIG` not called by tutor dev chain | register it in `AppConfig.ready()` (06) |
| Plugin page shows raw Mako source | extended `main.html` (Mako) | extend `main_django.html`, use `{% block body %}` (06) |
| Plugin page unstyled / white text unreadable | no Bootstrap in `main_django.html`; dark theme overrides text color | self-contained `<style>`, explicit `color:#1a1a1a !important` (06) |
| `OperationalError` on waffle table at startup | `ready()` runs before tables exist | keep the try/except in `waffle_init()` (06) |
| Login fails: "User has no profile" | `createsuperuser` makes no `UserProfile` | create one via shell (03, 04) |
| `course-v1:…,` → 500 `InvalidKeyError` | trailing comma in admin course-key field | enter the clean key (05, 13) |

## opaque-keys / pip

| Symptom | Cause | Fix |
|---|---|---|
| `opaque-keys` not found on PyPI | wrong name | it's `edx-opaque-keys` (01) |
| pip backtracks for 30+ min | `Django` unpinned in `base.in` | pin `Django>=4.2,<5.3` (01) |
| install blocked on Python 3.10 | `python_requires>=3.12` (cookiecutter default) | lowered to `>=3.8` (01) |
| local: `typing.Self` ImportError | `edx-opaque-keys>=2.13` needs 3.11 | `pip install "edx-opaque-keys==2.12.0"` locally (01) |
| LMS won't start: `cannot import name 'CollectionKey'` | opaque-keys pin in `constraints.txt` downgraded the LMS | 🚨 keep that pin OUT of constraints.txt (01) |

## MFEs / slots

| Symptom | Cause | Fix |
|---|---|---|
| `Template syntax error: Expected an expression` building env.config | `{{ }}` in a `.jsx` (even a comment) | hoist styles to a `const`; remove all `{{`/`}}` (09) |
| `ReferenceError: MyWidget is not defined` | missing the runtime-definitions line | add `{{- patch("MyWidget.jsx") }}` to `mfe-env-config-runtime-definitions` (09) |
| `ReferenceError` even with the line | put it in `buildtime-imports` (wrong patch) | move it to `runtime-definitions` (09) |
| Every MFE build breaks | one bad `.jsx` (raw concatenation) | fix/shrink the component (09) |
| Footer widget vanishes | footer slot dedupe drops `indigo_footer` if >2 inserts | use a different slot (09) |
| VPS build fails: file not found | new `tutorindigo/` file not `git add`-ed | commit it; VPS builds the committed tree (09, 10) |
| Local `tutor config save` doesn't show fork change | local runs pip indigo 21.1.3, not the fork | validate on the VPS (09, 11) |
| `tutor dev restart learner-dashboard` → no such service | it's a bundle inside `mfe` | restart `mfe` (08, 12) |
| MFE page blank, header only, no JS error | wrong Router basename (`PUBLIC_PATH` unset) | `wrapWithRouter={false}` + own `<BrowserRouter basename>`; set `PUBLIC_PATH` (08) |
| MFE page fully blank, console: "X required by Studio Footer" | `@edx/frontend-component-footer` `ensureConfig` fails in prod | remove that footer or set the 4 footer config keys (08) |
| webpack "Module not found: ./X" on mfe build | new file not pushed to GitHub | commit+push first; build clones from GitHub (08, 12) |

## Theming

| Symptom | Cause | Fix |
|---|---|---|
| Brand CSS never applies | `raw.githubusercontent.com` serves `text/plain` + nosniff | use jsDelivr pinned to a commit SHA (10) |
| jsDelivr `@ulmo/indigo` 404 | branch name has a slash | pin to a commit SHA, not the branch (10) |
| Responsive brand rules dead in prod | `--pgn-size-breakpoint-*` custom media ships raw, browsers drop it | use plain px queries (768px = md) (10) |
| 40-min mfe rebuild ships stale brand CSS | `npm install @edx/brand@…#branch` is a cached layer | `--no-cache` / bump the line; prefer runtime `brandOverride` (10) |
| Token CSS change "not deploying" on k8s | restart alone doesn't refresh ConfigMaps | `tutor k8s start` then `rollout restart deployment/lms` (10) |
| `INDIGO_WELCOME_MESSAGE` default ignored | `config.yml` override beats plugin default | `tutor config printvalue INDIGO_WELCOME_MESSAGE` (10, 11) |
| Footer logo looks wrong inverted | someone applied `filter: invert()` | never invert the footer logo (10) |

## Deploy / k8s

| Symptom | Cause | Fix |
|---|---|---|
| Rollout runs OLD code after push | `imagePullPolicy: IfNotPresent` + same tag | patch `imagePullPolicy: Always`, then rollout restart (12) |
| `tutor images push mfe` rejected | `MFE_DOCKER_IMAGE` points at overhangio default | set it to your GHCR before first build (12) |
| Build fast but new code missing | Docker cached the `pip …@main` layer | pin the commit SHA, or `--no-cache` (12) |
| Pod CrashLoopBackOff, `No module named pkg_resources` | ran `pip install --force-reinstall` in a live pod | 🚨 never do that; `tutor images build openedx --no-cache` (12) |
| `migrate` says "nothing" but table missing | a prior run fake-applied it | `migrate <app> <prev> --fake` then `migrate` real (12) |
| Plugin "not found" in pod after deploy | image cached old / build didn't include it | verify in image with `docker run … pip show`; check the Always patch (12) |
| Setting from a Tutor patch ignored in prod | `plugin_settings()` overwrites with `os.environ.get` | use a k8s Secret env var, not a patch, for those (04, 12, 13) |
| Dev setting "works", prod doesn't | dev and prod are different patches | add the `…-production-settings` twin (02, 07) |

## Stripe

| Symptom | Cause | Fix |
|---|---|---|
| Webhook signature invalid after new endpoint | new endpoint = new `whsec_` | copy it into the `sqa-stripe` Secret, rollout (14) |
| Can't retry a failed webhook | `processed=True` set even on handler error | delete the `StripeEvent` row to reprocess (14) |
| `AttributeError`/`.get()` fails on Stripe object | SDK v15 typed objects | attribute access + `[]`, never `.get()` (14) |
| Membership not updating after payment | webhook is the only write path | check webhook delivery + `StripeEvent` rows (14) |

## Gateway / broker

| Symptom | Cause | Fix |
|---|---|---|
| Gateway can't find DB on Vercel | Neon injects prefixed env vars, not `DATABASE_URL` | `AliasChoices` maps the prefixed var (13) |
| `/internal/...` 401 | wrong/missing bearer | `Authorization: Bearer <SQA_GATEWAY_INTERNAL_SECRET>` matches both sides (13) |
| Token sync silently fails | LMS↔gateway secret mismatch or no envFrom | check `sqa-gateway` Secret + `sqa_proxy_reconcile` (13) |

---

## Universal "nothing changed" debugging recipe

Grep your change down the chain; the first miss is the broken hop:
1. VPS clone (the source file)
2. `$(tutor config printroot)/env/...` (after `tutor config save`)
3. `docker run --rm <image> grep …` (after build)
4. `kubectl -n openedx exec deploy/<svc> -- grep …` (after push + rollout + the Always patch)

## "Which pipeline do I need?" — see runbook 00 §4 and the per-area runbooks.
