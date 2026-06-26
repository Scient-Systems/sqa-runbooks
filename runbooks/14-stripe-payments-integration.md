# 14 — Stripe payments integration

How the membership billing works end-to-end: the Stripe keys/prices/webhook, where each secret
lives in dev vs prod, the Stripe CLI dev loop, and the gotchas. The backend lives in
`sqa_django_app` (`stripe_service.py`, billing endpoints, the `StripeEvent`/`MembershipPayment`
models); the UI is the `frontend-app-sqa-payment` MFE (runbook 08).

---

## 1. The money flow

```
Pricing page (MFE) ──▶ POST /sqa/api/billing/checkout/ ──▶ Stripe Checkout (embedded, in-page)
                                                                    │ payment succeeds
Stripe ──webhook──▶ POST /sqa/api/webhooks/stripe/ ──▶ plugin sets the user's MembershipLevel
Success page (MFE) ──polls──▶ GET /sqa/api/billing/status/ until subscription_status == "active"
```

🚨 **The webhook is the single write path** for membership state. The success page only *reads*
`/billing/status/`; it never writes. Don't add a second write path.

## 2. The four things Stripe needs

| Thing | What | Where it lives |
|---|---|---|
| `sk_…` secret key | server-side API calls | dev: `sqa_stripe_dev.py`; prod: k8s Secret `sqa-stripe` |
| `pk_…` publishable key | browser (safe, public) | same; returned by the checkout endpoint |
| `whsec_…` webhook secret | verifies webhook signatures | same |
| `price_…` IDs → level | maps a purchase to a tier | `SQA_STRIPE_PRICE_TO_LEVEL` dict (settings) |

The price→level map (from `sqa_stripe_dev.py` / `sqa_stripe_prod.py`):
```python
SQA_STRIPE_PRICE_TO_LEVEL = {
    "price_xxx": "basic",
    "price_yyy": "premium",
    "price_zzz": "enterprise",
}
```
The `/sqa/api/levels/` endpoint also returns each level's `price_id` via the reverse map; the MFE
uses that to build the checkout. The webhook gets `level_slug` from the Checkout Session metadata.

## 3. Dev setup (test mode)

Stripe **test-mode** settings are injected by the host-only Tutor plugin
`~/.local/share/tutor-plugins/sqa_stripe_dev.py` (patches `openedx-lms-development-settings`).
🚨 Never enable this plugin in production.

One-time:
1. Test keys from <https://dashboard.stripe.com/test/apikeys>.
2. Create test Products/Prices in the dashboard; note the `price_…` IDs.
3. Webhook secret for local: `stripe listen --print-secret` (needs the Stripe CLI).
4. Put the `sk_test_`/`pk_test_`/`whsec_` and the price map into `sqa_stripe_dev.py`.
5. Enable: `tutor plugins enable sqa_stripe_dev && tutor dev stop && tutor dev start -d lms`.
6. Turn on the billing flags:
   ```bash
   tutor dev exec lms python manage.py lms shell -c "
   from waffle.models import Switch
   for n in ['sqa_django_app.api_membership','sqa_django_app.api_billing']:
       Switch.objects.update_or_create(name=n, defaults={'active': True}); print(n,'ON')"
   ```
7. ✅ Verify injection: `tutor dev exec lms python -c "import django; django.setup(); from django.conf import settings; print(bool(settings.SQA_STRIPE_SECRET_KEY))"` → `True`.

### Stripe CLI E2E loop
```bash
stripe login
stripe listen --forward-to http://local.openedx.io:8000/sqa/api/webhooks/stripe/
# (copy the whsec_ it prints into sqa_stripe_dev.py if it changed)
stripe trigger checkout.session.completed     # or do a real test-card checkout in the MFE
```

### Test the endpoints by hand
```bash
# Get sessionid + csrftoken from the browser DevTools after logging in, then:
SESSION=<sessionid>; CSRF=<csrftoken>
curl -s -H "Cookie: sessionid=$SESSION; csrftoken=$CSRF" \
  http://local.openedx.io:8000/sqa/api/billing/status/ | python3 -m json.tool
```
Expected (no subscription yet):
```json
{ "level_slug": "basic", "level_display": "Basic", "subscription_status": null,
  "current_period_end": null, "cancel_at_period_end": false }
```

## 4. Production setup (live mode)

Secrets go in the **k8s Secret `sqa-stripe`** — never a plugin file, git, or image (runbook 12 §1c).
The non-secret price map + MFE URL go in `sqa_stripe_prod.py` (patches
`openedx-lms-production-settings`).

```bash
kubectl -n openedx create secret generic sqa-stripe \
  --from-literal=SQA_STRIPE_SECRET_KEY=sk_live_... \
  --from-literal=SQA_STRIPE_WEBHOOK_SECRET=whsec_live_... \
  --from-literal=SQA_STRIPE_PUBLISHABLE_KEY=pk_live_... \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n openedx rollout restart deployment/lms
```
Register the **production** webhook endpoint in the Stripe Dashboard pointing at
`https://lms.stemquestacademy.com/sqa/api/webhooks/stripe/`.

🚨 **A new webhook endpoint issues a NEW `whsec_`.** When you (re)create the endpoint, copy the new
signing secret into the k8s Secret and `rollout restart deployment/lms`, or events from the new
endpoint won't validate.

## 5. The webhook idempotency gotcha

Every event is recorded as a `StripeEvent` with `processed=True` so re-deliveries are ignored. 🚨
Early on, `processed=True` was set even when the handler errored — so a bug couldn't be retried. If
you fix a webhook bug and need to reprocess an event, **delete its `StripeEvent` row** (admin or
shell) so the next delivery is handled fresh.

## 6. Stripe SDK v15 typed-object rule
We use Stripe SDK v15, which returns **typed objects**. Use attribute access and subscript `[]` —
**never `.get()`** on a Stripe object or its `metadata`. (`session.metadata["level_slug"]`, not
`session.metadata.get(...)`.)

## 7. The MFE side (brief)
`frontend-app-sqa-payment`: `/` PricingPage, `/checkout?price_id=…` (Stripe `EmbeddedCheckout`,
in-page, lazy-loaded via `@stripe/stripe-js/pure` + `React.lazy`), `/success?session_id=…` (polls
`/billing/status/` every 2.5s until active, 40s timeout), `/manage`. Deploy = mfe image rebuild
(runbook 12 §3). The header "Membership" link comes from `sqa_payment.py` (runbook 07 §4).

---

## What's next

- Deploy the MFE that drives this → **12-production-deployment-k8s.md** §3
- Anything broke? → **15-troubleshooting-and-gotchas.md**
