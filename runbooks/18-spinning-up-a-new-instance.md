# 18 — Spinning up a new Open edX instance (copy-paste guide)

You have a working Open edX. You want **another one** — a demo for a client, a sandbox, a
second brand — on a box that already runs one.

This page is every command, in order, start to finish. Set five variables at the top and the
rest is paste-and-go.

---

## TL;DR

```
DNS (4 names) → Caddy + TLS → Tutor config → plugins → start → init → theme + admin → search → features
```

- **~45–60 min**, of which `tutor k8s init` is ~18 min of waiting.
- **No image rebuild.** You reuse the images already on the box.
- The new instance gets **its own database, its own users, its own secrets**. It shares only
  the server and the Docker images.
- Four things must be **different** from every other instance on the box: `TUTOR_ROOT`,
  Kubernetes namespace, the Caddy NodePort, and the hostnames.

**The single biggest way to cause an outage:** running a `tutor` command without `TUTOR_ROOT`
set. It silently targets **production**. Set it in every shell. Twice if you're tired.

---

## Prerequisites

1. **Ensure that you have access to a VPS** with at least 8 GB of available RAM. 
2. **Ensure that you have a user account with SSH access to the VPS**, that can `sudo` (needed for Caddy) and run
   `kubectl`.
3. **Ensure you have Caddy** as host running on ports 80/443.   
3. **OpenedX is installed on the VPS** using Tutor on k3s.
4. **A GitHub token with `read:packages`** on the org that owns the images. Repo access is
   **not** the same thing — see Troubleshooting.
5. **A domain** you can add DNS records to.
6. **Available RAM** — roughly 6 GB for the instance, plus ~2.3 GB for each background worker.

---

## Step 0 — Set your variables

Everything below uses these. **Edit the top five, paste the whole block.**.

```bash
# ---------- REPLACE WITH CORRECT VALUES ----------
export INSTANCE=[replace with instance name e.g. demo2]                                  # short name, no spaces
export LMS_HOST=[replace with host name e.g. training2.stemquestacademy.com]         # your main hostname
export NODEPORT=[replace with port e.g. 30082]                                  # MUST be unused on this box
export ADMIN_USER=[replace with admin username e.g. demoadmin]
export ADMIN_EMAIL=[replace with admin email e.g. you@example.com]
# ---------- CREATE THE FOLLOWING EXACTLY AS THEY ARE ----------
export NS=openedx-$INSTANCE
export TUTOR_ROOT=$HOME/.local/share/tutor-$INSTANCE
export CMS_HOST=studio.$LMS_HOST
export MFE_HOST=apps.$LMS_HOST
export FILES_HOST=files.$LMS_HOST
export PATH=$HOME/.local/bin:$PATH
```

**Get the list of occupied ports which are free**

```bash
kubectl get svc -A -o jsonpath='{range .items[*].spec.ports[*]}{.nodePort}{"\n"}{end}' | sort -un | tail -20
```

NodePorts are unique **across the whole cluster**, not per namespace. Production usually uses
`30080`. Pick something else. (`mfe` and `minio` also create NodePorts, but Kubernetes
auto-assigns those — you don't manage them.)

---

## Step 1 — DNS: four domain names

OpenedX needs **four** hostnames. Three are derived from `LMS_HOST` automatically:

| What | Hostname |
|---|---|
| LMS | `training2.stemquestacademy.com` |
| Studio | `studio.training2.…` |
| MFEs (newer UI pages) | `apps.training2.…` |
| Uploads / media | `files.training2.…` |

Add either **one A record + a wildcard**, or four A records, all pointing at the VPS IP:

```
A      [replace with domain name e.g training2]              <VPS_IP>
A      [replace with wildcard domain e.g. *.training2]            <VPS_IP>
```

**Verify DNS Records have propagated:**

```bash
for h in $LMS_HOST $CMS_HOST $MFE_HOST $FILES_HOST; do printf '%-45s ' "$h"; dig +short "$h" A @8.8.8.8 | tail -1; done
```

All four domains must point to your VPS IP in order to consider it a success. 

---

## Step 2 — Caddy vhost + TLS certificates

Append this block to `/etc/caddy/Caddyfile` (keep existing block):

```caddy
# ---- Open edX: INSTANCE ----
[replace with LMS domain e.g. training2.stemquestacademy.com],
[replace with studio domain e.g. studio.training2.stemquestacademy.com],
[replace with MFE Apps domain e.g. apps.training2.stemquestacademy.com],
[replace with Files domain e.g. files.training2.stemquestacademy.com] {
	reverse_proxy 127.0.0.1:30082
	request_body {
		max_size 300MB
	}
}
```

Safe way to apply it — **validate before you touch `/etc`**:

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak.$(date +%F-%H%M)   # backup
sudo nano /etc/caddy/Caddyfile                                            # paste the block
caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile          # no sudo needed
sudo systemctl reload caddy
```

> 🚨 **`reload`, never `restart`.** A restart drops every other site on the box at the same
> time. `reload` is graceful. Confirm nothing restarted:
> `systemctl show caddy -p ActiveEnterTimestamp` — the timestamp should be old.

**Get the certificates issued** (just requesting the URL triggers it):

```bash
for h in $LMS_HOST $CMS_HOST $MFE_HOST $FILES_HOST; do printf '%-45s ' "$h"; curl -s -o /dev/null -w '%{http_code} ssl=%{ssl_verify_result}\n' "https://$h/"; done
```

✅ Expect **`502 ssl=0`** for all four. `502` is *correct* — nothing is listening on your
NodePort yet. `ssl=0` means the certificate is valid. If you get `000`, TLS failed:
`sudo journalctl -u caddy -n 50`.

---

## Step 3 — Tutor config (its own secrets)

```bash
mkdir -p "$TUTOR_ROOT"
tutor config save        # generates BRAND NEW secrets
```

> 🚨 **Never copy another instance's `config.yml`.** Sharing secrets means a login token from
> one instance is valid on the other. Two platforms, two sets of keys. Always.

---

## Step 4 — Plugin files

Two small Python files. Both go in `~/.local/share/tutor-plugins/` (shared by all instances —
the files only *define* plugins; each instance chooses which to *enable*).

**4a. The NodePort bridge.** Tutor's internal Caddy isn't reachable from the host, so publish
it on your NodePort:

```bash
cat > ~/.local/share/tutor-plugins/host_proxy_nodeport_$INSTANCE.py <<EOF
from tutor import hooks

hooks.Filters.ENV_PATCHES.add_item((
    "k8s-services",
    """
---
apiVersion: v1
kind: Service
metadata:
  name: caddy-nodeport
  labels:
    app.kubernetes.io/name: caddy-nodeport
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: caddy
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: $NODEPORT
""",
))
EOF
```

**4b. Payment MFE URL.** If you have a `sqa_stripe_*` plugin, it hardcodes the *production*
MFE URL. Make your own so the new instance points at itself:

```bash
cat > ~/.local/share/tutor-plugins/sqa_stripe_$INSTANCE.py <<EOF
from tutor import hooks

hooks.Filters.ENV_PATCHES.add_item(("openedx-lms-production-settings", """
SQA_STRIPE_PRICE_TO_LEVEL = {}
SQA_PAYMENT_MFE_URL = "https://$MFE_HOST/sqa-payment"
"""))

hooks.Filters.ENV_PATCHES.add_item(("openedx-cms-production-settings", """
FEATURES["ENABLE_OTHER_COURSE_SETTINGS"] = True
"""))
EOF
```

Leave `SQA_STRIPE_PRICE_TO_LEVEL` empty unless you're demoing payments — then fill it from the
Stripe **test** dashboard. Never paste live price IDs.

---

## Step 5 — Enable plugins, then set config (this order matters)

**Enable first.** Plugin-specific config keys don't exist until the plugin is on, so setting
config first fails.

```bash
tutor plugins enable mfe minio indigo codejail \
  enable_course_settings skip_email_verification disable_public_registration \
  sqa_payment sqa_stripe_$INSTANCE host_proxy_nodeport_$INSTANCE
```

(`indigo` and `mfe` switch themselves on — that's normal.)

**Now the config.** You need two image tags. **Don't invent them** — read them off the instance
already running on this box, so the new one reuses images that are already downloaded.

```bash
export SRC=$HOME/.local/share/tutor   # TUTOR_ROOT of the EXISTING instance (used again later)
export SRC_NS=openedx                 # namespace of the EXISTING instance (not your new one)

# Read the tags from the pods that are actually running
export IMG_OPENEDX=$(kubectl -n $SRC_NS get deploy lms -o jsonpath='{.spec.template.spec.containers[0].image}')
export IMG_MFE=$(kubectl -n $SRC_NS get deploy mfe -o jsonpath='{.spec.template.spec.containers[0].image}')

echo "openedx: $IMG_OPENEDX"   # e.g. ghcr.io/scient-systems/openedx:21.0.2
echo "mfe:     $IMG_MFE"       # e.g. ghcr.io/scient-systems/openedx-mfe:21.0.0-indigo

tutor config save \
  --set LMS_HOST=$LMS_HOST \
  --set CMS_HOST=$CMS_HOST \
  --set MFE_HOST=$MFE_HOST \
  --set ENABLE_HTTPS=true \
  --set ENABLE_WEB_PROXY=false \
  --set K8S_NAMESPACE=$NS \
  --set RUN_MYSQL=true \
  --set RUN_MONGODB=true \
  --set PLATFORM_NAME="My New Platform" \
  --set DOCKER_IMAGE_OPENEDX=$IMG_OPENEDX \
  --set MFE_DOCKER_IMAGE=$IMG_MFE
```

**Check both images are already on the node** — this is what makes startup take minutes
instead of a ~4 GB download:

```bash
sudo k3s crictl images | grep openedx
```

### Where image tags come from

Three sources. They usually agree. When they don't, believe the pods.

| Source | Command | Trust it? |
|---|---|---|
| **The running pods** | `kubectl -n <ns> get deploy lms -o jsonpath='{.spec.template.spec.containers[0].image}'` | ✅ **Use this.** What is actually serving traffic |
| The other instance's config | `TUTOR_ROOT=$SRC tutor config printvalue DOCKER_IMAGE_OPENEDX` | What the config *claims*. Can drift from what was deployed |
| Cached on the node | `sudo k3s crictl images \| grep openedx` | What will start instantly — good cross-check, but lists old tags too |

Gotchas baked into that block:

- It's **`MFE_DOCKER_IMAGE`**, not `DOCKER_IMAGE_MFE`. The second one doesn't exist — it fails
  with `Missing configuration value`. The two key names are not symmetrical.
- **These are plain image references**, `registry/name:tag`. Nothing generates them for you and
  there is no "latest" to fall back on — whatever you set is what the pods pull.
- **Set `MFE_HOST` explicitly** even though it looks derivable — it's stored as a literal and
  won't re-derive later.
- `ENABLE_WEB_PROXY=false` stops Tutor fighting host Caddy for ports 80/443.
  `ENABLE_HTTPS=true` keeps generated links on `https://`.

**Sanity check:**

```bash
for k in LMS_HOST CMS_HOST MFE_HOST MINIO_HOST K8S_NAMESPACE ENABLE_WEB_PROXY; do printf '%-18s ' $k; tutor config printvalue $k; done
```

`MINIO_HOST` should have derived to `files.<your host>` on its own.

**Prove the secrets are different** (one command, non-negotiable):

```bash
TUTOR_ROOT=$SRC        tutor config printvalue OPENEDX_SECRET_KEY | sha256sum
TUTOR_ROOT=$TUTOR_ROOT tutor config printvalue OPENEDX_SECRET_KEY | sha256sum
```

The two hashes **must differ**. If they match, you copied a config — go back to Step 3.

---

## Step 6 — Namespace, image pull secret, start, init

```bash
kubectl create namespace $NS

kubectl -n $NS create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USER \
  --docker-password=YOUR_TOKEN

kubectl -n $NS patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"ghcr-pull"}]}'
```

> **Why the patch?** No Tutor manifest mentions the pull secret. Attaching it to the namespace's
> default ServiceAccount is what makes every pod use it.

**Start:**

```bash
tutor k8s start
kubectl -n $NS get pods -w        # Ctrl-C when everything is Running
```

**Then initialise** — this is the ~18 minute one:

```bash
tutor k8s init
```

✅ Ends with `All services initialised.`

> 🚨 Don't blind-retry a failed init. Look first:
> `kubectl -n $NS get jobs` then `kubectl -n $NS logs job/<name> --tail=50`.
> A half-initialised database is worth deleting and redoing:
> `kubectl delete namespace $NS` then start again.

---

## Step 7 — Theme, admin user, search

```bash
tutor k8s do settheme indigo

tutor k8s do createuser --staff --superuser -p 'CHANGE_ME_STRONG' $ADMIN_USER $ADMIN_EMAIL

tutor k8s exec cms ./manage.py cms reindex_studio --init    # Studio's own search
tutor k8s exec cms ./manage.py cms reindex_course --setup   # the PUBLIC /courses catalog
```

> **Two different search indexes.** `reindex_studio` does **not** make courses appear in the
> public catalog. You need `reindex_course` for that. Run it again after every course import.
>
> Use `--setup`, not `--all` — `--all` asks for confirmation and dies with `EOFError` because
> there's no keyboard attached.

> The theme takes 30–60 seconds to show up. An unstyled page right after `settheme` is normal.

---

## Step 8 — Turn on the SQA features

All feature switches ship **off**. Turn them on, seed the membership levels, then **restart**.

```bash
cat > /tmp/seed.py <<'EOF'
from waffle.models import Switch
from sqa_django_app.models import MembershipLevel, AccountTypeMapping

for rank, slug, name, desc in [
    (0, 'free', 'Free', 'Basic free access'),
    (1, 'basic', 'Basic', 'Basic paid membership'),
    (2, 'premium', 'Premium', 'Premium membership'),
    (3, 'enterprise', 'Enterprise', 'Enterprise membership'),
]:
    _, c = MembershipLevel.objects.get_or_create(
        slug=slug, defaults={'display_name': name, 'rank': rank, 'description': desc})
    print('level', slug, 'created' if c else 'exists')

for slug, mode, name in [
    ('student', 'sqa-students', 'SQA Students'),
    ('teacher-pd', 'sqa-pd', 'SQA PD'),
    ('professional', 'sqa-professional', 'SQA Professional'),
    ('scient-staff', 'corporate-scient', 'Corporate Scient'),
]:
    _, c = AccountTypeMapping.objects.get_or_create(
        slug=slug, defaults={'course_mode_slug': mode,
                             'course_mode_display_name': name, 'is_active': True})
    print('mapping', slug, 'created' if c else 'exists')

n = Switch.objects.filter(name__startswith='sqa_django_app.').update(active=True)
print('switches enabled:', n)
EOF

kubectl -n $NS cp /tmp/seed.py $(kubectl -n $NS get pod -l app.kubernetes.io/name=lms -o name | head -1 | cut -d/ -f2):/tmp/seed.py
tutor k8s exec lms ./manage.py lms shell -c "$(cat /tmp/seed.py)"

# REQUIRED — see below
kubectl -n $NS rollout restart deployment/lms deployment/cms
```

> 🚨 **The restart is not optional.** The plugin builds its URL list **when the process starts**.
> A switch that's off means the endpoint **doesn't exist** — you get `404`, not `403`. Flipping
> the switch changes nothing until the LMS restarts and rebuilds its URLs.
>
> There are **8** switches, not 4. `update(active=True)` above catches all of them, including
> `api_billing` — the one people miss, which makes the billing page 404 on its own.

---

## Step 9 — Workers (needed to import courses)

```bash
kubectl -n $NS scale deploy/cms-worker deploy/lms-worker --replicas=1
```

> 🚨 **Course import is a background job.** With no worker, a Studio import uploads fine and
> then **hangs at "Unpacking" forever, with no error anywhere.** Same for export and reindex.
>
> Workers cost about **2.3 GB each**. If memory is tight you can park them at `0` and scale
> `cms-worker` back to `1` before importing. **`tutor k8s start` does not restore replicas you
> scaled by hand** — it reports the deployment `unchanged` and leaves it at zero.

---

## Health checks

### The one that matters

```bash
curl -s https://$LMS_HOST/heartbeat
```

```json
{"modulestore": {"status": true, "message": "OK"},
 "sql":         {"status": true, "message": "OK"}}
```

`modulestore` is MongoDB, `sql` is MySQL. **If this returns `200` with both `true`, the LMS,
MySQL and MongoDB are all fine.** One request covers three services.

### Public endpoints

```bash
curl -s -o /dev/null -w 'lms      %{http_code}\n' https://$LMS_HOST/heartbeat
curl -s -o /dev/null -w 'sqa      %{http_code}\n' https://$LMS_HOST/sqa/health/
curl -s -o /dev/null -w 'mfe-cfg  %{http_code}\n' https://$LMS_HOST/api/mfe_config/v1
curl -s -o /dev/null -w 'studio   %{http_code}\n' https://$CMS_HOST/heartbeat
curl -s -o /dev/null -w 'mfe      %{http_code}\n' https://$MFE_HOST/
curl -s -o /dev/null -w 'minio    %{http_code}\n' https://$FILES_HOST/minio/health/live
```

| Check | Expect | Notes |
|---|---|---|
| `$LMS_HOST/heartbeat` | **200** | covers MySQL + MongoDB |
| `$LMS_HOST/sqa/health/` | **200** | our plugin is loaded |
| `$LMS_HOST/api/mfe_config/v1` | **200** | MFEs can read their config |
| `$CMS_HOST/heartbeat` | **200** | Studio |
| `$MFE_HOST/` | **200 or 302** | 302 = redirect to login, normal |
| `$FILES_HOST/minio/health/live` | **200** | MinIO |
| `$FILES_HOST/` | **403** | ✅ **normal** — MinIO denies anonymous listing |

### Inside the cluster

```bash
kubectl -n $NS get pods                                            # all Running
kubectl -n $NS exec deploy/redis -- redis-cli ping                 # PONG
kubectl -n $NS exec deploy/mongodb -- mongosh --quiet --eval "db.adminCommand({ping:1}).ok"   # 1
kubectl -n $NS exec deploy/lms -- curl -s http://meilisearch:7700/health                      # {"status":"available"}
kubectl -n $NS exec deploy/lms -- curl -s -o /dev/null -w '%{http_code}\n' http://codejailservice:8550/   # 200
curl -s -o /dev/null -w '%{http_code}\n' -H "Host: $LMS_HOST" http://127.0.0.1:$NODEPORT/     # 200 — the NodePort bridge
```

`smtp` has no HTTP endpoint — "pod is Running" is the check.

### Background jobs

```bash
kubectl -n $NS get deploy | grep worker                                      # want 1/1
kubectl -n $NS exec deploy/redis -- redis-cli LLEN edx.cms.core.default      # want 0
```

A number that only goes **up** means no worker is consuming jobs.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Import stuck on **"Unpacking"** forever, no error | no `cms-worker` | `kubectl -n $NS scale deploy/cms-worker --replicas=1` |
| API returns **404** when it should work | feature switch off, **or** LMS not restarted after switching it on | Step 8, including the restart |
| Pods `ImagePullBackOff`, **403 from ghcr.io** | token has repo access but **not package access** | Org owner: package → *Package settings* → *Manage access* → add user (Read) |
| One pod still `ImagePullBackOff` after fixing access | backoff timer, can exceed 5 min | `kubectl -n $NS delete pod <name>` to force a retry |
| `init` dies partway | DNS or cert not ready — init uploads over HTTPS | Finish Steps 1–2 first, then `kubectl delete namespace $NS` and redo |
| Site is unstyled | theme cache lag | wait 60s; if still broken, `tutor k8s do settheme indigo` |
| Studio login: *Mismatching redirect URI* | only happens on a **restored** database | fresh installs register their own callback; append (never replace) `redirect_uris` |
| Course imported but not in `/courses` | wrong index reindexed | `reindex_course --setup` (not `reindex_studio`) |
| `curl \| grep` can't find a course on `/courses` | that page is **JavaScript-rendered** | check in a browser — grep proves nothing here |
| `tutor k8s reindex-courses` → not found | that command doesn't exist | use `./manage.py cms reindex_course` |
| `reindex_course --all` → `EOFError` | it wants a keyboard | use `--setup` |
| Pods `Pending` | box is out of memory | free memory; **stop**, don't force it |
| Everything mysteriously changed on the *other* instance | `TUTOR_ROOT` was unset | always export it; verify with `tutor config printvalue LMS_HOST` |

---

## Removing it again

```bash
kubectl delete namespace $NS          # deletes pods AND all data. No undo.
rm -rf $TUTOR_ROOT
rm ~/.local/share/tutor-plugins/host_proxy_nodeport_$INSTANCE.py
rm ~/.local/share/tutor-plugins/sqa_stripe_$INSTANCE.py
# then remove the Caddy block, validate, reload
sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy
```

Other instances on the box are untouched — that's the whole point of separate namespaces and
roots.

---

## What this deliberately doesn't do

- **No image rebuild.** If you want different *branding baked in* (logos inside the MFEs), that
  is a rebuild, ~73 min, and out of scope here.
- **No CI/CD.** The pipeline knows about production and staging. A new instance is updated by
  hand, and drifts from day one.
- **No backups.** Set them up separately if the instance holds anything you care about.
- **Nothing is copied from production** — no users, no courses, no secrets. Import courses as
  `.tar.gz` files if you want content. Courses travel; people don't.

---

## What's next

- Deploying a change to an existing instance → **17-shipping-a-change.md**
- Production deployment and the reverse-proxy setup → **12-production-deployment-k8s.md**
- When something breaks → **15-troubleshooting-and-gotchas.md**
