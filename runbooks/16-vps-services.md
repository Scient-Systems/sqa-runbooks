# 16 — Services on the VPS (getting started + what runs where)

Not everything is Open edX. A second box (**"the services VPS"**) runs the company's
supporting apps — n8n, castopod, the fijjy pipeline, the scient APIs, ollama, and more.
This runbook is the map: how to get on the box, what runs where, and how to start each thing.

> These services were migrated onto this box in **July 2026** and all run behind a single
> **Caddy** reverse proxy with automatic HTTPS. DNS is fronted by **Cloudflare**.

---

## 1. The box

| | |
|---|---|
| **IP** | `74.208.99.247` |
| **OS** | Ubuntu 26.04 LTS |
| **Login user** | `scient` (has `sudo` and is in the `docker` group) |
| **Runtimes** | Docker + Compose, and **k3s** (single-node Kubernetes) — they coexist |
| **Front door** | Caddy on `:80`/`:443` (auto Let's Encrypt); k3s Traefik is disabled |

```bash
ssh scient@74.208.99.247
```

**Where things live:**

- `~/apps/<service>/` — each service's source + `docker-compose.yml` + `.env`
- `~/migration/` — data tarballs used during the migration (castopod DB, n8n volume)
- `~/apps/fijjy-clean.yaml` — the fijjy Kubernetes manifest
- `/etc/caddy/Caddyfile` — all reverse-proxy routes

---

## 2. What's running (and on which URL)

Everything is reachable at `<name>.helpingdevelopers.com` via Caddy → Cloudflare.

### Docker services

| Service | Dir (`~/apps/…`) | Host port | Public URL |
|---|---|---|---|
| **n8n** (automation) | `n8n` | 5678 (localhost only) | `n8n.` |
| **castopod** (podcast host) | `castopod` | 8008 | `podcasthost.` |
| **fauxrun** (cover-letter API) | `fauxrun-api` | 9003 | `fauxrun.` |
| **scient-cod-api** | `scient-cod-api` | 8001 | `cod.` |
| **scient-email-be** (email sequencing) | `scient-email-be` | 9837 | `scientemailbe.` |
| **scient-business-card-api** (OCR) | `scient-business-card-api` | 8103 | `ocrbusinesscard.` |
| **ollama** (LLM server) | *(plain `docker run`)* | 11434 | `ollama.` |
| **ollama-wrapper** (auth layer) | `ollama-wrapper/ollama-wrapper` | 8000 | *(internal, no subdomain)* |

### Kubernetes (k3s) — the fijjy pipeline

Runs in namespace **`fijjy-generator`**. Images are pulled from GHCR
(`ghcr.io/scient-systems/*`) using the `ghcr-pull-secret`. Exposed via NodePorts:

| Workload | Service | NodePort | Public URL |
|---|---|---|---|
| whisperx-api (transcription) | `whisperx-api` | 30017 | `transcription.` |
| coordinator (media pipeline) | `coordinator` | 30090 | `fijjymediapipeline.` |
| coordinator-social | `coordinator-social` | 30092 | `social.` |

```bash
kubectl get pods -n fijjy-generator
```

---

## 3. Getting started — start / stop / check a service

### Docker services

```bash
cd ~/apps/<service>

docker compose ps            # what's up
docker compose up -d         # start (uses the existing built image)
docker compose up -d --build # rebuild from source, then start
docker compose logs -f       # tail logs
docker compose down          # stop
```

Quick health check from the box (each app serves FastAPI docs at `/docs`):

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8001/docs   # cod
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:9837/docs   # email-be
```

> Some APIs (cod, fauxrun) show **`(unhealthy)`** in `docker ps`. That's only because their
> Docker healthcheck hits `/health`, which they don't implement — the apps serve fine on
> `/` and `/docs`. Cosmetic, not a real failure.

### ollama (server + wrapper)

The **server** is a plain container; the **wrapper** is a compose app that talks to it at
`http://74.208.99.247:11434` (hardcoded in `main.py`) and uses the `phi4-mini` model.

```bash
# server
docker start ollama                       # already created; just start it
docker exec ollama ollama list            # models present
docker exec ollama ollama pull phi4-mini  # (re)pull the model the wrapper uses

# wrapper
cd ~/apps/ollama-wrapper/ollama-wrapper && docker compose up -d
```

### fijjy (k3s)

```bash
kubectl apply -f ~/apps/fijjy-clean.yaml   # apply/update the manifest
kubectl get pods -n fijjy-generator        # status
kubectl logs -n fijjy-generator deploy/coordinator --tail=50
```

If pods sit in **`ImagePullBackOff`**, the GHCR token expired. Refresh the pull secret:

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io --docker-username=<gh-user> \
  --docker-password=<new-PAT-with-read:packages> \
  -n fijjy-generator --dry-run=client -o yaml | kubectl apply -f -
kubectl delete pods --all -n fijjy-generator   # recreate so they re-pull
```

---

## 4. Routing (Caddy)

All routes live in `/etc/caddy/Caddyfile` — one block per subdomain, e.g.:

```caddyfile
cod.helpingdevelopers.com {
	reverse_proxy 127.0.0.1:8001
}
```

To add or change a route:

```bash
sudo nano /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Caddy fetches the TLS cert automatically once the name resolves to this box. k3s Traefik is
disabled (`disable: traefik` in `/etc/rancher/k3s/config.yaml`) so Caddy owns `:80`/`:443`;
fijjy is reached through its NodePorts, not Traefik.

---

## 5. Gotchas worth knowing (these bit us during the migration)

!!! warning "Unpinned build tools drift and break rebuilds"
    Two images built fine months ago but **failed to rebuild** because a tool floated to a
    newer, incompatible version:

    - **castopod** — the Dockerfile used `corepack prepare pnpm@latest`; pnpm v10 turned an
      ignored-build-scripts warning into a hard error. **Pinned `pnpm@9`.** Also raised
      `COMPOSER_PROCESS_TIMEOUT` (castopod pulls a dep from their own git server, which is
      occasionally down/502).
    - **scient-business-card-api** — `paddleocr==3.1.1` was pinned but its `paddlex`
      dependency floated to `3.7.2` and crashed on startup. **Pinned `paddlex==3.1.3`.**

    Lesson: if a rebuild suddenly fails, suspect an unpinned transitive dependency first.

!!! note "n8n encryption key"
    n8n's saved credentials only decrypt if the **encryption key** inside its data volume
    (`~/.n8n/config`) is preserved. It rode along in the volume during migration — never let
    n8n regenerate a fresh key or every stored credential breaks.

!!! note "Where the data actually lives"
    Most of these apps keep their real data **off-box** (fauxrun and the scient APIs use
    MongoDB Atlas; castopod media is on S3/Backblaze). The only volumes that had to be
    migrated were **castopod's MariaDB** and the **n8n** volume. fijjy's k8s data is
    regenerable (whisperx re-downloads its models).

---

## 6. Quick reference

```bash
# everything Docker
docker ps --format "{{.Names}}\t{{.Status}}\t{{.Ports}}"

# everything k8s
kubectl get pods -n fijjy-generator

# reverse proxy
sudo systemctl status caddy
cat /etc/caddy/Caddyfile
```
