# Problem 4 – Investigation and Fixes Report

## 1. Problems Found

### Critical: Wrong API port in Nginx (root cause of “inaccessible” API)

- **Symptom:** API sometimes returns 502 Bad Gateway or connection refused.
- **Cause:** The API listens on port **3000** (`app.listen(3000)` in `api/src/index.js`), but Nginx was configured to proxy to `http://api:3001`. Nginx was always talking to the wrong port, so the API was effectively unreachable through Nginx.

### Startup race (root cause of “unreliable” behavior)

- **Symptom:** Intermittent failures, especially on first requests or right after `docker compose up`.
- **Cause:** `depends_on` only waits for containers to **start**, not for Postgres or Redis to **accept connections**. The API often started before:
  - Postgres finished initializing (including `init.sql` and `max_connections`).
  - Redis was ready to accept connections.  
  So the API could hit DB/Redis connection errors on early requests.

### No recovery from failures

- **Cause:** No `restart` policy. If the API (or any service) crashed, it stayed down until someone restarted it manually.

### No readiness signal

- **Cause:** There were no healthchecks. Compose had no way to know when Postgres, Redis, or the API were actually ready. So Nginx could start and send traffic to the API before it was listening, and the API could start before DB/Redis were ready.


## 2. How They Were Diagnosed

1. **Port mismatch:** Compared Nginx config (`proxy_pass http://api:3001`) with the API code (`app.listen(3000)`).
2. **Startup race:** Noted that `depends_on` has no `condition` and that Postgres/Redis have no healthchecks, so “started” ≠ “ready”.
3. **Reliability:** Noted absence of `restart` and healthchecks in `docker-compose.yml`.


## 3. Fixes Applied

### 3.1 Nginx (`nginx/conf.d/default.conf`)

- Changed `proxy_pass` from `http://api:3001` to `http://api:3000` so it matches the API.
- Added timeouts and upstream error handling:
  - `proxy_connect_timeout 5s`
  - `proxy_read_timeout 30s`
  - `proxy_next_upstream error timeout`  
  So Nginx fails fast and can retry on another upstream if you add more API replicas later.

### 3.2 Docker Compose (`docker-compose.yml`)

- **Healthchecks (readiness):**
  - **postgres:** `pg_isready -U postgres` (interval 5s, retries 5, start_period 10s).
  - **redis:** `redis-cli ping` (interval 5s, retries 5).
  - **api:** `curl -f http://localhost:3000/status` (interval 5s, retries 5, start_period 10s).  
  So each service is only considered “up” when it actually responds.

- **Startup order (depends_on with condition):**
  - **api** depends on `postgres` and `redis` with `condition: service_healthy`.
  - **nginx** depends on **api** with `condition: service_healthy`.  
  So Nginx only starts after the API is healthy, and the API only starts after Postgres and Redis are healthy.

- **Restart policy:** `restart: unless-stopped` for all services so they recover after crashes.

### 3.3 API image (`api/Dockerfile`)

- Added `RUN apk add --no-cache curl` so the API container can run the healthcheck (`curl -f http://localhost:3000/status`).


## 4. Monitoring and Alerts to Add (production)

- **Health endpoint:** Expose `/status` (or a dedicated `/health`) and monitor it (every 10–30s). Alert if down or slow.
- **Dependency checks:** A richer health that checks DB and Redis (simple query + PING) and returns 503 if either fails; alert on 503.
- **Metrics:** Instrument API (request count, latency, errors) and export to Prometheus; alert on error rate and latency percentiles.
- **Logs:** Centralize logs (ELK/Loki); alert on repeated connection errors or stack traces.
- **Infra:** Monitor container restarts, Postgres connections, and Redis memory/connections.


## 5. How to Prevent This in Production

- **Config vs code:** Keep ports and hostnames in one place (e.g. env or config); validate at deploy that proxy targets match actual service ports (e.g. in CI or a startup check).

- **Readiness, not just “started”:** Always use healthchecks and wire `depends_on` (or K8s readiness) so nothing receives traffic until it and its dependencies are ready.

- **Restart and orchestration:** Use `restart` (or equivalent in your orchestrator) and consider multiple API replicas behind the proxy for availability.

- **Startup and resilience:** Optionally add connection retries/backoff in the API for DB and Redis so brief unavailability doesn’t crash the process.

- **Pre-production checks:** In CI/CD, run the stack (e.g. `docker compose up`), hit `/api/users` and `/status` until success or timeout, then tear down. Fail the build if the API is not reachable or returns 5xx.
