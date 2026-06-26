# 08 — MFEs: editing existing ones & adding a new one

MFEs (Micro-Frontends) are the React single-page apps that have replaced many old server-rendered
LMS pages. This runbook covers what they are, how to develop against one with hot reload, how to add
a brand-new MFE (like our `frontend-app-sqa-payment`), and the production gotchas that produce a
blank white page.

For injecting a React component into *someone else's* MFE without forking it, see runbook 09.
For theming MFEs, runbook 10.

---

## 1. The MFE landscape on this project

| MFE | Repo | Served at (dev) | Notes |
|---|---|---|---|
| learner-dashboard | openedx/frontend-app-learner-dashboard | `:1996/learner-dashboard` | our "Flight Deck" widgets inject here |
| profile | openedx/frontend-app-profile | `:<port>/profile` | SqaTokenCard injects here |
| account, learning, discussions, authn | openedx/frontend-app-* | various | stock + brand theme |
| **sqa-payment** | **Scient-Systems/frontend-app-sqa-payment** | `:2003/sqa-payment` | **ours** — pricing/checkout/manage |

> Dev ports live in the built-in `mfe` plugin's published range (1984, 1993–2002, 2025). Confirm a
> given MFE's port with `tutor config printvalue MFE_HOST` + the MFE's entry in `tutor mounts list`,
> or just open the learner dashboard and follow links. `learner-dashboard` is `:1996`; our custom
> `sqa-payment` is `:2003` (chosen to sit outside that range).

All the stock MFEs are run by Tutor's official **`mfe`** plugin. Custom ones are registered by our
own Tutor plugin (`sqa_payment.py`). We're on MFE release branch **`release/ulmo.3`**.

🚨 **One container, many MFEs.** In dev each MFE *can* run its own webpack dev server, but in
production all MFEs are compiled to static bundles and served by **one `mfe` container** behind Caddy.
So `tutor dev restart learner-dashboard` is "no such service" — the service is named `mfe`;
`learner-dashboard` is just a static bundle inside it.

## 2. Develop against an existing MFE (hot reload)

```bash
# 1. Mount your local clone so the dev server hot-reloads on save.
#    🚨 The folder name MUST match the MFE's repo name exactly for the bind to work.
tutor mounts add ~/frontend-app-sqa-payment
tutor mounts list                       # confirm it shows ↔ sqa-payment

# 2. Start that MFE's dev server (runs webpack in Docker):
tutor dev start -d sqa-payment

# 3. Browse:
#    http://apps.local.openedx.io:2003/sqa-payment
```
Edits to `src/**` reload automatically. The generated `env.config.jsx` is mounted in automatically —
no manual copy.

For an MFE you don't have a custom registration for, mount it and start the matching service name
(e.g. `tutor dev start -d profile`). If you want to point an MFE at a different code location without
a full Tutor mount, MFEs also support a local `module.config.js` (copy the
`module.config.js.example`) for swapping in local copies of shared libraries — but for our work the
`tutor mounts` flow above is the standard.

## 3. Add a brand-new MFE (the `frontend-app-sqa-payment` recipe)

### 3a. Scaffold the repo
Open edX's MFEs are all built from one template. Create a new one from
`openedx/frontend-template-application` (the canonical MFE template) — fork/clone it, rename, and
build your pages under `src/`. Keep the standard `@edx/frontend-platform` bootstrap (`initialize()`,
`AppProvider`).

### 3b. Register it with Tutor (a Tutor plugin — runbook 07 §4)
```python
from tutormfe.hooks import MFE_APPS

@MFE_APPS.add()
def _add_my_mfe(mfes):
    mfes["my-app"] = {
        "repository": "https://github.com/Scient-Systems/frontend-app-my-app.git",
        "port": 2004,            # 🚨 pick a port OUTSIDE the built-in range (1984,1993-2002,2025)
        "version": "main",
    }
    return mfes
```
- In `tutor dev`, the **local mounted source** is used; the `repository` URL is only cloned for
  production image builds.
- Assets serve at `http(s)://{MFE_HOST}/my-app`.
- `tutor config save` then `tutor dev start -d my-app`.

### 3c. 🚨 Two production gotchas that make the page render blank (we hit BOTH)

These don't show up in dev — only after the production build. Configure them up front:

1. **React Router `basename`.** `AppProvider` auto-derives `basename` from
   `getConfig().PUBLIC_PATH`. In a prod build, `PUBLIC_PATH` is often unset → `basename='/'` → none
   of your routes match `/my-app/...` → header renders, body is blank, no JS error. **Fix:** in
   `src/index.tsx` pass `wrapWithRouter={false}` to `AppProvider` and wrap with your own
   `<BrowserRouter basename="/my-app">`. Also set `PUBLIC_PATH='/my-app/'` in `.env`.
   ```tsx
   <AppProvider wrapWithRouter={false}>
     <BrowserRouter basename="/my-app"> … </BrowserRouter>
   </AppProvider>
   ```
2. **`@edx/frontend-component-footer` kills `initialize()`.** It calls `ensureConfig()` for
   `SUPPORT_EMAIL`, `TERMS_OF_SERVICE_URL`, `PRIVACY_POLICY_URL`, `ENABLE_ACCESSIBILITY_PAGE`. The
   *dev* LMS Config API returns these; the *prod* one may not → `APP_INIT_ERROR` → blank page. **Fix:**
   either don't import that footer in a custom MFE, or configure all four keys in LMS production
   settings.

## 4. Add a nav link to your MFE from elsewhere

To put a link to your MFE in the learner-dashboard header, inject a `MembershipLink`-style component
into the header slot — exactly the `sqa_payment.py` pattern (runbook 07 §4 / runbook 09). The URL
comes from a Tutor config key so it differs per environment.

## 5. Deploying an MFE change (build-time — always)

MFE source, `env.config.jsx`, slot widgets, and brand SCSS baked into the bundle are **all
build-time**. There is no runtime shortcut (except brand *token* CSS — runbook 10). Pipeline:

```bash
# 🚨 commit + push FIRST — the mfe image clones from GitHub, local files are ignored
git add -A && git commit && git push          # in the MFE repo (user does this)

tutor config save                              # if a Tutor plugin / slot changed
tutor images build mfe                         # LONG (15–40 min) — compiles ALL MFEs
docker run --rm $(tutor config printvalue MFE_DOCKER_IMAGE) ls /openedx/dist/ | grep my-app  # ✅
tutor images push mfe
# then roll out — runbook 12 §MFE
```
🚨 Set `MFE_DOCKER_IMAGE` to your own registry **before** the first build, or push is rejected
(runbook 12). Remember indigo appends `-indigo` to the image tag (runbook 11).

---

## What's next

- Inject a component into an MFE without forking it → **09-mfe-plugin-slots-react-components.md**
- Theme the MFEs → **10-theming-lms-and-mfes.md**
- Build/push/deploy the mfe image → **12-production-deployment-k8s.md**
