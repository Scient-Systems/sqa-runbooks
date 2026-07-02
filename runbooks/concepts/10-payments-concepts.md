# C10 — Payments concepts: why Stripe calls US, and why "twice" is the enemy

> **Goal:** understand the checkout + webhook model deeply enough to reason about
> failures (double-charges, stuck memberships, silent losses) instead of just reading
> Stripe docs commands-first.

## 1. The cast

- **Stripe** holds the card data and moves the money. We never see a card number — that's
  the entire point (PCI compliance stays Stripe's problem).
- **Our payment MFE** (`frontend-app-sqa-payment`) renders pricing/checkout/manage pages.
- **Our plugin** owns the truth: which user has which membership level.

## 2. The flow, and the one idea that explains its shape

```
student clicks "Buy Premium" in the MFE
  1. MFE → plugin API: "create a Checkout Session for level=premium"
  2. plugin → Stripe: creates the Session (embedded ui_mode),
     stamping metadata: {user_id, level_slug}          ← the join key!
  3. student types card details into Stripe's embedded form
     (the card data goes browser → Stripe directly, never touches us)
  4. Stripe processes payment…
  5. Stripe → OUR WEBHOOK: "checkout.session.completed" (signed!)
  6. plugin verifies the signature, reads metadata.level_slug,
     assigns the membership              ← money becomes membership HERE
  7. MFE polls/refreshes and shows the new level
```

**The one idea:** the browser is not trustworthy. It can close mid-payment, lie, retry,
or lose network at any step. So the state-changing step (6) is **server-to-server**:
Stripe calls *us*, on a URL only reachable with a validly *signed* event. The redirect
back to a success page is decoration; **the webhook is the transaction.**

Corollary you must internalize: membership assignment is **asynchronous**. A user can pay
and, for a few seconds (or, if webhooks break, indefinitely), not have the level. "Paid
but no membership" ≙ *look at webhook delivery first* (Stripe dashboard → Webhooks →
recent deliveries — Stripe records every attempt, response code, and will retry failures
for days).

The `metadata` stamp in step 2 is the only thread connecting a Stripe event back to *our*
user and level — set at creation, echoed back in every event. Lose it and a payment is
money without meaning.

## 3. Idempotency — the concept our nastiest billing bug lived in

**Idempotent** = safe to run twice. Webhooks *will* arrive twice (Stripe retries on
timeouts; networks duplicate). Processing "session completed" twice must not create two
memberships.

Standard cure (we use it): record every processed event ID (`StripeEvent` table); on
arrival, if the ID is already recorded — acknowledge and skip.

> 🚨 **The subtle failure mode we shipped (war story #3):** our handler marked events
> `processed=True` even when the handler *threw an error* halfway. Consequence: a failed
> event was never retried — Stripe got a 200, we got a permanently stuck membership, and
> nobody got an alarm. The fix distinguished "seen" from "successfully handled" (and the
> operational unstick: delete the `StripeEvent` row, replay the event from Stripe's
> dashboard). The general lesson generalizes far beyond billing: **dedupe keys must mark
> success, not attempts.**

## 4. Environments and safety rails

- **Test vs live mode:** two parallel Stripe universes with separate keys
  (`sk_test_…`/`sk_live_…`), products, and webhook secrets. Test cards
  (`4242 4242 4242 4242`) only work in test mode. Local dev runs test mode with
  `stripe listen` forwarding webhooks to WSL2 (→ runbook 14).
- Keys live in the `sqa-stripe` k8s Secret (→ concepts/09) — never in code, never in
  images.
- Level ↔ Stripe price mapping: the plugin exposes `/api/levels/` including each level's
  `price_id`, so the MFE never hardcodes Stripe IDs; the webhook maps back via metadata
  `level_slug`. One mapping, owned server-side.
- SDK quirk worth knowing before it bites: Stripe's Python SDK (v15) returns typed
  objects — attribute/`[]` access, **`.get()` does not exist** on them.

## 5. Trade-off corner

| Decision | Chosen | Alternative | Trade-off |
|---|---|---|---|
| Checkout UI | **Embedded** Checkout (Stripe form inside our MFE page) | Redirect to Stripe-hosted page | Hosted = less code, more trust-me-redirect UX; embedded keeps users on-brand at the cost of MFE integration quirks (lazy-load the Stripe JS; `ui_mode` history is a maze — see project memory) |
| Learning the money truth | **Webhooks** (push) | Polling Stripe for session status | Polling is simpler to reason about but slow, rate-limited, and still needs dedupe; webhooks are the industry answer — the cost is everything in §3 |
| Billing model | One-time level purchases (Membership 1.0) | Stripe Subscriptions | Subscriptions add proration/renewal/dunning machinery; not needed yet — revisit when memberships expire |

## You're ready when…

- You can say why the success-page redirect must never assign the membership.
- "Paid but didn't get the level" → your first three checks are: Stripe webhook delivery
  log, our webhook endpoint's responses, the `StripeEvent` row — in that order.
- You can define idempotency and give the §3 failure mode from memory.

**Next:** [C11 — the debugging method](11-debugging-method.md).
