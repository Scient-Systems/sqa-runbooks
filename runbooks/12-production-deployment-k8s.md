# 12 — Production deployment (Tutor k8s)

How code reaches production. Our VPS runs **Tutor k8s** (Kubernetes) on Tutor v21. Two custom images
(`openedx` and `mfe`) live on **GHCR**. This runbook is the canonical build → push → roll-out
procedure plus every deploy gotcha we've hit.

Production hosts: LMS `lms.stemquestacademy.com`, MFEs `apps.lms.stemquestacademy.com`.
Images: `ghcr.io/scient-systems/openedx-indigo`, `ghcr.io/scient-systems/openedx-mfe:21.0.0-indigo`.

> 🚨 Before any prod deploy: DB backup + a version tag. Deploys are hard to reverse.

---

## 1. One-time setup

### 1a. Install the Django plugin into the openedx image (Tutor v21 way)
🚨 Tutor v21 does **not** use `private.txt` for prod — it silently does nothing. Use
`OPENEDX_EXTRA_PIP_REQUIREMENTS`. The repo is **private**, so embed a PAT (scopes: `repo`,
`read:packages`) in the pip URL — Docker build has no git credentials:
```bash
tutor config save --set "OPENEDX_EXTRA_PIP_REQUIREMENTS=['git+https://<PAT>@github.com/Scient-Systems/sqa_django_app.git@main']"
```
🚨 The PAT ends up in the image's pip cache layer → **the registry must stay private** (GHCR private).

### 1b. Point images at your own registry (before the first build)
```bash
tutor config save --set "DOCKER_IMAGE_OPENEDX=ghcr.io/scient-systems/openedx"   # indigo appends -indigo
tutor config save --set "MFE_DOCKER_IMAGE=ghcr.io/scient-systems/openedx-mfe:21.0.0-indigo"
```
🚨 If you skip this, `tutor images push mfe` is rejected because the default points at
`docker.io/overhangio/...` which you can't push to. (Bug 5 in PROD_DEPLOYMENT.)

### 1c. Production secrets → k8s Secrets (NOT plugin files, NOT git, NOT images)
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

# Wire each secret into the LMS pod as env vars (envFrom):
kubectl -n openedx patch deployment lms --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/envFrom/-","value":{"secretRef":{"name":"sqa-gateway"}}}]'
kubectl -n openedx rollout restart deployment/lms
```
The plugin's `settings/common.py` reads these via `os.environ`. 🚨 This is *why* secrets are k8s
Secrets and not a Tutor patch — `plugin_settings()` overwrites with `os.environ.get(...)`, so a Tutor
patch would be silently clobbered.

## 2. Deploy the Django plugin (openedx image)

```bash
tutor images build openedx
# ✅ verify the plugin is actually IN the image before pushing — don't waste a push/deploy:
docker run --rm $(tutor config printvalue DOCKER_IMAGE_OPENEDX) pip show sqa-django-app
tutor images push openedx

# Force k8s to pull the new image (see §4), then roll:
kubectl -n openedx patch deployment lms \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"lms","imagePullPolicy":"Always"}]}}}}'
kubectl -n openedx rollout restart deployment/lms
kubectl -n openedx rollout status deployment/lms --timeout=300s

# ✅ verify in the live pod + endpoint:
tutor k8s exec lms -- pip show sqa-django-app
curl https://$(tutor config printvalue LMS_HOST)/sqa/health/

# If you added migrations:
tutor k8s exec lms -- python manage.py lms migrate sqa_django_app
```

### 🚨 Bust the Docker cache when only `@main` changed
The pip URL ends `@main`. Docker caches that layer keyed on the **string**, not the git HEAD — new
commits to `main` don't invalidate it. Either pin the current SHA, or rebuild `--no-cache`:
```bash
COMMIT=$(git ls-remote https://<PAT>@github.com/Scient-Systems/sqa_django_app.git main | cut -f1)
tutor config save --set "OPENEDX_EXTRA_PIP_REQUIREMENTS=['git+https://<PAT>@github.com/Scient-Systems/sqa_django_app.git@${COMMIT}']"
tutor images build openedx
# or, brute force (30–45 min): tutor images build openedx --no-cache
```
Use `--no-cache` for the **first** deploy after adding migrations (so the new migration files are in
the image).

## 3. Deploy an MFE (mfe image)

```bash
# 🚨 commit + push the MFE / tutor-indigo repo FIRST — the build clones from GitHub, ignores local files
tutor config save --set SQA_PAYMENT_MFE_URL=https://apps.lms.stemquestacademy.com/sqa-payment  # bakes the nav href
tutor images build mfe
# ✅ verify the app compiled in:
docker run --rm $(tutor config printvalue MFE_DOCKER_IMAGE) ls /openedx/dist/ | grep sqa   # → sqa-payment/
tutor images push mfe

kubectl -n openedx patch deployment mfe \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"mfe","imagePullPolicy":"Always"}]}}}}'
kubectl -n openedx rollout restart deployment/mfe
kubectl -n openedx rollout status deployment/mfe --timeout=300s

# ✅ verify:
curl -I https://apps.$(tutor config printvalue LMS_HOST)/sqa-payment/      # HTTP 200
```
🚨 `tutor dev restart learner-dashboard` / `restart mfe-app` don't exist — the service is `mfe`;
individual MFEs are static bundles inside it.

## 4. 🚨 The imagePullPolicy trap (re-apply EVERY same-tag push)

k8s uses containerd with `imagePullPolicy: IfNotPresent` by default. If you push a new image under
the **same tag**, k8s won't re-pull — the node has it cached → your rollout runs the OLD code. You
MUST patch `imagePullPolicy: Always` before each rollout where the tag is unchanged (shown in §2/§3).
This patch is also lost if a deployment is recreated — re-apply it then.

## 5. Migrations on k8s
```bash
tutor k8s exec lms -- python manage.py lms migrate sqa_django_app
tutor k8s exec lms -- python manage.py lms showmigrations sqa_django_app
```
🚨 If `migrate` says "No migrations to apply" but a table is missing, a prior run fake-applied it.
Recover: `migrate sqa_django_app <prev> --fake` then `migrate` for real.

## 6. 🚨 NEVER `pip install --force-reinstall` in a running pod
It can remove `pkg_resources`/`setuptools`; the running process survives but the **next restart
crashes the pod** (`ModuleNotFoundError: No module named 'pkg_resources'`, CrashLoopBackOff). The only
correct fix is `tutor images build openedx --no-cache` → push → deploy a fresh image. Fix pip/image
problems by rebuilding the image, never by mutating a live pod.

## 7. Condensed "subsequent deploy" recipes

```bash
# Django plugin (after pushing code):
tutor images build openedx && docker run --rm $(tutor config printvalue DOCKER_IMAGE_OPENEDX) pip show sqa-django-app
tutor images push openedx
kubectl -n openedx patch deployment lms -p '{"spec":{"template":{"spec":{"containers":[{"name":"lms","imagePullPolicy":"Always"}]}}}}'
kubectl -n openedx rollout restart deployment/lms && kubectl -n openedx rollout status deployment/lms --timeout=300s

# MFE (after pushing code):
tutor images build mfe && tutor images push mfe
kubectl -n openedx patch deployment mfe -p '{"spec":{"template":{"spec":{"containers":[{"name":"mfe","imagePullPolicy":"Always"}]}}}}'
kubectl -n openedx rollout restart deployment/mfe && kubectl -n openedx rollout status deployment/mfe --timeout=300s
```

## 8. Observability
```bash
kubectl -n openedx get pods
kubectl -n openedx logs deployment/lms --tail 50
kubectl -n openedx logs deployment/mfe --tail 20
tutor k8s exec lms -- python manage.py lms shell -c "..."
```
🚨 On k8s, `tutor k8s start` applies manifests/ConfigMaps; to recycle pods on an unchanged image tag
use `kubectl rollout restart`. Toggling a waffle switch needs a `rollout restart deployment/lms` to
take effect (switches read at import time).

---

## What's next

- Deploy the gateway / Streamlit companion services → **13-deploying-companion-services.md**
- Stripe specifics (keys/webhook/prices) → **14-stripe-payments-integration.md**
- Anything broke? → **15-troubleshooting-and-gotchas.md**
