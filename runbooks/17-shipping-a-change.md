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
| `tutor-indigo` | The theme. Our fork | `release` |
| `brand-openedx` | Brand CSS: colours, logos, fonts | `ulmo/indigo` |

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

You do **two** things by hand in this whole flow: **merge a PR**, and **click deploy**.
Everything else happens on its own.

```
  you push to a code repo
        │
        ▼
  its own tests run          ← in that repo. red = stop, nothing else happens
        │ green
        ▼
  a PR opens in openedx-deploy   ← AUTOMATIC. bumps your repo's pin in versions.yml
        │
        ▼  ← YOU MERGE IT   (human step 1)
        │
  images build in the cloud      ← AUTOMATIC. ~19 min, or ~73 min if the MFE changed
  and are checked before publish
        │
        ▼
  staging deploys itself         ← AUTOMATIC, then runs an 11-point health check
        │
        ▼  ← YOU LOOK AT STAGING, then CLICK DEPLOY   (human step 2)
        │
  production                     ← backs up first, refuses anything staging hasn't run
```

### The PR you will see

It appears in **`openedx-deploy`**, not in the repo you pushed to. It is titled something like:

```
pin: sqa_django_app 2f8fad270a46 -> a91c04ff21be
```

All it does is change one line in `versions.yml`. Merging it is what says *"this change is
ready to be built and shipped."* Until you merge, nothing is built.

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
| Baked into the MFE image at build time | follows the `ulmo/indigo` branch |
| Served live to browsers from a CDN | pinned to a **specific commit** by `BRAND_DIST_REF`, a line inside **`tutor-indigo`** |

The second one is what actually puts CSS in front of users. It lives in a *different repo*
from the one you edited.

### What you do

1. Edit the SCSS in `brand-openedx`
2. Run `npm run build` — this regenerates `dist/`
3. **Commit `dist/` as well as your source.** Its CI will fail if you forget
4. Push to `ulmo/indigo`

### What happens then

**Two** PRs open automatically:

- one in `openedx-deploy`, bumping the `brand-openedx` pin
- one in **`tutor-indigo`**, bumping `BRAND_DIST_REF`

Merge **both**. Then carry on with the normal flow.

> ⚠️ **If only one PR appears, stop and ask.** Skipping the `tutor-indigo` one fails silently
> and completely: every check goes green, the build passes, staging deploys, and the site
> keeps serving the **old** CSS while you wonder why your change did not appear. The health
> check now catches this, but it is much easier to just merge both PRs.

---

## 4. Deploying to production

Staging is automatic. Production is a deliberate human action.

1. Go to **`openedx-deploy` → Actions tab**
2. In the **left sidebar**, click **"Deploy production"**
   *(it will not appear in the main run list until it has run at least once — look in the sidebar)*
3. Click **"Run workflow"** on the right
4. Paste the **tag** and run

**Where to get the tag:** it is whatever staging is currently running. It is printed in the
build's summary and in the staging deploy's summary. It looks like `b4e989eac176`.

Three things happen automatically before your change reaches users:

- **It checks staging is already running that exact tag.** If not, it refuses. You cannot
  promote something that has not been validated.
- **It takes a full backup** of the database and all uploaded files, and copies it off the
  server. If any part of the backup is missing, it stops before touching anything.
- **It runs the same 11-point health check** afterwards.

### Rolling back

Same three clicks, with the **previous tag**. Every tag we have ever built is kept, so you can
always go back. Roll back first, investigate afterwards.

---

## 5. Things that will confuse you the first time

**"My PR opened in the wrong repo."** It did not. Pin PRs always open in `openedx-deploy`,
because that is where the version of everything is recorded.

**"I merged my code but nothing deployed."** Merging in the *code* repo does not deploy.
Merging the *pin PR* in `openedx-deploy` is what starts a build.

**"The theme looks wrong right after a deploy."** Give it a few minutes. The themed page takes
minutes to appear after any deploy, and the plain-looking page is served in the meantime. This
is normal and known. Do not go debugging it in the first ten minutes.

**"CI failed but the error is nothing to do with my code."** If you see *"job was not acquired
by Runner"* or *"Internal server error"*, that is GitHub's infrastructure, not you. The job
never ran. Check <https://githubstatus.com>, then re-run it.

**"A tiny CSS change took over an hour."** Known and annoying. Any change to `versions.yml`
rebuilds the MFE image, which takes about 73 minutes, even when the change did not need a
rebuild at all. There is a plan to add a fast path for config-only changes. Until then, batch
your brand tweaks rather than shipping them one at a time.

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

- Push to your repo. Tests run.
- **Merge the PR that appears in `openedx-deploy`.**
- Images build and staging deploys itself. Go look at staging.
- **Click "Deploy production" and paste the tag.**
- Changing brand CSS? Expect **two** PRs and merge both.
