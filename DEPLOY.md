# Production deployment (toolkit.globiva.com)

## What 502 Bad Gateway means

Nginx is running, but the **Python app behind it is not** reachable at `http://127.0.0.1:8001`.

Copying code alone is not enough — you must **build and start Docker** with the **production port mapping**.

```
Browser → nginx (443) → http://127.0.0.1:8001 → Docker gunicorn (:8000)
```

---

## Production server checklist (run in order)

SSH to the server as `obtool` (or your deploy user), then:

### Step 1 — Go to the project folder

```bash
cd ~/APP_DIR/Globiva_ToolKit
ls -la docker-compose.yml docker-compose.prod.yml
```

Both files must exist. If `docker-compose.prod.yml` is missing, copy it from the repo or create it (see bottom of this file).

### Step 2 — Start / rebuild the app (required after every code copy)

Your `curl` and `ss` output mean **nothing is listening on 8001 yet** — this step was not done or the container failed.

```bash
cd ~/APP_DIR/Globiva_ToolKit
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

Or use the deploy script:

```bash
bash scripts/deploy-prod.sh
```

Wait until the build finishes (can take several minutes). Then:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps
docker ps -a | grep pdf_tool
```

You should see `pdf_toolkit` and `pdf_tool_redis` with status **Up** (not Exited).

**Alternative** (single compose file + env var):

```bash
APP_HOST_PORT=8001 docker compose up -d --build
```

### Step 3 — Confirm port 8001 is listening

```bash
ss -tlnp | grep 8001
curl -sS http://127.0.0.1:8001/health
```

Expected:

- `grep` shows `127.0.0.1:8001`
- `curl` returns `{"status":"ok"}`

If `curl` says **Connection refused**, the app is not running on 8001 — fix Step 2 before touching nginx.

### Step 4 — If the container is not Up, read logs

```bash
docker logs pdf_toolkit --tail 100
docker logs pdf_tool_redis --tail 20
```

Common causes:

| Log / symptom | Fix |
|---------------|-----|
| Build failed (missing files, pip error) | Fix error, run `up -d --build` again |
| Container exits immediately | Read full `docker logs pdf_toolkit` |
| Only port **3076** in `docker ps` | You did not use `docker-compose.prod.yml` |
| `Cannot connect to Docker daemon` | `sudo systemctl start docker` |

### Step 5 — Confirm nginx points to 8001

```bash
sudo grep -r "proxy_pass" /etc/nginx/sites-enabled/ | grep -i toolkit
```

Must show:

```nginx
proxy_pass http://127.0.0.1:8001;
```

If it shows another port (e.g. 8000, 3076), edit the site config and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Step 6 — Quick automated check

```bash
bash scripts/prod-check.sh
```

---

## One-command deploy (after code copy)

```bash
cd ~/APP_DIR/Globiva_ToolKit
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
curl -sS http://127.0.0.1:8001/health
```

Only when `curl` returns `{"status":"ok"}` will `https://toolkit.globiva.com` work.

---

## Files that must be on the server

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Base services |
| `docker-compose.prod.yml` | Maps **127.0.0.1:8001→8000** |
| `Dockerfile` | Builds the image |
| `app/` | Application code |
| `requirements.txt` | Python deps |

---

## `docker-compose.prod.yml` (if missing on server)

```yaml
services:
  compressor:
    ports:
      - "127.0.0.1:8001:8000"
    restart: unless-stopped
```

---

## Local dev vs production

| Environment | Command | URL |
|-------------|---------|-----|
| Local | `docker compose up --build` | `http://127.0.0.1:3076` |
| Production | `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build` | nginx → `8001` |

---

## Still 502 after health works?

Then nginx config or SSL vhost is wrong — not the app:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8001/login
sudo nginx -t
sudo systemctl status nginx
```
