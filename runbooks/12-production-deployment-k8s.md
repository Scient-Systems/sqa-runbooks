# 12 — Production deployment (Tutor 22 on k8s)

How code reaches the servers. Every site runs **Open edX Verawood on Tutor 22** (`tutor22`)
on Kubernetes (k3s). Two custom images, `openedx` and `mfe`, live on **GHCR** and are built
by CI in [`openedx-deploy`](https://github.com/Scient-Systems/openedx-deploy). This runbook is
the reference for the environments, the deploy script, secrets and the gotchas. For the
day-to-day flow (push → pin → build → staging → production) read **17** first.

> 🚨 Before any production deploy: a backup and a known previous tag. The production workflow
> takes the backup for you; doing it by hand, take it yourself.

---

## 0. The four sites

| Site | LMS / MFE host | Box | Tutor root (`~/.local/share/…`) | Namespace | Deployed by |
|---|---|---|---|---|---|
| Production | `lms.stemquestacademy.com` / `apps.lms.…` | VPS 1 | `tutor-prod-verawood` | `openedx` | "Deploy production" workflow |
| Demo | `training-demo.stemquestacademy.com` / `apps.training-demo.…` | VPS 1 | `tutor-demo-verawood` | `openedx-demo` | by hand |
| Staging | `staging.stemquestacademy.com` / `apps.staging.…` | VPS 2 | `tutor-staging-verawood` | `openedx` | automatic after every build |
| k-12 | `k-12.stemquestacademy.com` / `apps.k-12.…` | VPS 2 | `tutor-verawood` | `openedx-verawood` | by hand |

Studio is `studio.stemquestacademy.com` on production (a *sibling* of `lms.`) and
`studio.<lms-host>` everywhere else.

Every root has its own plugins folder next to it (`<root>-plugins`). Every command on a box
needs both set:

```bash
export PATH="$HOME/.local/bin:$PATH"
export TUTOR_ROOT=$HOME/.local/share/tutor-prod-verawood
export TUTOR_PLUGINS_ROOT=$TUTOR_ROOT-plugins
tutor22 config printvalue LMS_HOST        # ✅ always check you are where you think you are
```

> 🚨 **Only `tutor22`.** Plain `tutor` (Tutor 21, Ulmo) is uninstalled on both boxes and must
> stay that way. On 2026-10-26 one run of it against staging re-applied Ulmo manifests:
> Meilisearch went back to v1.8.4 and crash-looped on the newer data, MySQL, MongoDB and Caddy
> were downgraded, and the LMS lost its secret references. The Ulmo roots are archived in
> `~/archive/ulmo-20261027/` on each box.

> 🚨 **VPS 1 runs production and the demo together**, with about 8 GB of memory free at rest.
> Anything heavy you run there (a Studio shell, a reindex, a course import or publish loop) is
> production-risk. Check `free -h` first. On 2026-10-22 two overlapping Studio publish loops on
> the demo froze the whole box, production included, until it was restarted from the hosting
> panel.

The theme plugin is loaded from **`~/tutor-indigo-verawood`**, one checkout shared by every
root on a box. `config save` renders settings from it (the brand CSS link among them), so it
must sit at the `tutor-indigo` commit pinned for the images you run. Fetch by URL:
`git -C ~/tutor-indigo-verawood fetch https://github.com/Scient-Systems/tutor-indigo.git verawood/indigo`.

---

## 1. How images are built

CI builds both images from `versions.yml` on every push to `openedx-deploy` `main` that touches
it (`.github/workflows/build-images.yml`):

- **Tag** = the first 12 characters of `sha256(versions.yml)`, shared by both images
  (`ghcr.io/scient-systems/openedx:<tag>`, `ghcr.io/scient-systems/openedx-mfe:<tag>`). Any
  change to the manifest is a new tag, so Kubernetes can never serve a stale image under an old
  name, and `imagePullPolicy` stops mattering.
- **Verified before publishing:** the openedx image must contain `sqa_django_app` at exactly the
  pinned commit; the mfe image must contain the compiled payment app. Otherwise nothing is
  pushed.
- **Time:** ~7 minutes when the previous tag's cache is warm; a cold MFE build is up to ~1 hour.
- **Private sources** (`sqa_django_app`, `frontend-app-sqa-payment`) are fetched with the
  `MFE_CLONE_TOKEN` secret, written as `x-access-token:<token>` in the URL. GitHub redirects the
  fetch, and the git inside the image drops a bare token on the redirect.

> 🚨 **No server ever builds an image.** If CI is down, fix CI. A server build leaves the
> registry and the manifest disagreeing about what is running.

Manual run (any branch): `gh workflow run build-images.yml -R Scient-Systems/openedx-deploy
--ref main -f image=both -f push=true`. If the MFE job fails on `npm` `ECONNRESET`, re-run just
that job.

---

## 2. How a deploy runs: `scripts/deploy.sh`

Staging (automatic) and production (workflow) both copy `scripts/deploy.sh` to the box and run:

```bash
bash /tmp/deploy.sh <staging|prod> <tag> [tutor-indigo-commit]
```

What it does, in order:

1. Uses `tutor22` with that environment's root and plugins folder — never anything else
2. Refuses to start with less than **3 GB** `MemAvailable` (`MIN_MEM_GB` to change)
3. Backs up the environment's `config.yml`
4. Fetches `tutor-indigo` by URL and checks out the pinned commit in the shared checkout
5. Saves the config with the two new image tags and renders
6. **Refuses to apply** if anything other than the two app images would change
   (`ALLOW_SERVICE_CHANGE=1` overrides, deliberately)
7. `k8s start`, waits for the rollouts, runs migrations via `kubectl exec`
8. Prints the previous tag and theme commit — **rollback is the same command with those**

It is a **code** deploy only. It decrypts nothing and never rewrites environment config
(hostnames, secrets, per-site settings): that lives on the box in the Tutor root.

The production workflow adds, before `deploy.sh`: a check that staging already runs that exact
tag, and a full backup copied off the box. There is **no second-person approval** — GitHub's
required reviewers need a paid plan on a private repo — so a human typing the tag is the gate.

### k-12 and the demo

Not in the pipeline. Roll them forward by hand with the tag staging runs: back up (MySQL dump,
Mongo dump, `config.yml`, plugins), check out the matching theme commit, `tutor22 config save
--set DOCKER_IMAGE_OPENEDX=… --set MFE_DOCKER_IMAGE=…`, check the rendered settings, `k8s
start`, migrate, then confirm the **other** environment on the same box is untouched (its
`config.yml` hash and deployments). Each keeps its own settings in its Tutor config, which the
roll-forward must not lose — see runbook **17** §4b.

---

## 3. Production secrets → k8s Secrets (NOT plugin files, NOT git, NOT images)

Created by hand, per environment. `SECRETS.md` in `openedx-deploy` lists every one.

```bash
kubectl -n openedx create secret generic sqa-stripe \
  --from-literal=SQA_STRIPE_SECRET_KEY=sk_live_... \
  --from-literal=SQA_STRIPE_WEBHOOK_SECRET=whsec_... \
  --from-literal=SQA_STRIPE_PUBLISHABLE_KEY=pk_live_... \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n openedx create secret generic sqa-gateway \
  --from-literal=SQA_GATEWAY_URL="https://sqa-proxytoken-service.vercel.app" \
  --from-literal=SQA_GATEWAY_INTERNAL_SECRET="<secret>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

The plugin's `settings/common.py` reads these via `os.environ`. 🚨 This is *why* they are k8s
Secrets and not a Tutor patch — `plugin_settings()` overwrites with `os.environ.get(...)`, so a
Tutor patch would be silently clobbered.

**Image pulls** use the `ghcr-pull` docker-registry secret attached to each namespace's
`default` ServiceAccount (`openedx-deploy/k3s/serviceaccount-pull-secret.sh`). If pods sit in
`ImagePullBackOff` with `403` from ghcr.io, the token in that secret has been revoked: replace
it in **every** namespace on the box and delete the stuck pods.

---

## 4. Migrations

`deploy.sh` runs them. By hand:

```bash
kubectl -n $NS exec deploy/lms -- ./manage.py lms migrate --noinput
kubectl -n $NS exec deploy/cms -- ./manage.py cms migrate --noinput
kubectl -n $NS exec deploy/lms -- ./manage.py lms showmigrations sqa_django_app
```

🚨 If `migrate` says "No migrations to apply" but a table is missing, a prior run fake-applied it.
Recover: `migrate sqa_django_app <prev> --fake` then `migrate` for real.

---

## 5. 🚨 NEVER `pip install --force-reinstall` in a running pod

It can remove `pkg_resources`/`setuptools`; the running process survives but the **next restart
crashes the pod** (`ModuleNotFoundError: No module named 'pkg_resources'`, CrashLoopBackOff).
Fix code by shipping a new image through CI, never by mutating a live pod.

---

## 6. Observability

```bash
kubectl -n $NS get pods
kubectl -n $NS get deploy lms mfe -o jsonpath='{range .items[*]}{.metadata.name}={.spec.template.spec.containers[0].image}{"\n"}{end}'
kubectl -n $NS logs deployment/lms --tail 50
kubectl -n $NS logs deployment/caddy --since=1h | grep '"status":500'   # every 500, with its URL
kubectl -n $NS exec -i deploy/lms -- ./manage.py lms shell < script.py
free -h                                                                   # before anything heavy
```

🚨 `tutor22 k8s start` applies manifests and ConfigMaps; to recycle pods on an unchanged image
use `kubectl rollout restart`. Toggling a waffle switch needs a `rollout restart deployment/lms`
to take effect (the plugin builds its URL list at startup).

🚨 Never `pkill -f <pattern>` when the pattern also appears in your own ssh command line — it
kills your own shell. Find the PID and kill that.

---

## What's next

- The everyday flow, k-12 and the demo → **17-shipping-a-change.md**
- A new instance on a box → **18-spinning-up-a-new-instance.md**
- Deploy the gateway / Streamlit companion services → **13-deploying-companion-services.md**
- Stripe specifics (keys/webhook/prices) → **14-stripe-payments-integration.md**
- Anything broke? → **15-troubleshooting-and-gotchas.md**
