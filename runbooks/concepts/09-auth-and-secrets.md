# C09 — Auth & secrets: the three unrelated things people call "login"

> **Goal:** stop confusing the three credential worlds, know where every secret in our
> platform lives, and internalize the lifecycle rule that caused a production outage when
> we ignored it (tokens *expire*).

## 1. The three worlds

**World 1 — humans logging into the product.** Session cookies + JWTs on the shared
cookie domain (→ concepts/02 §3). Managed by the LMS; we mostly just consume it (our API
views use the standard `@view_auth_classes` machinery). Failure smell: *one user* can't
do a thing / 401s in the browser.

**World 2 — services trusting each other.** No human present, so: shared secrets and
signatures.
- Stripe → our webhook: Stripe **signs** each event; we verify the signature so nobody
  can fake a "payment succeeded" (→ concepts/10).
- Plugin → AI gateway: a shared secret (`sqa-gateway`) authenticates the control plane;
  students get short-lived **scoped course tokens** instead of real API keys — the entire
  point of the proxytoken design (minors never hold a real key; tokens can be rotated
  and revoked per course, → the broker plan).
Failure smell: a *feature* is broken for everyone (webhooks rejected, gateway 401s), users
fine otherwise.

**World 3 — infrastructure fetching infrastructure.** Machines proving themselves to
platforms: `docker login` with a GitHub **PAT** to push images; the cluster's
`ghcr-pull` imagePullSecret to pull them; SSH to the VPS. **Nothing in world 3 is shared
with worlds 1–2** — logging into the site does not help docker; docker login does not
help the cluster (→ concepts/05 §4). Failure smell: *deploys and pulls* fail; the running
site is initially fine… until a pod needs recreating.

When something auth-ish breaks, first classify the world — it names the fix.

## 2. Where every secret in our platform lives (the census)

| Secret | World | Lives in | Used by | Rotate how |
|---|---|---|---|---|
| Django `SECRET_KEY`, DB passwords | 1 | Tutor `config.yml` → rendered settings | LMS/CMS | tutor config + restart ⚠️ rotating `SECRET_KEY` invalidates every session & signed cookie — the whole site logs out. Planned event, not a casual one |
| Stripe secret key + webhook signing secret | 2 | k8s Secret `sqa-stripe` | plugin webhook/API code | update Secret, restart lms |
| Gateway shared secret + URL | 2 | k8s Secret `sqa-gateway` | plugin ↔ proxytoken service | update both sides, restart |
| Scoped course tokens | 2 | issued/stored by proxytoken service (Neon) | course tooling | admin: rotate/revoke (built-in) |
| GitHub PAT (packages) | 3 | your `docker login` credential store on the VPS | image push/pull by docker | make new PAT, re-login |
| `ghcr-pull` imagePullSecret | 3 | k8s Secret in `openedx` ns | **kubelet** pulling private images | recreate secret with new PAT (see below) |
| SSH keys | 3 | `~/.ssh` | you | standard practice |

Two mechanics worth knowing about k8s Secrets: they're **base64-encoded, not encrypted**
(access control is the protection — never treat a Secret dump as safe to paste), and pods
read them **at start** — updating a Secret does nothing to running pods until recreation.

```bash
# the rotation we actually performed in production (war story #2):
kubectl -n openedx create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io --docker-username=<user> --docker-password='<NEW_PAT>' \
  --dry-run=client -o yaml | kubectl apply -f -    # overwrite-in-place trick
```

## 3. The lifecycle rule (our scar)

Every credential has: **creation → storage → use → expiry/rotation.** Engineers design
the first three and forget the fourth. Our PAT expired; nothing broke *that day* — the
cluster coasted on cached images for weeks. Then a rollout recreated pods, the node
finally had to re-pull, `ImagePullBackOff`, site down — the failure detonated **weeks
after the cause**, disguised as a deployment problem (→ war story #2).

Rules we now follow:

1. **Expiring credentials get a calendar entry.** A PAT with an expiry date is a
   scheduled outage until proven otherwise.
2. **Secrets never enter git or images.** Repos are semi-public (brand-openedx *must* be
   public for the CDN); images leave the machine. Secrets ride config injection only.
3. **Scope minimally.** The PAT has `write:packages`, not `repo`. The gateway secret
   can't touch Stripe. Blast radius is a design input.

## 4. Trade-off corner

- **PATs vs deploy keys / GitHub Apps vs public packages:** finer-grained machinery
  exists (org-scoped apps, OIDC); on a solo project PATs win on simplicity and lose on
  expiry management — accepted, now with reminders.
- **k8s Secrets vs a vault product:** Vault/SOPS add encryption-at-rest and audit for real
  operational cost; single-node, single-operator → overkill today. The census table above
  is deliberately vault-migration-shaped.
- **Scoped short-lived tokens for students** (chosen for the AI broker) vs handing out a
  metered real key: more moving parts (issuance, rotation, revocation, a whole gateway
  service) — but "a minor never holds a real key" was a requirement, not a preference.

## You're ready when…

- Given any 401/403/denied anywhere in the stack, you name the world in one breath and
  the credential store in the next.
- You can perform the PAT + `ghcr-pull` rotation from the census table without this doc.
- You never say "but I logged in!" about the wrong world again.

**Next:** [C10 — payments concepts](10-payments-concepts.md).
