# Concepts — understand the platform before you operate it

The numbered runbooks (00–15) tell you **what to type**. This section tells you **what is
actually going on** — so that when a runbook says "rebuild the mfe image and rollout", you
know what an image is, what a rollout is, why this change needs one and that change doesn't,
and what to do when it breaks in a way no runbook predicted.

**The promise:** read these in order and you will not be lost. Each doc starts from zero,
uses plain language, builds the concept first, then shows how *our* platform uses it, then
tells you what we chose, what else we could have chosen, and what the trade-offs were.

## How to read this section

- **Don't skim all of it in one sitting.** Read 01 and 02 today. Read the rest when the
  topic becomes relevant to what you're doing — each doc says which runbooks it unlocks.
- Every doc ends with a **"You're ready when…"** list. If you can answer those questions
  without looking, move on. If not, re-read — the runbooks will assume it.
- 🔗 Cross-references: "→ runbook 07" means the operational doc one level up.
  "→ concepts/05" means another doc in this folder.

## The learning path

| # | Doc | One-line promise | Unlocks runbooks |
|---|---|---|---|
| 01 | [The big picture](01-big-picture.md) | Every moving part on one page, top-down | 00 |
| 02 | [A request's journey](02-request-journey.md) | What happens between typing the URL and seeing pixels | all |
| 03 | [Django & the plugin idea](03-django-and-plugins.md) | How our code gets *inside* Open edX, and what signals are | 06 |
| 04 | [Open edX anatomy](04-openedx-anatomy.md) | LMS vs CMS vs MFEs, and why the platform is two generations at once | 05, 06, 08, 09 |
| 05 | [Containers, images, registries](05-containers-and-images.md) | What Docker actually does, why builds cache, why pulls need auth | 12 |
| 06 | [The Tutor mental model](06-tutor-mental-model.md) | config → rendered files → images → running platform | 02, 03, 07 |
| 07 | [Kubernetes from zero](07-kubernetes-from-zero.md) | Pods, deployments, secrets, rollouts — and how to *inspect* anything | 12 |
| 08 | [Theming concepts](08-theming-concepts.md) | Design tokens, the two delivery layers, dark mode, CDNs | 10, 11 |
| 09 | [Auth & secrets](09-auth-and-secrets.md) | The three unrelated things people call "login", and where every secret lives | 12, 13, 14 |
| 10 | [Payments concepts](10-payments-concepts.md) | Checkout sessions, webhooks, idempotency — why Stripe calls *us* | 14 |
| 11 | [The debugging method](11-debugging-method.md) | The one skill that replaces panic: find the first broken hop | 15 |
| 12 | [War stories](12-war-stories.md) | Real incidents, how we diagnosed them, what else we could have done | — |
| 13 | [State, data & backups](13-state-and-data.md) | The one domain a redeploy can't fix — migrations, rollbacks, backups | 12 |
| 14 | [Anatomy of a feature](14-anatomy-of-a-feature.md) | How Membership 1.0 was really built — the repeatable 10-step method | 06 |

## Practice missions — reading is not capability

Concepts stick when you *use* them. These are safe (read-only or dev-only) missions,
in rising order. Do one after its matching doc; each should take under an hour.

1. **(after C01/C02)** Open the live site with browser devtools → Network tab. Load a
   legacy page and an MFE page. Find: the `mfe_config` call, the jsDelivr CSS, the JWT
   cookie. Narrate each hop out loud.
2. **(after C07)** On the VPS, read-only: `kubectl -n openedx get pods`, then
   `describe` one healthy pod and read its Events; find which ConfigMaps the lms pod
   mounts and locate the content-hash in the name. Run the "Allocated resources" check
   and say how much scheduling headroom the node has.
3. **(after C06)** On your dev machine: change a trivial value in a Tutor plugin, prove
   the three stages: `tutor config save`, grep it in `env/`, see it live in `tutor dev`.
4. **(after C03/C14)** Tier-1 only: write one new pytest against
   `check_enrollment_eligibility` for a case you *think* is covered. If it was, you
   learned the rule; if it wasn't, you just contributed.
5. **(after C08)** Trace one CSS token: pick a color on a live MFE page in devtools,
   find its `--pgn-…` variable, find that token's JSON in `brand-openedx`, and name the
   pipeline + caches a change to it would ride through.
6. **(after C13)** The tabletop drill: write down, from memory, the "VPS vanished
   tonight" recovery list, then verify each item actually exists off-machine. This
   mission has found a gap every single time anyone has run it, anywhere.

## Where this section came from

Everything here was learned the hard way on this project between May and July 2026 —
including a day where a one-line CSS fix uncovered a stuck Kubernetes rollout, a node out
of memory, and an expired registry token, all stacked on top of each other
(→ [war story #2](12-war-stories.md#2-the-deploy-that-said-yes-but-did-nothing-the-triple-fault-day-july-2026)). The
runbooks record the *commands* that got us out; these docs record the *understanding*
that got us out.
