# C13 — State, data, and not losing it

> **Goal:** the one domain where a mistake can't be fixed by redeploying. Pods, images,
> config — all rebuildable from git in an hour. Data is not. This doc is how to reason
> about it, and it's deliberately the most conservative doc in the folder.

## 1. The stateless/stateful split (the most important line in the cluster)

Every pod is one of two kinds:

- **Stateless** — lms, cms, workers, mfe, caddy, codejail. Kill any of them, nothing is
  lost; they're regenerated from images + config. This is *why* we can treat deploys,
  restarts, and rollbacks so casually everywhere else in these docs.
- **Stateful** — mysql, mongodb, redis, minio, meilisearch. Their pods are still
  disposable, but each mounts a **PVC** (persistent volume — a directory on the VPS disk
  that survives pod death). The pod is cattle; **the PVC is the herd book.**

Corollary that surprises people: `kubectl delete pod mysql-…` is safe (it reattaches the
PVC). `kubectl delete pvc mysql` is the end of the business. Know which command class
you're typing.

## 2. What lives where (the data census)

| Store | Holds | If lost… |
|---|---|---|
| **MySQL** | users, enrollments, grades, **our memberships & levels**, `StripeEvent` log, waffle switches, Django sessions | catastrophic — this is *the* database |
| **MongoDB** | course content (the "modulestore" — what Studio edits) | catastrophic-ish — re-import from OLX exports *if you kept them* |
| **minio** | uploaded files (profile images, course assets, certificates) | painful, partially recoverable |
| **Redis** | cache + Celery task queue | harmless — rebuilds itself (worst case: in-flight background tasks lost) |
| **Meilisearch** | search index | harmless — reindexable from Mongo/MySQL |
| **Stripe** (offsite) | the *financial* truth: payments, sessions, event history | not ours to lose — and it can replay events to us (→ C10) |

Two design consequences worth internalizing: **Stripe is our billing backup** (a lost
`StripeEvent` table can be reconstructed by replaying webhooks), and **OLX exports are
our course backup** (every finished course should have its export committed somewhere —
the Silly Bot course does).

## 3. Migrations × deployments × rollbacks (the classic trap)

Recall (→ C03): a migration is a script that alters real tables, run once at deploy
(in `tutor k8s`, as init **Jobs** — that's what those `mysql-job-*` pods were).

The trap: **code rolls back in seconds (`rollout undo`); schema does not roll back with
it.** After a deploy-with-migration, undoing the deployment leaves *old code reading a
new schema*.

How to reason about it — classify every migration you write:

- **Additive** (new table, new nullable column): old code simply ignores the new bits.
  Rollback-safe. **This should be ~all of your migrations.**
- **Destructive** (drop/rename column, change type): old code breaks against the new
  schema — rollback is *not* an escape hatch anymore, only backup-restore is.
  The professional pattern is **expand → migrate → contract**: ship the new column
  additively first, move the data, and only drop the old column in a *later* deploy,
  after you're sure you'll never roll back past it.

Rule of thumb for this project's scale: if a migration isn't additive, stop and ask why
not, and take a fresh backup immediately before that specific deploy.

## 4. Backups — the discipline (not optional, and already policy)

Project policy (→ runbook 12): **DB backup + version tag before every prod deploy.**
The reasoning: a deploy is the moment of maximum change; the backup converts every
worst case from "unrecoverable" to "restore + lose a few hours".

Concepts that make backups real instead of theater:

1. **A backup you've never restored is a hope, not a backup.** Do one practice restore
   into a scratch database. Once. You'll find the missing step *then*, not during an
   outage.
2. **Dump-level beats disk-level here.** `mysqldump`/`mongodump` produce portable,
   inspectable files, independent of PVC internals. (Disk snapshots of a *running*
   database can be internally inconsistent.)
3. **Off-machine or it doesn't count.** The VPS disk dying takes the PVCs *and* any
   backups sitting next to them. Copy dumps off the box.
4. **Consistency pairing:** MySQL and Mongo reference each other (enrollments ↔ course
   content). Back them up at the same time, restore them as a pair.

## 5. Where state hides outside the databases (the sneaky list)

- **Waffle switches** — behavior toggles are *rows in MySQL*, not code. A restored old
  backup can silently flip features back. Check Django Admin after any restore.
- **Django sessions** — MySQL rows; wiping them (or rotating `SECRET_KEY`, → C09) logs
  everyone out. Annoying, not dangerous.
- **The k8s Secrets** — not in any backup dump! Rebuild-from-scratch needs config.yml
  **plus** the secret values from wherever you keep them (password manager). Do the
  thought experiment now: *if the VPS vanished tonight, what exactly do I need to rebuild
  by morning?* Answer: git repos + config.yml + secret values + last DB dumps + OLX
  exports. If any of those five exists only on the VPS, fix that today.
- **Stripe webhook endpoint config** — lives in the Stripe dashboard, not in our repos.

## 6. Trade-off corner

- **Manual pre-deploy dumps** (current) vs scheduled automated backups: automation
  removes the human forgetting, adds silent-failure risk ("the cron job broke in March").
  If/when automated: alert on backup *absence*, not just failure. Honest status: at our
  scale, manual-but-ritualized is defensible; automated + off-site is the next maturity
  step.
- **Self-hosted MySQL/Mongo on PVCs** (current, Tutor default) vs managed databases:
  managed = backups, failover, and disk health become someone else's job, at real monthly
  cost + latency. The single strongest argument for migrating someday is everything in
  this doc becoming somebody else's pager.

## You're ready when…

- You can answer the "VPS vanished tonight" question with the five-item list, from memory.
- Before any deploy you can say whether its migrations are additive, and what your escape
  hatch is (undo vs restore) — *before* pressing enter.
- You know which `kubectl delete` targets are cattle and which are the herd book.

**Next:** [C14 — anatomy of a feature](14-anatomy-of-a-feature.md) — the build-side
synthesis of everything.
