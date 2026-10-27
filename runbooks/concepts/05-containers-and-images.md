# C05 — Containers, images, registries: how code gets boxed and shipped

> **Goal:** understand Docker well enough that image builds, layer caching, tags, and
> registry authentication stop being rituals and start being obvious. Two of our real
> production incidents were pure "didn't understand this layer" (→ war stories #2, #5).

## 1. The container idea

A **container** is a process in a box: it gets its own filesystem, network view, and
process list, but shares the machine's kernel. Unlike a virtual machine there's no second
operating system — it starts in milliseconds and costs almost nothing extra.

Why we care: the LMS needs *exact* versions of Python, system libraries, node, and a
hundred pip packages. A container ships **the entire environment**, so "works on my
machine" becomes "works in the image, therefore works everywhere".

## 2. Images and layers (this explains the weird build behavior)

An **image** is the frozen filesystem a container starts from — built from a
**Dockerfile**, a recipe of steps (`FROM ubuntu`, `RUN pip install…`, `COPY code…`).

Crucial mechanic: **every step produces a layer, and layers are cached.** When you
rebuild, Docker walks the recipe top-down and reuses each cached layer *if the step looks
unchanged* — for `COPY`, it fingerprints the actual files (reliable); for `RUN`, **only
the command text** (treacherous!).

> 🚨 The treachery, concretely — our war story #5: the MFE Dockerfile has
> `RUN npm install '@edx/brand@github:Scient-Systems/brand-openedx#ulmo/indigo'` (today the
> same line points at `#verawood/indigo`, with the same trap).
> We push new CSS to that branch, rebuild… and Docker says "that command text is
> unchanged" and reuses the *months-old cached layer*. The 40-minute build succeeds and
> ships stale CSS with zero warning. Fix: `--no-cache` (or change the command text).
> Rule of thumb: **any RUN step that fetches something mutable from the network is a
> caching landmine.**

Also: layer order matters. Recipes put slow-changing stuff (OS, dependencies) first and
fast-changing stuff (our code) last, so a normal code change only rebuilds the tail.
That's why our openedx image rebuilds take 10 minutes, not 40.

## 3. Tags vs digests (movable stickers vs fingerprints)

- A **digest** (`sha256:8f3c…`) is the true, immutable ID of an image.
- A **tag** (`openedx:21.0.2`, our old Ulmo-era tag) is a *sticker* someone can peel off and
  put on a different image tomorrow. `tutor images push` did exactly that: same tag, new content.

So "the cluster runs 21.0.2" was **not** a complete statement — it ran *whatever that tag
pointed to when the node last pulled it*. Corollary: pushing a new image under the same
tag does nothing until pods are recreated *and* the node decides to re-pull.

**How we fixed it:** CI now tags every build with the first 12 characters of the sha256 of
`openedx-deploy`'s `versions.yml` (e.g. `openedx:7bde8ac98ba1`). Any change to any input is a
new tag, so a tag is never reused and behaves like a digest. Same idea as how we pin the theme
CSS: jsDelivr URLs use a **commit SHA**, so caches can never serve the wrong version
(→ concepts/08).

## 4. Registries and pull authentication

A **registry** is a warehouse for images. Ours is **GHCR** (GitHub Container Registry),
and our packages are **private** — because the openedx image contains our proprietary
plugin code.

Private means *every* puller must authenticate — and there are **two different pullers
with two different credential stores**:

1. **The build machine**: today that is GitHub Actions in `openedx-deploy`, which pushes
   with the workflow's own `GITHUB_TOKEN` — no personal token to expire. (It reads the two
   private *source* repos with a separate secret, `MFE_CLONE_TOKEN`.)
2. **The Kubernetes node**: when a pod starts and the image isn't cached locally, the
   *kubelet* pulls — using an `imagePullSecret` stored in the cluster
   (ours: `ghcr-pull` in **every** namespace, attached to its `default` ServiceAccount). A
   login anywhere else does not help the cluster. Different program, different credential
   store.

> 🚨 PATs expire. When ours expired: builds failed (`denied` even pulling *public* base
> images — a stale login poisons every ghcr request), and later the *cluster* couldn't
> pull, so new pods hung in `ImagePullBackOff` while the site was down (→ war story #2).
> The node had been coasting on its local image cache for weeks, hiding the dead
> credential. Lesson: **a cached image is a loan, not an asset** — any pod reschedule can
> call it in.

## 5. Our images, concretely

| Image | Contains | Rebuilt when | Typical cost |
|---|---|---|---|
| `ghcr.io/scient-systems/openedx:<tag>` | edx-platform + **our plugin** + legacy theme | any pin in `versions.yml` changes | built by CI |
| `ghcr.io/scient-systems/openedx-mfe:<tag>` | *all* MFE bundles compiled, incl. sqa-payment + baked brand CSS | same | built by CI |

Both images always share one tag. A build of both takes ~7 minutes when the previous tag's
layer cache is warm, up to about an hour cold.

Everything else (mysql, redis, caddy…) = stock public images, never rebuilt.

The full loop: push code → change its pin in `versions.yml` → CI renders the Dockerfiles with
Tutor (→ concepts/06), builds, checks our code is inside, pushes under the new tag → the deploy
points the site's config at that tag and recreates the pods (→ concepts/07). Break any link and
you get the classic "I deployed but nothing changed" (→ concepts/11 for the hop-by-hop hunt).

## 6. Trade-off corner

- **Private registry** (chosen): code stays private; cost: credential lifecycle
  management — rotation, pull secrets, expiry incidents. A public image would delete that
  whole failure class at the price of publishing our plugin.
- **Registry-based deploys** (chosen, forced by k8s): images travel via GHCR even though
  build machine and cluster are the same VPS. `tutor local` (docker-compose) would use
  the local docker daemon directly — no push, no pull secrets. That simplicity is a real
  argument for compose on a single machine; we paid the k8s tax for other reasons (→ C07 §6).

## You're ready when…

- You can explain why a rebuild was fast, or suspiciously fast (cache hit on a layer that
  should have changed).
- "Tag" vs "digest" is instinct: you never say "it's pushed" and mean "it's running".
- You know the two pull credential stores, and which one `ImagePullBackOff` points at.

**Next:** [C06 — the Tutor mental model](06-tutor-mental-model.md) — the tool that writes
all these Dockerfiles for us.
