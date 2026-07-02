# C11 — The debugging method: find the first broken hop

> **Goal:** one transferable skill. Every incident in war stories (→ concepts/12) was
> solved with this method; every hour we ever wasted was wasted by abandoning it.

## 1. The core idea

Every change you make travels a **pipeline of hops** before a user sees it, and every
observed bug sits at **exactly one hop**. Debugging is not staring at symptoms and
guessing — it is walking the pipeline **in order** and asking at each hop one binary
question: *"did my change / the request make it THIS far, intact?"* The first "no" is
the bug's address. Everything downstream of it is noise.

Two disciplines make it work:

1. **Never skip hops.** The temptation is to jump to the end ("it must be the browser
   cache!") and start changing things. Each unverified hop you skip can hide the actual
   break, and each thing you change while guessing adds a *second* variable.
2. **State the hypothesis before the command.** "If the env/ render worked, this grep
   finds my string." Then run it. A command you can't predict the meaning of both
   outcomes for is a flashlight waved at random.

## 2. Our standard chains (memorize the shape, look up the commands)

**Code/theme deploy chain** — for "I deployed X but production shows old X":

```
1 repo        git log — is the change committed & pushed, right branch?
2 VPS clone   git -C /home/edx/<repo> log — pulled?
3 render      grep -r "<change>" $(tutor config printroot)/env/   (→ C06)
4 image       docker run --rm <image> grep …  — baked in? (layer cache! → C05)
5 registry    pushed? tag moved?
6 cluster     new ConfigMap/pod spec? (tutor k8s start ran?)      (→ C07)
7 pod         rollout status — did it CONVERGE? (stuck rollout = the great liar)
8 process     kubectl exec … grep — the running container has it?
9 delivery    curl the URL — right content, right Content-Type?
10 browser    hard refresh; devtools Network tab — what was actually fetched?
```

**Request chain** — for "the site is broken": exactly journey A/B from → concepts/02,
walked with the failure catalog table there (502 → pods; 500 → logs; blank MFE → config
API + devtools).

**Money chain** — for "payment weirdness": MFE → session created (plugin logs) → Stripe
dashboard events → webhook delivery attempts + our responses → `StripeEvent` rows →
membership row (→ concepts/10).

## 3. The cache ambush (a hop-walker's field guide)

Caches exist at half the hops, and each can serve you yesterday convincingly:

| Cache | Smells like | Flush / bypass |
|---|---|---|
| Docker layer cache | build "succeeded" suspiciously fast; stale artifact ships | `--no-cache` (→ C05 §2) |
| Node's image cache | pushed new image; pods run old one | recreate pods; remember tag ≠ digest |
| LMS mfe-config cache (Redis, 5 min) | MFEs ignore new config after restart | wait 5 min, or clear cache via LMS shell |
| jsDelivr CDN (~5 min) | CSS URL serves old content | SHA-pinned URLs make this harmless (→ C08 §4) |
| Browser | only you see the old thing | Ctrl+Shift+R; verify in devtools what was fetched |

Rule: when a hop's output looks stale, **suspect the cache at that hop before suspecting
the hop's input** — then verify upstream anyway.

## 4. Reading the three great error surfaces

- **Browser devtools → Network tab.** For every suspicious resource: final URL, status
  code, response Content-Type, response body. (Our raw.githubusercontent bug was
  invisible everywhere *except* the Content-Type column — the browser dropped the
  stylesheet without logging an error. → war story #4)
- **Pod logs** (`logs deploy/lms --tail=100`, `--previous` after crashes). Python
  tracebacks read bottom-up: last line = the error, walk up to the first line that's
  *our* code.
- **`describe pod` Events.** Scheduling, image pulls, probe failures — infrastructure
  confesses here, not in app logs (→ C07 §6).

## 5. When multiple things break at once

War story #2 was three independent faults stacked (stale config + unschedulable rollout +
dead registry credential). The method still works, with one addition: **fix one fault,
then re-walk the chain from the top.** Fixing the rollout *exposed* the pull failure;
that's the method succeeding, not failing — each pass peels one layer. Panic-fixing all
three at once would have produced a state nobody could reason about. Also: keep a scratch
log of what you changed — under stress, memory lies.

And know your **escape hatches** before you need them: `kubectl rollout undo` (previous
ReplicaSet), DB backup before deploys (→ runbook 12 & concepts/13), git revert +
redeploy. A calm rollback beats a heroic forward-fix at 2 a.m.

One honest limitation to hold in mind: **this platform has no automated alerting.**
Nothing pages anyone; broken means *broken until a human looks*. That's a deliberate
solo-scale trade-off, and it converts into a personal discipline: after every deploy,
*you* are the monitoring — run the verification steps (curl the site, `get pods`,
Stripe webhook deliveries after billing changes) even when everything "obviously
worked". War story #3 ran silently for exactly as long as nobody looked.

## You're ready when…

- "It deployed but nothing changed" triggers chain-walking, not vibes.
- You can name, for each of the 5 caches, one symptom and the flush.
- You've internalized: **apply ≠ converge, pushed ≠ running, built ≠ fresh, 200 ≠
  handled** — the four lies of a healthy-looking pipeline.

**Next:** [C12 — war stories](12-war-stories.md), the method applied to everything that
actually happened to us.
