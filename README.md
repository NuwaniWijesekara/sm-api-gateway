# ScanMe — API Gateway

Pure Nginx reverse proxy. Single entry point for all API traffic on port 80. Routes requests to the correct backend service, applies rate limiting per route, and handles CORS headers.

**Port:** `80`  
**Stack:** Nginx 1.25 Alpine — no Python, no application code

---

## What This Gateway Does

| Route | Routes To | Rate Limit | Notes |
|---|---|---|---|
| `/auth/*` | Photographer Service :8001 | 20 req/s | Public — signup and login |
| `/events/*` | Photographer Service :8001 | 20 req/s | JWT required (enforced inside service) |
| `/guest/*` | Guest Service :8002 | 200 req/s | Public — event gallery |
| `/match/*` | Guest Service :8002 | 5 req/min | Public — GPU protected, strict limit |
| `/health` | Gateway itself | None | Returns `{"status":"ok"}` |

### Why Nginx and not a Python gateway

The gateway has no business logic — it only routes, rate limits, and forwards headers. Nginx handles this natively with near-zero overhead. A Python gateway (FastAPI/Express) would add latency and a crash surface for no benefit.

---

## Project Structure

```
sm-api-gateway/
├── nginx/
│   └── nginx.conf     # All routing, rate limiting, and CORS rules
├── Dockerfile
└── README.md
```

There are no application files, no `.env`, and no dependencies to install.

---

## Running with Docker (only supported method)

Nginx runs inside a Docker container — there is no local install required on Windows.

### Build and run

```bash
cd sm-api-gateway

docker build -t scanme-gateway .

docker run -p 80:80 scanme-gateway
```

### Verify it is running

```bash
curl http://localhost:80/health
# Expected: {"status":"ok"}
```

---

## Local Development — Connecting to Local Services

When your backend services run locally with uvicorn (not in Docker), the container cannot reach `localhost` directly. Use `host.docker.internal` which Docker Desktop resolves to your Windows/macOS host machine.

Edit `nginx/nginx.conf` upstreams before building:

```nginx
# For local development (services running with uvicorn)
upstream photographer_service {
    server host.docker.internal:8001;
}

upstream guest_service {
    server host.docker.internal:8002;
}
```

Then rebuild:

```bash
docker build -t scanme-gateway .
docker run -p 80:80 scanme-gateway
```

Your full local stack then looks like:

```
Browser → localhost:3000 (Next.js)
               ↓ API calls to localhost:80
          Docker Nginx Gateway
               ↓                    ↓
    host.docker.internal:8001   host.docker.internal:8002
    (uvicorn photographer)      (uvicorn guest)
```

---

## Production — Connecting to Dockerised Services

When all services run in the same Docker network (via `docker-compose`), use the Docker service names:

```nginx
# For production / docker-compose
upstream photographer_service {
    server photographer-service:8001;
}

upstream guest_service {
    server guest-service:8002;
}
```

These names resolve automatically within Docker's internal network.

---

## Rate Limiting

Limits are defined in `nginx.conf` and applied per client IP:

| Zone | Limit | Applied To | Burst |
|---|---|---|---|
| `event_limit` | 20 req/s | `/auth/` and `/events/` | 10 |
| `guest_limit` | 200 req/s | `/guest/` | 50 |
| `match_limit` | 5 req/min | `/match/` | 2 |

The `/match/` route has the strictest limit because selfie matching runs GPU inference. Without this limit, a single client could flood the GPU and block all other guests.

When a client exceeds the limit, the gateway returns:
```json
HTTP 429
{"detail": "Too many requests. Slow down."}
```

---

## CORS

CORS headers are added by Nginx for requests from `http://localhost:3000`. For production, update the `$cors_origin` check in `nginx.conf`:

```nginx
# Current (development)
if ($http_origin ~* "^http://localhost:3000$") {

# Change to your production domain
if ($http_origin ~* "^https://scanme.yourdomain.com$") {
```

---

## Reloading Config Without Restart

If running in Docker, rebuild and restart the container. If running Nginx natively:

```bash
nginx -s reload
```

---

## Troubleshooting

**`connection refused` on port 80**
- Check the container is running: `docker ps`
- Check the port mapping: should show `0.0.0.0:80->80/tcp`

**`502 Bad Gateway`**
- The upstream service is not running or not reachable
- For local dev: make sure uvicorn is running on the correct port
- For local dev: make sure upstreams use `host.docker.internal` not `localhost`

**`listen 89` bug**
- If you see this in your config, change it to `listen 80` — Nginx will not receive any traffic on the wrong port

**Check Nginx logs**
```bash
docker logs <container_id>
```

---

## Related Services

| Service | Repo | Description |
|---|---|---|
| Photographer Service | `scanme-photographer` | Auth and event management on :8001 |
| Guest Service | `scanme-guest` | Gallery and selfie matching on :8002 |
| Ingestion Worker | `scanme-ingestion-worker` | Background worker — not routed through gateway |
| Frontend | `scanme-frontend` | Calls this gateway at `NEXT_PUBLIC_API_URL=http://localhost:80` |
