# C12 — War stories: how we actually did what we did

> Real incidents from this platform, written as case studies. Each one: what we saw, how
> we hunted it, the root cause, what we did, **what else we could have done and the
> trade-offs**, and the transferable lesson. Read these like an accident report board —
> this is the fastest way to inherit our experience.

---

## 1. The white-on-white dropdowns (dark mode, June–July 2026)

**Symptom.** In dark mode, every `<select>` dropdown across the site (account page,
profile, registration) opened as a white panel with near-invisible white text. Only the
browser-highlighted row was readable.

**Investigation.** First instinct — "the dropdown CSS is wrong" — was itself wrong in an
instructive way: our CSS for the *closed* select box was fine (dark background, light
text). The broken part was the *popup list*, which is a **native browser widget**:
the browser paints it, not our stylesheet. It painted light-mode colors (white) while the
option text inherited our light theme text color. White on white.

**Root cause.** The dark theme never declared `color-scheme: dark`. That one property is
how a page tells the browser "paint your native widgets (select popups, checkboxes,
scrollbars, date pickers) in dark". Without it we'd spent weeks patching *symptoms* —
`accent-color` for checkboxes, `scrollbar-color`, autofill colors — never the cause.

**Fix.** `color-scheme: dark` in both dark themes (brand-openedx for MFEs, tutor-indigo
for legacy pages) + an explicit `option { background; color }` fallback. Bonus finds
during the audit: two `color: dark;` declarations — invalid CSS (missing `$` on the Sass
variable), silently dropped by browsers for months; and a stack of hardcoded light-mode
hex values that only worked in dark mode because a later override happened to win.

**Alternatives & trade-offs.** Keep symptom-patching every widget (endless, and select
popups *can't* be fully styled — dead end); restyle selects as custom React dropdowns
(controllable, but heavy and fights Paragon). The one-line root fix was strictly better —
the lesson is that we couldn't *see* it while thinking in terms of "styling elements"
instead of "who paints this pixel".

**Lesson.** When a fix list keeps growing (accent-color, scrollbars, autofill, …), stop
and ask what single missing concept generates the whole list. Symptom clusters have a
common parent more often than not.

---

## 2. The deploy that said yes but did nothing (the triple-fault day, July 2026)

The crown jewel — three independent faults stacked, each hiding the next. What follows is
the *actual sequence*.

**Symptom 1.** The dropdown fix (story #1) was pushed, `tutor config save` +
`tutor k8s start` + `rollout restart` all ran green — but production still served the old
CSS URL. Deploy said yes; reality said no.

**Hunt 1.** Walking the deploy chain (→ C11 §2): repo ✓, VPS clone ✓, rendered env ✓, new
ConfigMap created ✓ ("`openedx-settings-lms-…` **created**" in the apply output proved
the render). So the config *reached the cluster* — meaning the break was at the pod hop:
`kubectl describe pod` → `FailedScheduling: 0/1 nodes available: Insufficient memory`.

**Root cause 1.** Default rollouts **surge**: start the new ~2 GB LMS pod *while the old
one runs*, then swap. Our 8 GB single node cannot hold two LMS pods. The new pod sat
`Pending` forever; the old pod — old config — kept serving. **A rollout that never
completes is indistinguishable from a successful deploy unless you check
`rollout status`.** (Concepts: → C07 §4.)

**Fix 1.** `maxSurge: 0, maxUnavailable: 1` (replace-in-place) on lms/cms/workers,
proposed in `K8S_ROLLOUT_MEMORY_FIX.md` first, then applied. Trade-off accepted
knowingly: ~2 min downtime per deploy, in exchange for deploys that finish.

**Symptom 2.** The moment replace-in-place killed the old pods… the new ones went
`ImagePullBackOff`. Site fully down. (This felt like the fix broke everything. It
didn't — it *unmasked* fault #2, which surge had been hiding by never killing old pods.)

**Hunt 2 & root cause 2.** The node needed to re-pull the (private) image from GHCR and
was refused: the cluster's `ghcr-pull` secret held an **expired PAT**. It had been dead
for weeks — invisible, because the node's local image cache satisfied every pod start
until now. (Concepts: → C05 §4, C09 §3. The same expired PAT had already broken image
*builds* earlier that day — same root cause, different world-3 consumer.)

**Fix 2.** New PAT → overwrite the secret (`--dry-run=client -o yaml | kubectl apply`)
→ delete stuck pods → node pulls → pods Running → walked the chain again from the top:
config API now served the new SHA → hard refresh → dropdowns dark. Done.

**Alternatives we weighed.** For fault 1: 16 GB VPS (the *real* fix — standing decision:
do it when budget allows); shrink the stack (fewer uwsgi workers — viable trim);
`kubectl patch` per-incident (transient, k8s start reverts it — rejected as permanent
answer). For fault 2: make the GHCR package public (instant fix, but publishes our plugin
code — rejected); credential with no expiry (fails differently later); calendar-tracked
PAT rotation (chosen).

**Lessons (each now lives in a concepts doc).** `apply` ≠ converge — always
`rollout status` (C07). Cached images are loans (C05). Credentials fail at *use* time,
weeks after they die (C09). Stacked faults peel one at a time — fixing #1 exposing #2 is
progress, not regression (C11 §5).

---

## 3. The webhook that ate payments silently (June 2026)

**Symptom.** A test purchase completed on Stripe, no membership appeared, no error
anywhere. Retrying the webhook from Stripe's dashboard: still nothing.

**Root cause.** Our idempotency guard recorded the event as `processed=True` **even when
the handler threw** — "seen" and "succeeded" were one flag. First delivery failed mid-way
(a handler bug), got recorded as done, and every replay was politely skipped forever.
The outer 200 response also told Stripe never to retry. A perfect silence machine.

**Fix.** Fix the handler bug; separate seen-vs-succeeded semantics; operational unstick =
delete the `StripeEvent` row, replay from the Stripe dashboard.

**Alternatives & trade-offs.** Return 500 on handler errors so Stripe retries
(simple, but retries a deterministic bug pointlessly and pages Stripe's dashboard red);
a dead-letter status + alert (the grown-up version, worth doing at scale).

**Lesson.** Idempotency keys must mark **success, not attempts** — and any "skip if seen"
logic is a silence machine unless failures are made loud somewhere else. (→ C10 §3.)

---

## 4. The stylesheet the browser threw away without telling anyone (June 2026)

**Symptom.** Runtime theme CSS (pipeline B) simply didn't apply. No console error, no
404 — the URL returned the right bytes when opened by hand.

**Hunt.** Devtools → Network tab → the CSS response: status 200, body correct…
**Content-Type: `text/plain`** + `X-Content-Type-Options: nosniff`. Browsers refuse to
apply a stylesheet not declared `text/css` — *silently*. MFEs fell back to baked-in CSS,
so pages looked plausibly styled, just stale.

**Root cause.** We served the CSS from `raw.githubusercontent.com`, which is a
file-viewer, not a CDN — it deliberately serves text/plain.

**Fix.** jsDelivr (`cdn.jsdelivr.net/gh/...@<commit-sha>/dist/…`), which serves real
`text/css` — with commit-SHA pinning both because branch-with-slash breaks jsDelivr's
syntax *and* because immutable URLs make CDN caching harmless (→ C08 §4).

**Lesson.** A 200 with the right body can still be a failure — **headers are part of the
contract**. And "no error in console" bounds nothing; check what the browser *did*, not
what it said.

---

## 5. The 40-minute build that shipped last month's CSS (June 2026)

**Symptom.** Brand CSS changes pushed, mfe image rebuilt (40 minutes, success), deployed —
MFEs styled with the *old* brand.

**Root cause.** The Dockerfile line `RUN npm install '@edx/brand@github:…#ulmo/indigo'`
never changes as *text*, so Docker's layer cache reused the months-old layer. The branch
ref pointed somewhere new; the cache didn't care (→ C05 §2).

**Fix & follow-up.** Immediate: `--no-cache`. Structural (the interesting part): we
migrated as much styling as possible out of the baked layer entirely, into the runtime
`PARAGON_THEME_URLS` layer — deliberately converting code into config (→ C01 §3, C08 §3)
so most theme iterations never touch Docker again.

**Alternatives.** Pin the npm install to a commit SHA and bump it each change (correct,
cache-friendly, but a manual bump someone forgets); always `--no-cache` (correct, always
40 minutes). The runtime-layer migration dominated both.

**Lesson.** When a tool's cache key is *the command text*, network-fetching commands are
landmines. And the deeper play: don't get better at rebuilding — arrange to rebuild less.

---

## 6. The responsive rules that compiled fine and did nothing (June 2026)

**Symptom.** Responsive breakpoint styles in a new brand partial worked in local builds,
were dead in production MFEs. No errors at any stage.

**Root cause.** The partials use Paragon's *custom media queries*
(`@media (--pgn-size-breakpoint-min-width-md)`), which are a **build-time feature** — a
preprocessor rewrites them into real media queries. Our runtime pipeline B ships some CSS
**raw**, without that build step; browsers meet `(--pgn-…)`, understand nothing, and
drop the whole block. Silently. (Same genre: the legacy-LMS `calc(50% - 50vw)` full-bleed
trick that stock `width:100%` containers break — see project memory.)

**Fix.** Plain pixel queries (`@media (min-width: 768px)`) in anything that can ship raw.

**Lesson.** Know **which pipeline processes which file** before using pipeline-specific
syntax. "It compiles" only vouches for the pipeline it compiled *in*.

---

## The meta-lessons (if you only remember five things)

1. **Green output is not deployed reality.** Verify convergence: `rollout status`, curl
   the live thing, look at what the browser fetched.
2. **Caches lie kindly.** Docker layers, node images, Redis config, CDN, browser — know
   all five (→ C11 §3).
3. **Silent failure is designed-in** at several layers (nosniff CSS drops, dropped media
   queries, idempotency skips). Absence of errors ≠ presence of function.
4. **Root causes generate symptom families.** A growing patch list means you're
   downstream of the real bug.
5. **Trade-offs beat heroics.** Every fix above had a cheaper-worse and pricier-better
   sibling; the job is choosing consciously and writing the choice down — which is what
   this folder is.

**Next:** [C13 — state, data & backups](13-state-and-data.md) — the one domain where
none of these recovery moves work.
