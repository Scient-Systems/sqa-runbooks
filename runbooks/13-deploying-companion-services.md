# 13 — Deploying companion services (gateway + Streamlit)

Not everything runs inside Tutor. Two services live **outside** Open edX and deploy on their own:

1. **`sqa-proxytoken-service`** — the AI-token **gateway** (FastAPI on Vercel + Neon Postgres).
2. **The Silly Bot companion app** — a **Streamlit** chat app (Streamlit Community Cloud).

This runbook covers deploying both and wiring the gateway back into the LMS.

---

## 1. Why the gateway exists (the broker model)

Students (often minors) must use AI inside a course without ever holding a real API key. So:

- The **control plane** lives in the Django plugin (`sqa_django_app`): it issues/rotates/revokes
  scoped `sqa_proxy_*` tokens and stores only their **hashes**. (`ProxyToken`,
  `CourseIntegrationGrant`, the `/sqa/tokens/` page, `/sqa/api/proxy/...` endpoints.)
- The **data plane** is this standalone gateway: it validates a presented token against the synced
  hash + scope, and forwards the chat request to an upstream model. The student's app talks only to
  the gateway with the scoped token — never to the real model API with a real key.

The two stay in sync over an internal HTTP API protected by a shared bearer secret
(`SQA_GATEWAY_INTERNAL_SECRET`).

## 2. The gateway's shape (FastAPI)

```
app/main.py            FastAPI app + /health
app/config.py          pydantic-settings (DATABASE_URL, SQA_GATEWAY_INTERNAL_SECRET, model keys)
app/routers/internal.py   POST /internal/tokens        (upsert hash+scopes+limits; Bearer secret)
                          POST /internal/tokens/revoke (mark revoked; 404 if unknown)
app/routers/inference.py  POST /v1/chat                (validate token+scope, forward to upstream)
app/db.py              asyncpg pool + init_db (creates `token` table)
api/index.py           Vercel entry point: `from app.main import app`
vercel.json            rewrites all paths → /api/index
```
Endpoints:
- `GET  /health` → `{"status":"ok","service":"sqa-proxytoken-gateway"}`
- `POST /internal/tokens` and `/internal/tokens/revoke` — called **by the LMS plugin**, guarded by
  `Authorization: Bearer <SQA_GATEWAY_INTERNAL_SECRET>`.
- `POST /v1/chat` — called **by the student's app**, guarded by the scoped `sqa_proxy_*` token;
  `validate()` checks the hash + required scope (`gemini.chat.generate`), then forwards to the
  upstream model (`OLLAMA_BASE_URL`/`OLLAMA_MODEL`, configurable).

## 3. Deploy the gateway (Vercel + Neon)

The repo is already linked to Vercel (`.vercel/`). Day-to-day deploy is just **`git push`** — Vercel
auto-builds. For a fresh setup:

1. **Neon** — create a Postgres project; copy its connection string. The Neon–Vercel integration
   injects prefixed env vars automatically (e.g. `PROXYTOKENSTORAGE_POSTGRES_URL_NO_SSL`).
2. **Vercel** — import the GitHub repo. `vercel.json` already routes everything to `api/index.py`.
3. **Env vars in Vercel** (Project → Settings → Environment Variables):
   - `DATABASE_URL` — 🚨 **don't add a plain one if Neon already injected its prefixed vars.**
     `config.py` uses `AliasChoices` to map the Neon var into the `database_url` field:
     ```python
     database_url: str = Field(validation_alias=AliasChoices(
         "database_url", "proxytokenstorage_postgres_url_no_ssl", "proxytokenstorage_database_url"))
     ```
   - `SQA_GATEWAY_INTERNAL_SECRET` — **add manually** (not provided by any integration). This is the
     shared secret with the LMS.
   - `GEMINI_API_KEY` / model creds — `config.py` also aliases `google_generative_ai_api_key`
     (from the Google/Vercel integration) into `gemini_api_key`. Upstream model URL/model via
     `OLLAMA_BASE_URL` / `OLLAMA_MODEL`.
4. **Verify:**
   ```bash
   curl -s https://sqa-proxytoken-service.vercel.app/health
   ```

## 4. Wire the gateway into the LMS (production)

The LMS plugin reads `SQA_GATEWAY_URL` + `SQA_GATEWAY_INTERNAL_SECRET` from `os.environ`. The
correct prod pattern is a **k8s Secret** (same as Stripe), NOT a Tutor plugin patch — because
`plugin_settings()` overwrites env-derived settings, so a Tutor patch would be clobbered.

```bash
kubectl -n openedx create secret generic sqa-gateway \
  --from-literal=SQA_GATEWAY_URL="https://sqa-proxytoken-service.vercel.app" \
  --from-literal=SQA_GATEWAY_INTERNAL_SECRET="<same secret as Vercel>" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n openedx patch deployment lms --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/envFrom/-","value":{"secretRef":{"name":"sqa-gateway"}}}]'
kubectl -n openedx rollout restart deployment/lms
```

## 5. ✅ Verify token sync end-to-end

After issuing a token on the LMS `/sqa/tokens/` page, confirm it reached the gateway DB:
```bash
# 1. Get the hash from the LMS:
tutor k8s exec lms -- python manage.py lms shell -c "
from sqa_django_app.models import ProxyToken
t = ProxyToken.objects.filter(status='active').last()
print('sync_state:', t.sync_state); print('hash:', t.token_hash)"

# 2. Probe the gateway with that hash (revoke endpoint, internal secret):
curl -s -X POST https://sqa-proxytoken-service.vercel.app/internal/tokens/revoke \
  -H "Authorization: Bearer <SQA_GATEWAY_INTERNAL_SECRET>" \
  -H "Content-Type: application/json" -d '{"token_hash": "<hash>"}'
# {"ok": true} = present in gateway DB.  {"detail":"Token not found"} = sync failed.
```
There's also a reconcile command for drift: `tutor k8s exec lms -- python manage.py lms sqa_proxy_reconcile`.

🚨 `CourseIntegrationGrant` course-key field is strict — a trailing comma raises `InvalidKeyError` →
500. Enter the clean key (`course-v1:SQA+SQA405+2026_T2`).

## 6. The Streamlit companion app

Lives in `~/silly_bot_course_project/silly_bot_streamlit_app/` (also mirrored at the repo root). It's
the student-facing chat UI that talks to the gateway (or, in the course's "own-key" variant, to the
model directly with the student's personal free key).

Deploy on **Streamlit Community Cloud** (deploys from a GitHub repo):
1. Push the app folder to its own GitHub repo.
2. Streamlit Cloud → New app → pick repo + `streamlit_app.py`.
3. **Secrets** (Streamlit → app → Settings → Secrets) — set the key in `secrets.toml` form:
   ```toml
   GEMINI_API_KEY = "..."
   # or the gateway token + URL for the broker variant
   ```
4. Share the public URL — students submit it via the course ORA.

🚨 The app reads its key from `st.secrets`, **never** hardcoded; `secrets.toml` is gitignored. The
model ID must match across the app, the quizzes, and the course text (the build pinned
`gemini-3.5-flash`).

---

## What's next

- Stripe billing specifics → **14-stripe-payments-integration.md**
- Broker control-plane endpoints live in the plugin → **06-installing-a-django-plugin-app.md**
