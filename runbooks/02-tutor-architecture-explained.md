# 02 — Tutor architecture explained

Tutor is the Docker-based manager for Open edX. Almost everything you do touches Tutor, so spend
10 minutes here — it will save you days. Nothing in this runbook is a command you must run right
now; it's the mental model the other runbooks assume.

We run **Tutor 22.0.2** (the "Verawood" Open edX release) since October 2026. The previous line,
Tutor 21 / Ulmo, is retired. On the servers the CLI is installed as **`tutor22`**; on your laptop
it is plain `tutor`. Everything below applies to both.

---

## 1. The three run modes

Tutor runs the *same* Open edX in three different ways. Pick the right one for the job:

| Mode | Command prefix | What it is | When |
|---|---|---|---|
| **dev** | `tutor dev …` | Dockerized, with **live code mounting + hot reload + dev settings**. The only mode where editing a file on your host changes the running app. | Day-to-day local development. |
| **local** | `tutor local …` | Dockerized, **production-like** on one machine (no live mounting, prod settings, Caddy/TLS). | Rarely — a prod smoke test on your laptop. |
| **k8s** | `tutor22 k8s …` (servers) | Deploys to a **Kubernetes** cluster (k3s on our two VPSes). | Production, staging, k-12, the demo. Runbook 12. |

> 🚨 You cannot iterate on code in `local` or `k8s` — there's no mounting. For dev work it's
> always `tutor dev`. (Task discipline: "DO NOT use `tutor local` for iteration.")

## 2. Where Tutor keeps everything

```
~/.local/share/tutor/
├── config.yml          ← your settings + generated secrets (JWT key, DB passwords, MFE host…)
├── env/                ← GENERATED. Tutor renders this from templates on `tutor config save`.
│   ├── build/openedx/  ← the Dockerfile + theme + requirements baked into the LMS image
│   └── plugins/mfe/…   ← generated env.config.jsx etc.
└── data/               ← container data volumes (MySQL, Mongo, media)

~/.local/share/tutor-plugins/      ← our custom Tutor plugins (*.py). See runbook 07.
```

That is the layout on your laptop. On the **servers** each site has its own pair, named after
it, and both must be exported before any `tutor22` command:
`TUTOR_ROOT=~/.local/share/tutor-prod-verawood` and
`TUTOR_PLUGINS_ROOT=~/.local/share/tutor-prod-verawood-plugins` (runbook 12 lists all four).

Two rules that follow from this:

- **`env/` is generated — never hand-edit it.** Your edits get overwritten on the next
  `tutor config save`. Change the *source* (a plugin, or `config.yml`) and re-run `tutor config save`.
- **`config.yml` holds real secrets** (JWT private key, DB passwords). Don't commit it anywhere.

Inspect config values:
```bash
tutor config printroot                 # path to ~/.local/share/tutor
tutor config printvalue LMS_HOST       # read one value
tutor config save --set KEY=VALUE      # set one value, then re-render env/
```

## 3. The render pipeline (the thing to internalize)

```
config.yml  +  enabled plugins  ──[ tutor config save ]──▶  env/  (rendered files)
                                                              │
                                          ┌───────────────────┴───────────────────┐
                            [ tutor images build openedx/mfe ]            [ tutor dev/k8s start ]
                                          │                                       │
                                   custom Docker image                    running containers
```

- **`tutor config save`** re-renders `env/` from templates + plugins. Fast (seconds). You run it
  after changing `config.yml` or any Tutor plugin.
- **`tutor images build <name>`** bakes `env/` (+ git-cloned source) into a Docker image. Slow
  (10–45 min). Needed after changes that must live *inside* the image (Python deps, themes,
  MFE bundles). For our servers **CI does this**, never a person on a box (runbook 12).
- **`tutor dev/local/k8s start`** runs containers from the image.

**Knowing which of these three a change requires is the whole game.** See the §4 table in
runbook 00 and the per-task runbooks.

## 4. Plugins — how Tutor is extended

A Tutor **plugin** is a Python file (or pip package) that hooks into Tutor's render pipeline.
Ours live in `~/.local/share/tutor-plugins/`. List them:

```bash
tutor plugins list
```

On your laptop:
```
indigo          ✅ enabled   theme + MFE component injection (forked tutor-indigo)
mfe             ✅ enabled   runs all the MFEs (official plugin)
sqa_payment     ✅ enabled   registers our sqa-payment MFE + the "Membership" nav link
sqa_stripe_dev  ✅ enabled   injects Stripe test keys into LMS dev settings
```

On the servers (staging shown): `indigo`, `mfe`, `minio`, `forum`, `codejail`, `sqa_payment`,
`sqa_stripe_staging`, `sqa_gateway_prod`, `enable_course_settings`, `host_proxy_nodeport`
(publishes Tutor's Caddy to the host Caddy) and `studio_content_groups_redirect`. Each site has
its own `sqa_stripe_*` and NodePort plugin.

Enable/disable:
```bash
tutor plugins enable <name>
tutor plugins disable <name>
tutor config save           # ALWAYS re-render after enable/disable
```

Writing your own plugin → runbook **07**.

## 5. Patches — how a plugin injects into Open edX

A plugin can't edit Open edX's source. Instead Tutor exposes named **patches** — insertion points
in the generated files. A plugin adds a string to a patch; `tutor config save` stitches it in.

The patches you'll actually use:

| Patch name | Injects into | Applies in |
|---|---|---|
| `openedx-common-settings` | Django settings, **both** LMS+CMS | all envs |
| `openedx-lms-common-settings` / `openedx-cms-common-settings` | LMS-only / CMS-only settings | all envs |
| `openedx-lms-production-settings` | LMS settings | `local` + `k8s` |
| `openedx-lms-development-settings` | LMS settings | `dev` only |
| `openedx-cms-development-settings` | CMS settings | `dev` only |
| `openedx-dockerfile-post-python-requirements` | extra `RUN pip install …` in the image | image build |
| `mfe-lms-common-settings` | the `MFE_CONFIG` dict the LMS serves to MFEs | all envs |
| `mfe-env-config-runtime-definitions[-<app>]` | code into the assembled `env.config.jsx` | mfe image build |
| `mfe-dockerfile-post-npm-install-<app>` | extra `RUN npm install …` per MFE | mfe image build |

> 🚨 **dev vs production settings are different patches.** `sqa_stripe_dev.py` patches
> `openedx-lms-**development**-settings`; the prod equivalent patches
> `openedx-lms-**production**-settings`. A dev patch is invisible in `k8s` and vice-versa. This is
> *why* dev "worked" but prod didn't, more than once.

Concrete example (from `sqa_stripe_dev.py`):
```python
from tutor import hooks
hooks.Filters.ENV_PATCHES.add_item(("openedx-lms-development-settings", """
SQA_STRIPE_SECRET_KEY = "sk_test_..."
"""))
```

## 6. Config defaults — a plugin can add Tutor config keys

```python
hooks.Filters.CONFIG_DEFAULTS.add_item(("SQA_PAYMENT_MFE_URL", "http://apps.local.openedx.io:2003/sqa-payment"))
```
Then anyone can override per-environment with `tutor config save --set SQA_PAYMENT_MFE_URL=https://…`.

## 7. Images: openedx vs mfe (and why secrets leak into images)

Two custom images matter:

- **openedx** — the LMS/CMS image. Bakes in: our Django plugin (pip from GitHub), the legacy theme
  (tutor-indigo templates), and any `openedx-dockerfile-*` patches.
- **mfe** — serves all the MFEs as static bundles via Caddy. Bakes in: each MFE's compiled JS,
  including `env.config.jsx` and brand CSS.

> 🚨 The openedx image installs the **private** plugin repo via a token embedded in the pip URL
> (`OPENEDX_EXTRA_PIP_REQUIREMENTS`, written as `x-access-token:<token>@github.com/…`). That
> token can end up in the image's pip cache layer — **treat the image as containing a credential.
> Only push to a *private* registry (GHCR private).** CI does all of this for you (runbook 12).

## 8. Image names and tags

On a fresh local install, the `indigo` plugin appends `-indigo` to the default image names, so
`DOCKER_IMAGE_OPENEDX` becomes `…/openedx-indigo`. Check with
`tutor config printvalue DOCKER_IMAGE_OPENEDX`.

On the servers that never applies: every site's config sets both names explicitly, to
`ghcr.io/scient-systems/openedx:<tag>` and `ghcr.io/scient-systems/openedx-mfe:<tag>`, where the
tag is the hash of `openedx-deploy`'s `versions.yml`. Runbooks 11 and 12.

---

## What's next

- Actually start the platform → **03-running-openedx-locally.md**
- Write a Tutor plugin / patch → **07-writing-tutor-plugins-and-patches.md**
