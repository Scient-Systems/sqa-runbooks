# 19 — Making a layout change in SQA Core (start to finish)

Someone asked you to move, resize, hide or add something on a page.

This page is the whole job. Start at Step 0. It tells you which of three paths you are on.
You do that one path and skip the other two.

---

## TL;DR

```
read the hostname → try it in DevTools → edit ONE repo → push → merge the PR
   → check staging → click Deploy production
```

- **About 2 hours.** ~75 min of that is the build. The edit itself takes minutes.
- **The hostname tells you which repo to edit.** Nothing else does.
- **You cannot preview locally.** Use DevTools, then staging.
- **Every change costs a full ~75 min build**, even one line of CSS. So **batch your tweaks**.

Note: editing the wrong repo is the number one time waster. It does not error. The build goes
green, the deploy works, the page just does not change. Do Step 0 first.

---

## Prerequisites

1. **Ensure you can open the site in a browser** and open DevTools (F12).
2. **Ensure you have both theme repos cloned**, on the right branch: `tutor-indigo` on `release`,
   `brand-openedx` on `ulmo/indigo`.
3. **Ensure you can push** to the `Scient-Systems` org.
4. **Ensure you can merge PRs in `openedx-deploy`.** That is where the deploy happens.
5. **Ensure you have Node 18+ and npm.** Only needed for `brand-openedx`.
6. **Ensure you have `kubectl` on the prod cluster.** Only needed for Step 2.3.
7. **A ticket that names the page and says what should change.** "Make it nicer" is not enough.
   Go get specifics.

---

## Step 0 — Find which surface your page is on

### Step 0.1 - Open the page and read the address bar

Note: read the real hostname. Not the page title, not the design mock.

### Step 0.2 - Look it up here

| Address bar | Surface | Repo to edit | Go to |
|---|---|---|---|
| `lms.stemquestacademy.com/…` | Legacy LMS page | `tutor-indigo` | **Step 2** |
| `apps.stemquestacademy.com/…` | MFE page | `brand-openedx` or `tutor-indigo` | **Step 3** |
| `apps.stemquestacademy.com/sqa-payment/…` | Payment MFE | `frontend-app-sqa-payment` | **Step 4** |
| `studio.stemquestacademy.com/…` | Studio | none, it is unthemed on purpose | **stop, ask first** |
| `stemquestacademy.com` (no subdomain) | Marketing site | `sqa-homepage` | runbook 13 |

Note: `lms.` pages are our own HTML and CSS. `apps.` pages are stock React apps we do not own.
Different code, different steps.

Note: if the page redirects, use where it **lands**. `lms.…/dashboard` sends you to
`apps.…/learner-dashboard`. That is an MFE page, so Step 3.

### Step 0.3 - Set variables and pull

```bash
export INDIGO=$HOME/tutor-indigo
export BRAND=$HOME/brand-openedx
export PAY=$HOME/frontend-app-sqa-payment

git -C $INDIGO checkout release     && git -C $INDIGO pull
git -C $BRAND  checkout ulmo/indigo && git -C $BRAND  pull
git -C $PAY    checkout main        && git -C $PAY    pull
```

Note: `brand-openedx` is not on `master`. It is on `ulmo/indigo`. Wrong branch means nothing ships.

---

## Step 1 — Try it in DevTools first

Note: do this on every path. It is your only fast feedback. After this, the next time you see your
change is on staging, about 75 minutes later.

### Step 1.1 - Inspect the element

Right-click the thing, click **Inspect**. In the **Styles** panel, edit the values until the page
looks right.

### Step 1.2 - Write down two things

1. The **selector** you targeted.
2. The **CSS lines** that fixed it.

Note: check it in light mode, dark mode, and at phone width (Ctrl-Shift-M). Most of our layout bugs
are a rule that only works at one screen size.

### Step 1.3 - Decide what kind of change it is

| What you did in DevTools | Kind | Go to |
|---|---|---|
| Changed size, spacing, colour, order, or hid it | **CSS** | Step 2.1 or Step 3.2 |
| Dragged it somewhere else, or it did not exist yet | **structure** | Step 2.3 or Step 3.3 |

Note: pick CSS if you can. CSS is one file. Structure on an MFE is three files that must agree, and
each mistake costs a full build to find.

---

## Step 2 — Path A: a legacy LMS page (`lms.…`)

### Step 2.1 - CSS: edit the SCSS file for that page

Paths are under `$INDIGO/tutorindigo/templates/indigo/lms/static/sass/`.

| Page | File |
|---|---|
| Home page | `home/_home.scss` |
| Header | `extra/_header.scss` |
| Footer | `extra/_footer.scss` |
| Dashboard | `dashboard/_dashboard.scss` |
| Course catalog (`/courses`) | `courseware/_discover.scss` |
| Course about page | `courseware/_about.scss` |
| Certificates | `certificates/_certificate.scss` |
| Wiki / Teams | `views/_wiki.scss` / `views/_teams.scss` |
| Colours, fonts, dark-mode variables | `partials/lms/theme/_variables.scss` |
| Anything else | `partials/lms/theme/_extras.scss` |

### Step 2.2 - Made a new file? Import it

```scss
# at the BOTTOM of partials/lms/theme/_extras.scss, with the others
@import '../../../<folder>/<name>';
```

Note: drop the underscore and the `.scss`. So `dashboard/_hero.scss` becomes
`'../../../dashboard/hero'`.

Note: a file you forget to import compiles to nothing. No error, anywhere. If your change does
nothing after a deploy, check this line first.

### Step 2.3 - Structure: copy the stock template first

```bash
kubectl -n openedx exec deploy/lms -- cat /openedx/edx-platform/lms/templates/<path>.html
```

Note: save it at the **same path** under `$INDIGO/tutorindigo/templates/indigo/lms/templates/`,
then edit your copy. The theme matches by path. A typo in the path means your file is never used.

Note: read it out of the running pod, not GitHub. The pod has the exact version production runs, so
there is no branch to guess.

Note: these pages are already overridden. Edit them where they are, do not re-copy them:
`index.html`, `index_overlay.html`, `course.html`, `footer.html`, `header/*.html`,
`courseware/courses.html`, `courseware/course_about.html`,
`learner_dashboard/programs_fragment.html`, `static_templates/*.html`.

### Step 2.4 - Dark mode

Note: dark rules are the same selector with `body.indigo-dark-theme` in front. Skip it and your
text goes unreadable at night.

### Step 2.5 - Do not break the templating

Note: `{{ … }}` and `{% … %}` in these files are Tutor config, not your code. Never add a new `{{`.
`${…}` is normal and fine in the `.html` files.

### Step 2.6 - Commit

```bash
cd $INDIGO
git add -A
git status --short          # every new file must show up here
git commit -m "layout: <page> — <what changed>"
```

Note: use `git add -A`. The build uses the committed files only. A new file left untracked has
broken a build before.

**→ go to Step 5.**

---

## Step 3 — Path B: an MFE page (`apps.…`)

Note: you cannot edit these pages. `learning`, `learner-dashboard`, `profile`, `account`,
`discussions` and `authn` are stock Open edX apps and we do not fork them. You change them from
outside, in one of two ways.

### Step 3.1 - Pick which one

| What you want | Repo | Go to |
|---|---|---|
| Restyle, move, resize or hide something already on the page | `brand-openedx` | **3.2** |
| Replace a block, or add something that is not there | `tutor-indigo` | **3.3** |

Note: 3.2 is one file and no React. Try to make your change fit it.

### Step 3.2 - CSS: edit the brand file

Paths are under `$BRAND/paragon/`.

| Page | File |
|---|---|
| Learner dashboard | `_dashboard.scss` |
| Courseware | `_learning.scss` |
| Profile | `_profile.scss` |
| Account settings | `_account.scss` |
| Discussions | `_discussion.scss` |
| Login / register | `_login.scss` |
| Progress tab | `_progress.scss` |
| Dates tab | `_dates.scss` |
| Header / footer (all MFEs) | `_header.scss` / `_footer.scss` |
| Colours and fonts | `tokens/src/themes/{light,dark}/*.json` |
| Anything else | `_overrides.scss` |

New file? Add `@import "./<name>";` to `_overrides.scss`, **above** the `@import "./dark";` line at
the bottom.

Now rebuild the CSS. This is not optional:

```bash
cd $BRAND
npm ci
make build
git add -A paragon dist
git status --short          # dist/*.css MUST show up here
git commit -m "layout: <page> — <what changed>"
```

Note: `dist/` is committed on purpose. Browsers load that compiled file from a CDN. No `dist/`
means users see nothing.

Note: never use `@media (--pgn-size-breakpoint-min-width-md)` or any other `--pgn-` media query in
a new rule. Browsers drop those blocks in production and your rule quietly dies. Use pixels:
`@media (min-width: 768px)`.

Note: `_dark.scss` imports these same files again inside `[data-paragon-theme-variant="dark"]`, so
each one compiles twice. A hardcoded hex colour applies to **both** themes. A
`var(--pgn-color-…)` changes per theme. Pick on purpose.

### Step 3.3 - Structure: put a widget in a slot

MFEs have named **slots**. You can hide what is in a slot, and put your own React widget there.
This needs **three files that agree**. Miss one and nothing renders and nothing errors.

```
1. $INDIGO/tutorindigo/components/MyWidget.jsx                    ← write it
2. $INDIGO/tutorindigo/patches/mfe-env-config-runtime-definitions ← define it
3. $INDIGO/tutorindigo/plugin.py                                  ← mount it
```

**File 1** — `components/MyWidget.jsx`:

```jsx
const boxStyle = { padding: 16, borderRadius: 12 };
const MyWidget = () => {
  return (<div style={boxStyle}>Hello</div>);
};
```

**File 2** — one line in `patches/mfe-env-config-runtime-definitions`, next to the others:

```
{{- patch("MyWidget.jsx") }}
```

**File 3** — in `plugin.py`, next to the other `PLUGIN_SLOTS.add_items([...])`:

```python
PLUGIN_SLOTS.add_item((
    "learner-dashboard",                                        # the MFE
    "org.openedx.frontend.learner_dashboard.course_list.v1",    # the slot
    """
    { op: PLUGIN_OPERATIONS.Hide,   widgetId: 'default_contents' },
    { op: PLUGIN_OPERATIONS.Insert,
      widget: { id: 'my_widget', type: DIRECT_PLUGIN, RenderWidget: MyWidget } },
    """,
))
```

Note: `Insert` on its own **adds** your widget next to what is already there. `Hide` plus `Insert`
**replaces** it. Delete the `Hide` line if you only want to add.

Slots we already use. Reuse one if it fits:

| MFE | Slot ID | Where it shows |
|---|---|---|
| learner-dashboard | `org.openedx.frontend.learner_dashboard.course_list.v1` | above the course list |
| learner-dashboard | `org.openedx.frontend.learner_dashboard.widget_sidebar.v1` | right sidebar card |
| learner-dashboard | `org.openedx.frontend.learner_dashboard.no_courses_view.v1` | the empty state |
| profile | `org.openedx.frontend.profile.additional_profile_fields.v1` | left side of the profile form |
| (all) | `org.openedx.frontend.layout.header_desktop_main_menu.v1` | desktop top nav |
| (all) | `org.openedx.frontend.layout.footer.v1` | footer. **avoid**, it has dedupe logic |

Note: need a different slot? Get the ID from `github.com/openedx/frontend-app-<name>`, branch
`release/ulmo.3`, in `src/plugin-slots/*/index.jsx`. Do not guess. A wrong ID renders nothing.

Four rules for the `.jsx`. All four have bitten us:

- **No `export`, no `import`.** The file gets pasted in as text. It is not a module.
- **No `{{` or `}}` anywhere.** Not in JSX, not in a string, not in a comment. It breaks the build.
  This is why the style object is a `const` instead of `style={{...}}`. `${…}` is fine.
- **Only use names already in `components/Imports.jsx`** (React, `useState`, `useEffect`,
  `getConfig`, `getAuthenticatedHttpClient`, `Icon`, `useIntl`). Add to that file if you need more.
  No new npm packages. They are not installed.
- **Keep it small.** One syntax error breaks every MFE, not just yours.

### Step 3.4 - Commit

```bash
cd $INDIGO
git add -A
git status --short          # the new .jsx must show up here
git commit -m "layout: <page> — <what changed>"
```

**→ go to Step 5.**

---

## Step 4 — Path C: the payment MFE (`apps.…/sqa-payment`)

Note: we own this one, so you edit the React directly. No slots, no brand package.

### Step 4.1 - Edit the page

Paths are under `$PAY/src/`.

| What | File |
|---|---|
| Frame shared by every page | `components/Layout.tsx` |
| Pricing / Checkout / Manage / Success | `membership/PricingPage.tsx`, `CheckoutPage.tsx`, `ManagePage.tsx`, `SuccessPage.tsx` |
| Pathways | `pathways/PathwaysPage.tsx`, `pathways/PathwayDetailPage.tsx` |
| Header and footer | `index.scss` |

Note: layout here is inline `style={{…}}` inside the components, not a stylesheet. Inline styles
beat stylesheet rules, so editing `index.scss` will lose. Edit the component.

### Step 4.2 - Commit

```bash
cd $PAY
git add -A && git commit -m "layout: <page> — <what changed>"
```

**→ go to Step 5.**

---

## Step 5 — Push, then merge the PR

Note: pushing does not deploy. Merging the **pin PR** starts the build.

### Step 5.1 - Push to the right branch

```bash
git -C $INDIGO push origin release        # tutor-indigo
git -C $BRAND  push origin ulmo/indigo    # brand-openedx
git -C $PAY    push origin main           # frontend-app-sqa-payment
```

### Step 5.2 - Wait for that repo's tests to go green

Note: red means stop. Nothing happens next until it is green.

### Step 5.3 - Merge the pin PR in `openedx-deploy`

A PR opens on its own, in `openedx-deploy`. Not in the repo you pushed to. It looks like:

```
pin: tutor-indigo 058e4f836a31 -> 9c1d40ab77e2
```

It changes one line in `versions.yml`. Merging it is what says "build and ship this".

### Step 5.4 - If you edited `brand-openedx`, merge TWO PRs

| PR opens in | What it changes |
|---|---|
| `openedx-deploy` | the `brand-openedx` pin |
| `tutor-indigo` | `BRAND_DIST_REF`, the CDN commit browsers actually load |

Note: merge both. Merging only the first ships the **old** CSS.

Note: merging the `tutor-indigo` one opens a third pin PR in `openedx-deploy`. Merge that too.

Note: if you skip it, staging fails its health check with `BRAND PIN DRIFT`. That check exists
because this used to go unnoticed. If you see that message, this step is why.

---

## Step 6 — Wait for the build, then check staging

### Step 6.1 - Watch the build

`openedx-deploy` → **Actions**.

Note: about **75 minutes**. Every change rebuilds both images, even one line of CSS. Do not cancel
it.

Note: the build opens the finished image and checks our code is really inside it. If not, it fails
and ships nothing. That is correct behaviour, not a flake.

### Step 6.2 - Staging deploys itself

Note: this is automatic, then it runs an 11-point health check. You do nothing.

### Step 6.3 - Look at your page on staging

| Surface | Staging URL |
|---|---|
| LMS pages | `https://staging.stemquestacademy.com` |
| MFE pages | `https://apps.staging.stemquestacademy.com` |
| Studio | `https://studio.staging.stemquestacademy.com` |

Note: hard-refresh with **Ctrl-Shift-R**. Brand CSS is cached at the CDN for up to 5 minutes.

Note: give the theme a few minutes after the deploy. An unstyled page in the first few minutes is
normal. Do not go debugging it.

Check all four:

- [ ] desktop
- [ ] phone width
- [ ] light mode
- [ ] dark mode

---

## Step 7 — Deploy to production

Note: staging is automatic. Production is a button you press.

### Step 7.1 - Get the tag

Note: it is whatever staging is running. It is printed in the build summary and the staging deploy
summary. It is 12 characters, like `b4e989eac176`.

### Step 7.2 - Run the workflow

1. `openedx-deploy` → **Actions**
2. In the **left sidebar**, click **"Deploy production"**
   *(it is not in the main run list. Look in the sidebar.)*
3. Click **Run workflow** on the right
4. Paste the tag and run

Three things happen on their own first:

- **It checks staging is already on that tag** and refuses if not.
- **It backs up the database and uploaded files** and copies them off the server. Missing backup
  means it stops before touching anything.
- **It runs the same 11-point health check** after.

---

## Step 8 — Check production

```bash
curl -s -o /dev/null -w 'lms %{http_code}\n' https://lms.stemquestacademy.com/heartbeat
curl -s -o /dev/null -w 'mfe %{http_code}\n' https://apps.stemquestacademy.com/
```

Then open the real page, hard-refresh, and check the same four boxes as Step 6.3.

Note: `200` on both plus the page looking right is the whole check.

Note: do not `curl | grep` an MFE page. It is drawn by JavaScript, so grep proves nothing. Use a
browser.

---

## Rolling back

Same clicks as Step 7.2, with the **previous tag**. Every tag we ever built is kept.

Note: roll back first. Investigate after.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Deployed, page looks exactly the same | wrong repo for that hostname | redo Step 0 |
| New SCSS file does nothing, no error | never imported | Step 2.2 or Step 3.2 |
| Staging health check says `BRAND PIN DRIFT` | did not merge the `tutor-indigo` PR | Step 5.4 |
| MFE CSS change shows up only sometimes | CDN cache, up to 5 min | wait, then Ctrl-Shift-R |
| Widget missing, console says `ReferenceError: MyWidget is not defined` | missing the definitions line | Step 3.3, file 2 |
| Widget missing, no console error at all | wrong slot ID | check it against `release/ulmo.3` |
| Build fails: `Template syntax error: Expected an expression` | a `{{` in your `.jsx`, often `style={{…}}` | move styles to a `const` |
| Build fails on a file you never touched | your `.jsx` broke the shared config | fix yours, the error names the wrong file |
| Build fails right after you added a file | never `git add`-ed | `git add -A`, push again |
| Responsive rule works for you, dead in prod | a `--pgn-` media query | use `@media (min-width: 768px)` |
| Text unreadable in dark mode only | no `body.indigo-dark-theme` rule | Step 2.4 |
| Payment MFE ignores your CSS | inline styles always win | edit the component |
| CI says *"job was not acquired by Runner"* | GitHub is broken, not you | check githubstatus.com, re-run |
| One line of CSS took 75 minutes | known, any pin change rebuilds both images | batch your tweaks |

---

## What this deliberately doesn't do

- **No local preview.** Your laptop runs the pip version of the theme, not our fork. DevTools, then
  staging. Never test by deploying to production.
- **No Studio changes.** Studio is left unthemed on purpose. If a ticket asks for it, ask first.
- **No forking an MFE.** If a layout cannot be done with a slot or CSS, that is a design
  conversation, not a bigger diff.
- **No feature flags.** Layout goes live to everyone the moment production deploys. There is
  nothing to hide it behind.

---

## What's next

- The pipeline in more detail → **17-shipping-a-change.md**
- Slot widgets in more detail → **09-mfe-plugin-slots-react-components.md**
- Which repo styles what → **10-theming-lms-and-mfes.md**
- When something breaks → **15-troubleshooting-and-gotchas.md**
