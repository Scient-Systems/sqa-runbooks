# C07 — Kubernetes from zero (using only our own cluster as the example)

> **Goal:** enough Kubernetes to operate and debug our production **with understanding**.
> Everything here uses our real cluster; nothing is hypothetical. This is the longest
> concepts doc because it's where a wrong mental model costs the most (→ war story #2).

## 1. Why Kubernetes exists

Containers (→ concepts/05) run programs, but somebody must *babysit* them: restart the
crashed ones, wire them to each other, feed them config and secrets, replace them with
new versions without chaos. Kubernetes (k8s) is that babysitter, with one radical design
choice — it is **declarative**:

> You never tell k8s what to *do*. You tell it what should be *true*
> ("one LMS pod of image X with config Y exists"), and its controllers work
> continuously to make reality match.

This inverts your instincts. `kubectl apply` doesn't "run" anything — it updates the
*wish*. `tutor k8s start` just applies all our wishes. The doing happens asynchronously,
which is why a deploy can "succeed" while reality quietly fails to follow — you must
**check convergence**, not trust the apply (the deepest lesson of war story #2).

## 2. The vocabulary, via our actual cluster

`kubectl -n openedx get pods` shows our whole world. Decoder:

- **Node** — a machine that runs pods. We have exactly one: the 8 GB VPS. Every
  scheduling problem we've ever had comes from that sentence.
- **Pod** — the running unit: one or more containers + shared network. Pods are
  *disposable* — killed and recreated freely; anything precious must live elsewhere.
- **Deployment** — the wish "N copies of this pod spec, and here's how to roll out new
  versions". You almost never touch pods directly; you touch deployments
  (`lms`, `cms`, `mfe`, `caddy`, `mysql`, …).
- **ReplicaSet** — the deployment's bookkeeping generation. Each pod-spec change makes a
  new one; the hash in pod names (`lms-7454cd859c-8pfbw`) identifies it.
- **Service** — a stable internal name+IP in front of pods ("call `lms`, whichever pod
  that is today"). Pods' own IPs change constantly; services are why nobody cares.
- **ConfigMap / Secret** — config data injected into pods as files or env vars. Secret =
  same but for credentials, access-controlled (⚠️ base64-encoded, **not** encrypted).
  Ours include `sqa-stripe`, `sqa-gateway`, and `ghcr-pull` (→ concepts/09).
- **PVC (PersistentVolumeClaim)** — a disk that survives pod death. Under mysql, mongo,
  redis, minio. **This is where the actual data is** — pods are cattle, PVCs are the herd
  book.
- **Job** — a run-once pod (migrations, init). The old `mysql-job-* Error` entries in our
  listing are dead leftovers, not live problems.
- **Namespace** — a folder for all of the above. Everything ours: `openedx`. Hence the
  eternal `-n openedx`.

## 3. Config flow — why restarts are even needed

A pod reads its ConfigMaps **when it starts**. So new settings reach production in three
moves: render the files (`tutor config save`) → get them into cluster objects + specs
(`tutor k8s start`) → **recreate the pods**.

Neat mechanism worth knowing (you'll see it in apply output): Tutor's manifests give
ConfigMaps **content-hashed names** (`openedx-settings-lms-k6ctb8bd57`). Changed
settings ⇒ new hash ⇒ new ConfigMap ⇒ the deployment's pod spec now references a
different name ⇒ **the apply itself triggers a rollout**. That's why war story #2's
apply printed `…-k6ctb8bd57 created` and `deployment.apps/lms configured` — the config
chain had provably worked, which is what let us pin the failure to the rollout hop. The
extra `rollout restart` in our pipelines is belt-and-braces for the cases where content
*didn't* change name.

> 🚨 Skip the third and you get the sneakiest state in operations: **cluster updated,
> pods stale.** Everything *looks* deployed; the running processes never heard about it.

## 4. Rollouts — where our node's size becomes destiny

Default deployment strategy: **RollingUpdate with surge** — start the *new* pod, wait
until it's Ready, only then kill the old. Zero downtime… **if the node can hold both
copies at once.**

Scheduling runs on **requests** (each pod's declared appetite), not live usage. One LMS
pod wants ~2 GB. Our node can't host two. So the surge pod sits `Pending`
(`FailedScheduling: Insufficient memory`) *forever*, while the old pod — old image, old
config — keeps serving. **A rollout that never finishes looks exactly like a deploy that
lied to you.**

Our fix: `maxSurge: 0, maxUnavailable: 1` on lms/cms/workers — kill old first, then start
new. Honest trade: **every deploy now has ~2 minutes of downtime** in exchange for
deploys that always complete. On one 8 GB node those are the only two options; pick your
poison consciously (we did: `K8S_ROLLOUT_MEMORY_FIX.md`). The real third option is a
16 GB node.

Emergency lever: `kubectl -n openedx rollout undo deployment/lms` — the previous
ReplicaSet is kept around precisely so you can go back in seconds.

## 5. Image pulls — the other silent dependency

When a pod is scheduled, the node needs the image: cached locally → start instantly;
otherwise pull from GHCR → **needs the `ghcr-pull` secret to be valid**. `ImagePullBackOff`
= "I asked the warehouse and was refused (or it wasn't there)" — an *auth/registry*
problem wearing a Kubernetes costume (→ concepts/05 §4, concepts/09).

## 6. The inspection toolbox — organized by the question you're asking

| Question | Command |
|---|---|
| Is everything running? | `kubectl -n openedx get pods` — read STATUS + READY + RESTARTS |
| Why is this pod not running? | `kubectl -n openedx describe pod <name>` — **read Events at the bottom**; scheduling, pulling, and probe failures all confess here |
| What is the app itself saying? | `kubectl -n openedx logs deploy/lms --tail=50` (add `-f` to follow, `--previous` for the *last* crash) |
| What happened cluster-wide recently? | `kubectl -n openedx get events --sort-by=.lastTimestamp \| tail -20` |
| Did my rollout actually finish? | `kubectl -n openedx rollout status deployment/lms` — **the honest one; never trust `apply` alone** |
| Let me look inside the container | `kubectl -n openedx exec -it deploy/lms -- bash` (or `tutor k8s exec lms bash`) |
| Who's eating the node? | `kubectl top pods -n openedx --sort-by=memory` (live usage) / `kubectl describe node \| grep -A10 "Allocated resources"` (what the *scheduler* believes — these differ, and scheduling runs on the second!) |

Status decoder: `Pending` = no room/can't schedule → describe, look for FailedScheduling.
`ImagePullBackOff` = can't fetch image → auth or missing tag. `CrashLoopBackOff` = starts
then dies → `logs --previous`. `Running 0/1` = alive but failing its readiness probe →
still booting, or sick → logs.

## 7. Trade-off corner: was k8s even the right call?

On a single VPS, `tutor local` (docker-compose) would be *simpler*: no registry pulls, no
scheduling, no pull secrets, fewer moving parts. We chose k8s for: declarative
reproducibility, first-class secrets/config objects, `rollout undo`, and a growth path.
Be honest about that growth path though: a second node would relieve the *stateless*
pods (lms could surge again), but our databases sit on **node-local PVCs** — they're
pinned to this machine until we move to networked storage or managed databases
(→ concepts/13 §6). "Just add a node" is half true; the stateful half is a project.
The costs we knowingly eat: the memory-vs-downtime rollout dilemma, registry auth
lifecycle, and a steeper learning curve — this doc *is* that cost, paid forward.

## You're ready when…

- `get pods` output reads like a sentence, not a wall.
- You can narrate a rollout: new ReplicaSet → surge or replace → readiness → old scaled
  down — and where ours deadlocks and why.
- `Pending` / `ImagePullBackOff` / `CrashLoopBackOff` each map instantly to a *different*
  next command in your head.
- You know which objects hold real data (PVCs, Secrets) and which are cattle (pods).

**Next:** [C08 — theming concepts](08-theming-concepts.md).
