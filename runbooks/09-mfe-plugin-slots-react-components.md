# 09 — MFE plugin slots: injecting React components

How to inject a **custom React component** into a stock Open edX MFE (learner-dashboard, profile,
account…) **without forking the MFE** — via our `tutor-indigo` plugin. This is how the dashboard
"Flight Deck" widgets and the profile SqaTokenCard got there.

> **The one-line lesson:** a custom widget needs **THREE wiring points in three different files**,
> and they must agree. Miss any one and the build either errors or silently renders nothing. We've
> been burned by each of the three. This runbook is the checklist that prevents it.

This is the *component-injection* counterpart to runbook 10 (theming/CSS). Read it before adding any
slot widget.

---

## 1. Why three files (the mental model)

`tutor-indigo` does not ship a finished `env.config.jsx`. It **assembles** it at `tutor config save`
time from patches. Your `.jsx` component file is **raw-concatenated as text** into the generated
`env.config.jsx` — it is NOT imported, NOT a module. Consequences:

- It must be **self-contained**: `const MyWidget = () => {...};` — **no `export`, no top-level
  `import`.** Shared imports come from `Imports.jsx` only.
- It is rendered **through Jinja first** (it's a Tutor template patch). Any `{{ … }}` in the file —
  **even inside a `//` comment** — is parsed as a Jinja print statement and breaks the build.
- `PLUGIN_SLOTS` in `plugin.py` only emits the `RenderWidget: MyWidget` **reference**. The
  *definition* must be placed in scope by a *separate* patch line.

```
components/MyWidget.jsx ──(glob auto-registers as ENV_PATCH "MyWidget.jsx")──┐
patches/mfe-env-config-runtime-definitions  {{ patch("MyWidget.jsx") }} ─────┘ ← DEFINES it
plugin.py  PLUGIN_SLOTS.add_item((mfe, slot_id, "...RenderWidget: MyWidget")) ← REFERENCES it
```
Reference without definition → `ReferenceError: MyWidget is not defined` in the browser console; page
renders without your widget.

## 2. The exact three steps (do ALL three)

### Step 1 — write the component
`tutorindigo/components/MyWidget.jsx`:
```jsx
const cardStyle = { padding: 16, borderRadius: 12 };   // hoist styles — see gotcha #1
const MyWidget = () => {
  const config = getConfig();
  const [data, setData] = useState(null);
  useEffect(() => {
    let alive = true;
    getAuthenticatedHttpClient()
      .get(`${config.LMS_BASE_URL}/sqa/api/.../`)   // ${} is JS; Jinja ignores single braces — OK
      .then((res) => { if (alive) setData(res.data); })
      .catch(() => {});
    return () => { alive = false; };
  }, []);
  return (<div style={cardStyle}>...</div>);
};
```
Rules: no `export`, no `import`, self-contained. Only identifiers from `Imports.jsx` (see §3).
**No `{{` or `}}` anywhere** — not in JSX, not in comments, not in strings. Hoist style objects to a
`const` (inline `style={{...}}` is a double-brace = Jinja tag). `${...}` template literals are fine.
Keep it small — a syntax error in ONE component breaks EVERY MFE's `env.config.jsx` build.

### Step 2 — register the definition (THE STEP PEOPLE FORGET)
Add ONE line to **`tutorindigo/patches/mfe-env-config-runtime-definitions`**:
```
{{- patch("MyWidget.jsx") }}
```
Put it next to the existing widgets (IndigoFooter, SqaDashboardHero, …). This is the line that
actually defines your component in the generated config.

> 🚨 Do **not** put it in `mfe-env-config-buildtime-imports`. That patch is for top-level ES `import`
> statements only; a component placed there lands in the wrong scope and still throws `ReferenceError`.

### Step 3 — mount it in a slot
In `tutorindigo/plugin.py`, add a `PLUGIN_SLOTS` entry:
```python
PLUGIN_SLOTS.add_item((
    "profile",                                                    # MFE app id
    "org.openedx.frontend.profile.additional_profile_fields.v1",  # slot id (verify — §4)
    """
    { op: PLUGIN_OPERATIONS.Insert,
      widget: { id: 'my_widget', type: DIRECT_PLUGIN, RenderWidget: MyWidget } },
    """,
))
```
`PLUGIN_OPERATIONS.Hide` widgetId `'default_contents'` + `Insert` = full replacement of a slot's
stock content. `Insert` alone = add alongside it.

## 3. What you can import (`Imports.jsx`)

`tutorindigo/components/Imports.jsx` is concatenated above all components. It currently provides:
`React, { useEffect, useState }`; `Cookies`; `getConfig` (`@edx/frontend-platform`);
`getAuthenticatedHttpClient, getAuthenticatedUser` (`@edx/frontend-platform/auth` — use these for
same-origin authenticated calls to the sqa plugin API); `Icon` + `Nightlight, WbSunny`
(`@openedx/paragon`); `useIntl` (`@edx/frontend-platform/i18n`).

Need something else? Add it to `Imports.jsx` — but **only packages the MFE already bundles**
(`@edx/frontend-platform`, `@openedx/paragon`, `react`). 🚨 **No new npm deps** — they won't be
installed in the MFE image.

## 4. Finding the right slot ID

Slot IDs differ per MFE and per release. **Verify against the deployed release branch
(`release/ulmo.3`)** — don't assume. Source: `github.com/openedx/frontend-app-<name>` at that branch,
in `src/plugin-slots/*/index.jsx` (the `id` prop on each `PluginSlot`). Or the
docs.openedx.org "frontend plugin slots" reference.

Slots we've used:
| MFE | Slot ID | Renders |
|---|---|---|
| profile | `org.openedx.frontend.profile.additional_profile_fields.v1` | left column of profile form |
| learner-dashboard | `org.openedx.frontend.learner_dashboard.course_list.v1` | above the course list |
| learner-dashboard | `org.openedx.frontend.learner_dashboard.widget_sidebar.v1` | right sidebar card |
| learner-dashboard | `org.openedx.frontend.learner_dashboard.no_courses_view.v1` | empty state |
| (all) | `org.openedx.frontend.layout.header_desktop_main_menu.v1` | desktop top nav |
| (all) | `org.openedx.frontend.layout.footer.v1` | footer — already filled by indigo_footer; avoid (dedupe, §6) |

## 5. Deploy (component/slot change = MFE image rebuild)

There is no runtime shortcut for components (unlike brand token CSS). The VPS runs `tutor-indigo` as
an **editable pipx inject**, so `git pull` + `tutor config save` picks up fork changes — no reinstall.
```bash
# user commits + pushes tutor-indigo first
cd /home/edx/tutor-indigo && git pull
tutor config save           # regenerates env.config.jsx from patches + plugin.py
tutor images build mfe      # LONG — don't cancel midway
tutor images push mfe
kubectl -n openedx rollout restart deployment/mfe   # (+ imagePullPolicy patch, runbook 12)
```

## 6. Gotchas (every one has bitten us — 2026-06-12 and 2026-06-25)

1. **`{{ }}` anywhere = build error.** Jinja renders the `.jsx` before it's JS. A doubled brace —
   inline `style={{}}`, a comment, a string — throws `Template syntax error: Expected an expression`.
   Hoist style objects to a `const`; never write `{{`.
2. **Missing the runtime-definitions line = `ReferenceError` at runtime.** `PLUGIN_SLOTS` only
   references; step 2 defines.
3. **Wrong patch file = also `ReferenceError`.** `buildtime-imports` is for ES imports only;
   definitions go in `runtime-definitions`.
4. **Local `tutor config save` does NOT validate the fork.** Local WSL runs the pip-installed
   `tutor-indigo` 21.1.3, not your fork — a local save won't reflect or error on fork changes. Real
   validation happens on the VPS. Before pushing: mentally verify the 3 wiring points + scan the
   `.jsx` for `{{`. Don't "test by deploying to prod."
5. **One bad `.jsx` breaks ALL MFEs' builds** (raw concatenation). Keep components small and tidy.
6. **Footer slot has dedupe logic** — `mfe-env-config-runtime-final` drops `indigo_footer` if >2
   inserts land in the footer slot. Prefer any other slot.
7. **New files under `tutorindigo/` must be `git add`-ed** — the VPS builds from the committed tree;
   an untracked file once broke the image build.

## 7. Pre-push checklist
- [ ] `components/MyWidget.jsx`: no `export`, no `import`, no `{{`/`}}` anywhere
- [ ] every identifier used is in `Imports.jsx` (added there if new — no new npm deps)
- [ ] `{{- patch("MyWidget.jsx") }}` added to **`mfe-env-config-runtime-definitions`**
- [ ] `PLUGIN_SLOTS.add_item((mfe, slot_id, "...RenderWidget: MyWidget"))` in `plugin.py`
- [ ] slot ID verified against the `release/ulmo.3` MFE source
- [ ] new files `git add`-ed
- [ ] after deploy, verify in the pod:
      `kubectl -n openedx exec deploy/mfe -- sh -c 'find / -name "env.config.js*" 2>/dev/null'`
      then grep that file for `const MyWidget` — present = wired correctly.

---

## What's next

- Theme the LMS + MFEs (CSS/tokens) → **10-theming-lms-and-mfes.md**
- The indigo plugin itself → **11-tutor-indigo-plugin.md**
