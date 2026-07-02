# C02 — A request's journey: from URL to pixels

> **Goal:** know every hop a request makes, because **every bug lives at exactly one
> hop**, and debugging is just finding which one (→ concepts/11).

## 1. Journey A — a legacy LMS page (server-rendered)

A logged-in student opens `https://lms.stemquestacademy.com/dashboard`:

```
browser
  1 ──DNS──▶  "lms.stemquestacademy.com" → the VPS's IP address
  2 ──TLS──▶  Caddy pod (owns ports 80/443, has the HTTPS certificate)
  3 ──route─▶ Caddy sees the hostname, forwards to the `lms` Service
  4 ──k8s──▶  the Service forwards to the LMS pod (→ concepts/07)
  5 ──django▶ Django looks at the URL, picks a view function
  6 ──auth──▶ the session cookie says who you are
  7 ──data──▶ the view queries MySQL (your enrollments), maybe Redis (cache)
  8 ──html──▶ a Mako template renders a COMPLETE HTML page, with CSS from
              our theme baked into the openedx image
  9 ◀─────── browser receives finished HTML and just displays it
```

Key property: **the server did all the work.** The HTML arrives ready. If this page looks
wrong, the problem is server-side (theme SCSS in the image, template, view logic).

## 2. Journey B — an MFE page (browser-rendered)

The same student opens `https://apps.lms.stemquestacademy.com/profile/u/their-name`:

```
browser
  1–3   same DNS/TLS/Caddy dance, but routed to the `mfe` pod
  4 ──▶ the mfe pod is just nginx serving FILES: it returns a mostly-empty
        HTML page plus a big compiled JavaScript bundle (React)
  5 ──▶ the browser RUNS the JavaScript. React boots up
  6 ──▶ the app calls the LMS API to configure itself:
        GET lms.stemquestacademy.com/api/mfe_config/v1
        (this returns runtime settings — including our theme CSS URLs!)
  7 ──▶ the app loads theme CSS from a CDN (jsDelivr) per that config
  8 ──▶ the app calls more LMS APIs for actual data (profile, enrollments…)
        authenticated by a JWT cookie shared across *.stemquestacademy.com
  9 ──▶ React renders the page in the browser
```

Key properties, and they explain SO many of our incidents:

- **The MFE pod is dumb.** It serves static files. MFE bugs are either in the bundle
  (build-time) or in the APIs/config the browser fetches (runtime).
- **The page assembles itself from multiple sources** — bundle from our pod, config from
  the LMS, CSS from a CDN, data from LMS APIs. Each source can independently be stale,
  cached, broken, or blocked. When "the deploy didn't work", ask: *which source am I
  actually changing, and did that source update?*
- **The LMS is the brain even for MFE pages.** If the LMS is down, MFEs break too — they
  can't even fetch their config.

## 3. The auth thread through both journeys

You logged in once, at the authn MFE. How does every other app know you?

- The LMS set cookies on the **parent domain** — so both `lms.…` and `apps.lms.…` send
  them automatically.
- One cookie is a classic Django **session** (used by legacy pages); another is a **JWT**
  — a signed token MFE JavaScript can send to APIs (→ concepts/09).

That's the whole trick: shared cookie domain + signed tokens. No magic.

## 4. Where each journey can break (the failure catalog)

| Hop | Failure looks like | First check |
|---|---|---|
| DNS | site unreachable, wrong server | `dig` / did DNS change? |
| Caddy/TLS | certificate errors, connection refused | `kubectl -n openedx logs deploy/caddy` |
| Caddy→pod | **502 Bad Gateway** = front door fine, nobody home behind it | `kubectl -n openedx get pods` — is the LMS pod Ready? |
| Django view | 500 + stack trace | `kubectl -n openedx logs deploy/lms` |
| Database | 500s everywhere, timeouts | mysql/mongo pod health |
| MFE bundle | old UI served after "deploy" | did the mfe image actually rebuild? (→ concepts/05 cache story) |
| mfe_config API | MFE ignores new settings | the LMS caches this response for 5 min (Redis) |
| CDN CSS | styles stale or missing | browser devtools → Network tab → what did the CSS URL return? |
| Browser cache | only *you* see the old version | hard refresh (Ctrl+Shift+R) |

Memorize one: **502 means the request made it to Caddy but the app pod behind it isn't
answering.** During our stuck-rollout incident (→ war story #2) the site showed 502
precisely because the old LMS pod was killed and the new one couldn't start — the front
door was fine the whole time.

## 5. Trade-off corner: why does the platform even have two journey types?

Server-rendered pages (A) are simple: one place to debug, no API layer, but every
interaction reloads the page and the UI code is tangled into the backend. Browser-rendered
apps (B) give a modern, instant-feeling UI and let frontend teams ship independently — at
the price of distributed failure modes (config APIs, CORS, token auth, CDNs). Open edX is
migrating from A to B and is **permanently mid-migration** as far as we're concerned —
which is why our theme has to be delivered through two pipelines (→ concepts/08) and why
you must always first ask: *is this page legacy or MFE?* (Quick tell: `apps.` subdomain =
MFE.)

## You're ready when…

- You see a 502 and your hand types `kubectl -n openedx get pods` before your brain panics.
- You can explain why an MFE page can be broken even though the mfe pod is perfectly
  healthy (its config/data/CSS come from elsewhere).
- Given any URL on our platform you can say: legacy or MFE, and which pod serves it.

**Next:** [C03 — Django & the plugin idea](03-django-and-plugins.md).
