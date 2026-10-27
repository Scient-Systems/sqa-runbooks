# 17 — Shipping a change (the developer's view)

Everything you need to get a code change from your laptop to production, in one page.
No jumping to other runbooks. If you read only one thing before your first change, read this.

---

## 1. The six repos, and which ones matter to you

Four repos **build into the platform**. Change any of them and you are changing what users see.

| Repo | What lives there | Branch you push to |
|---|---|---|
| `sqa_django_app` | Our Django plugin: memberships, enrollment gating, payments backend, pathways | `main` |
| `frontend-app-sqa-payment` | The payment UI (pricing, checkout, manage) | `main` |
| `tutor-indigo` | The theme and the MFE widgets (header, dashboard, catalog, course-complete page). Our fork | `verawood/indigo` |
| `brand-openedx` | Brand CSS: colours, logos, fonts, plus the demo's second palette | `verawood/indigo` |

> 🚨 The platform is **Verawood (Tutor 22)** everywhere since 2026-10-27. The old Ulmo branches
> (`tutor-indigo` `release`, `brand-openedx` `ulmo/indigo`) are frozen: a push there reaches no
> site. GitHub still shows them as the default branch on those two repos (changing that needs repo
> admin), so check the branch before you push.

Two repos are **infrastructure**. You will rarely touch these.

| Repo | What lives there |
|---|---|
| `openedx-deploy` | The pipeline itself, plus per-environment config. **This is the only repo that knows about all four above.** |
| `vps-infra` | The server itself: the other services on the box, proxy, cluster. Nothing to do with Open edX. |

> **Why the pipeline lives in `openedx-deploy` and not in the code repos:** four separate repos
> build into one image. No single code repo can own that build, and two of them
> (`tutor-indigo`, `brand-openedx`) are public while the other two are private. So the build
> sits above all of them.

---

## 2. What happens when you push

You do **two** things by hand in this whole flow: **merge a pin change**, and **click deploy**.
Everything else happens on its own.

```
  you push to a code repo
        │
        ▼
  its own tests run          ← in that repo. red = stop, nothing else happens
        │ green
        ▼
  a pin PR in openedx-deploy     ← AUTOMATIC for sqa_django_app only; by hand for the others
        │                          (see below). Changes your repo's ref in versions.yml
        ▼  ← YOU MERGE IT TO main   (human step 1)
        │
  images build in the cloud      ← AUTOMATIC. ~7 min when the cache is warm, up to ~1 hour cold
  and are checked before publish
        │
        ▼
  staging deploys itself         ← AUTOMATIC (scripts/deploy.sh), then runs the smoke check
        │
        ▼  ← YOU LOOK AT STAGING, then CLICK DEPLOY   (human step 2)
        │
  production                     ← backs up first, refuses anything staging hasn't run
        │
        ▼  ← BY HAND, when you choose
  k-12 and the demo              ← not in the pipeline; see §4b
```

### The PR you will see

It appears in **`openedx-deploy`**, not in the repo you pushed to. It is titled something like:

```
pin: sqa_django_app 2f8fad270a46 -> a91c04ff21be
```

All it does is change one line in `versions.yml`. Merging it into `main` is what says *"this
change is ready to be built and shipped."* Until you merge, nothing is built.

### Which repos open that PR for you

Only **`sqa_django_app`** does today. Its tests go green, it notifies `openedx-deploy`, and
the pin PR appears. The other three do not:

| Repo | Why no PR | What you do |
|---|---|---|
| `tutor-indigo` | its notify job skips: the repo has no `DEPLOY_DISPATCH_TOKEN` secret | edit its `ref:` in `versions.yml` by hand, PR to `main` |
| `brand-openedx` | no notify job on `verawood/indigo` | see §3 |
| `frontend-app-sqa-payment` | no workflows at all | edit its `ref:` by hand, PR to `main` |

The pin-bump job also refuses a commit that is not on the branch `versions.yml` names, so a
push to an old Ulmo branch can never sneak into a build.

### What "checked before publish" means

The build opens the finished image and confirms our code is genuinely inside it and is the
exact commit `versions.yml` asked for. **If it is not, the build fails and publishes nothing.**

This exists because we were repeatedly bitten by builds that looked fine but quietly shipped
old code. That is now impossible rather than merely unlikely.

---

## 3. 🔗 The one coupled pair: `brand-openedx` and `tutor-indigo`

**If you are changing brand CSS, read this bit properly.** It is the only place in the system
where one repo's change requires another repo to change too.

Brand CSS is used **two ways at once**:

| Where | How it is pinned |
|---|---|
| Baked into the MFE image at build time | the `brand-openedx` `ref:` in `versions.yml` |
| Served live to browsers from a CDN (jsDelivr) | pinned to a **specific commit** by `BRAND_DIST_REF`, a line inside **`tutor-indigo`** `tutorindigo/plugin.py` |

The second one is what actually puts CSS in front of users. It lives in a *different repo*
from the one you edited.

### What you do

1. Edit the SCSS in `brand-openedx` (branch `verawood/indigo`)
2. Run `make build` — this regenerates `dist/` (and `dist-scient/`, the demo's palette)
3. **Commit `dist/` and `dist-scient/` as well as your source.**
4. Push to `verawood/indigo` and note the commit SHA
5. In **`tutor-indigo`** (`verawood/indigo`), set `BRAND_DIST_REF` to that SHA, commit, push
6. In **`openedx-deploy`**, set **both** refs in `versions.yml` — `brand-openedx` to the brand
   SHA, `tutor-indigo` to the commit from step 5 — and PR it to `main`

Nothing opens these for you any more: `brand-openedx` has no notify job on `verawood/indigo`.

> ⚠️ **Skip step 5 and it fails silently and completely**: every check goes green, the build
> passes, staging deploys, and the site keeps serving the **old** CSS while you wonder why your
> change did not appear. Nothing in the pipeline catches it today (a brand-drift check exists
> on the old `ci/build-images` branch but is not in `main`'s `smoke.sh`), so do all three steps.

> ✅ Check before step 6: `curl -sI https://cdn.jsdelivr.net/gh/Scient-Systems/brand-openedx@<sha>/dist/core.min.css`
> returns `200`. jsDelivr serves any public commit, so this proves the CSS will be reachable.

---

## 4. Deploying to production

Staging is automatic. Production is a deliberate human action.

1. Go to **`openedx-deploy` → Actions tab**
2. In the **left sidebar**, click **"Deploy production"**
   *(it will not appear in the main run list until it has run at least once — look in the sidebar)*
3. Click **"Run workflow"** on the right
4. Paste the **tag** and run

**Where to get the tag:** it is whatever staging is currently running. It is printed in the
build's summary and in the staging deploy's summary. It looks like `7bde8ac98ba1` — the first
12 characters of the sha256 of `versions.yml`, so every change to the manifest is a new tag.

Three things happen automatically before your change reaches users:

- **It checks staging is already running that exact tag.** If not, it refuses. You cannot
  promote something that has not been validated.
- **It takes a full backup** of the database and all uploaded files, and copies it off the
  server. If any part of the backup is missing, it stops before touching anything.
- **It runs the same smoke check** (`scripts/smoke.sh`) afterwards.

`scripts/deploy.sh` does the work on the box in both cases. It uses `tutor22` with the
environment's own root, refuses to start with less than 3 GB of memory free, backs up the
config, moves the shared theme checkout to the pinned `tutor-indigo` commit, and refuses to
apply if anything other than the two app images would change.

> ⚠️ There is **no second-person approval** on production: GitHub's required reviewers need a
> paid plan for a private repo. A human must still trigger the run and type the tag.

### Rolling back

Same three clicks, with the **previous tag**. Every tag we have ever built is kept, so you can
always go back. Roll back first, investigate afterwards.

---

## 4b. k-12 and the demo: by hand

The pipeline knows **staging and production only**. The other two sites are rolled forward by
hand when you decide, with the same images:

| Site | Box | Tutor root | Namespace |
|---|---|---|---|
| k-12 (`k-12.stemquestacademy.com`) | VPS 2, with staging | `tutor-verawood` | `openedx-verawood` |
| Demo (`training-demo.stemquestacademy.com`) | VPS 1, with production | `tutor-demo-verawood` | `openedx-demo` |

The steps: back up, `tutor22 config save --set DOCKER_IMAGE_OPENEDX=… --set MFE_DOCKER_IMAGE=…`
with the tag staging runs, check the rendered settings, `tutor22 k8s start`, migrate, then
check the neighbour on the same box is untouched. The demo's own scripts are in
`openedx-deploy/environments/demo/scripts/`.

Each of these sites keeps **settings of its own** in its Tutor config, and a roll-forward must
not lose them:

- **k-12:** no footer links (`INDIGO_FOOTER_NAV_LINKS=[]`), membership hidden
  (`INDIGO_SHOW_MEMBERSHIP=false`), name "Stem Quest Academy"
- **Demo:** the Scient brand profile (`INDIGO_BRAND_PROFILE=scient`), no footer links,
  sign-up closed, membership and AI tokens hidden

> 🚨 **VPS 1 is shared by production and the demo** and has about 8 GB free at rest. Never run
> course or content fixes there as a loop of Studio shells or per-course publishes: on
> 2026-10-22 that froze the whole box, production included, until it was restarted from the
> hosting panel. One shell, save only, check `MemAvailable` first.

---

## 5. Things that will confuse you the first time

**"My PR opened in the wrong repo."** It did not. Pin PRs always open in `openedx-deploy`,
because that is where the version of everything is recorded.

**"I merged my code but nothing deployed."** Merging in the *code* repo does not deploy.
Merging the *pin PR* into `openedx-deploy` `main` is what starts a build. For every repo except
`sqa_django_app`, nobody opens that PR for you (§2).

**"The theme looks wrong right after a deploy."** Give it a few minutes. The themed page takes
minutes to appear after any deploy, and the plain-looking page is served in the meantime. This
is normal and known. Do not go debugging it in the first ten minutes.

**"CI failed but the error is nothing to do with my code."** If you see *"job was not acquired
by Runner"* or *"Internal server error"*, that is GitHub's infrastructure, not you. The job
never ran. Check <https://githubstatus.com>, then re-run it.

**"The build took an hour this time."** Any change to `versions.yml` rebuilds both images. With
the cache warm from the previous tag that is about 7 minutes; a cold MFE build is closer to an
hour. If the MFE job fails on `npm` `ECONNRESET`, that is the npm registry cutting downloads
off mid-build, not your change: re-run the failed job.

**"The build says it can't read the private repo."** The build fetches `sqa_django_app` and
the payment MFE with the `MFE_CLONE_TOKEN` secret, written as `x-access-token:<token>` in the
URL. GitHub redirects the fetch and older git drops a bare token on the redirect. If the token
was revoked, set a new one (read access to both repos):
`gh secret set MFE_CLONE_TOKEN -R Scient-Systems/openedx-deploy`.

---

## 6. What the pipeline deliberately never does

Do not wait for a deploy to do any of these — it will not.

- **Turn features on.** Code ships with feature switches **off**. You flip them by hand in
  Django Admin. `gate_enrollment` especially: flipped carelessly it would mass-unenroll real
  students.
- **Change secrets.** Rotating a key is a manual operation, never a pipeline run.
- **Build images on a server.** If the build system is down, the answer is to fix it, not to
  build by hand on production.

---

## 7. The short version

- Push to your repo's Verawood branch. Tests run.
- **Merge the pin PR into `openedx-deploy` `main`** (open it yourself unless it's `sqa_django_app`).
- Images build and staging deploys itself. Go look at staging.
- **Click "Deploy production" and paste the tag.**
- Roll k-12 and the demo forward by hand when you want them on it too.
- Changing brand CSS? Three steps: brand, `BRAND_DIST_REF` in `tutor-indigo`, both pins.
