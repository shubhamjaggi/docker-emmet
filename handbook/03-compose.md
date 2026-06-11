# Docker Compose — Core Scenarios, Dependencies & Healthchecks

Real apps are rarely one container — you've got your app *plus* a database, a cache, maybe a worker. **Compose** describes all of them in one `docker-compose.yml` and starts the whole set with `docker compose up`. Each entry under `services:` becomes a container. Two things Compose gives you for free:

- **A shared network** — every service can reach the others using the *service name* as a hostname (the app connects to `db:5432`, not an IP). No manual networking needed.
- **Lifecycle management** — `up` starts everything, `down` tears it all down, and `depends_on` controls start order.

## App + Postgres + Redis

A typical web app: your code, a Postgres database, and a Redis cache. Note how `app` reaches the others by service name (`db`, `cache`) in its connection strings, and how `depends_on` makes it wait until the database is actually *ready* (not just started) before booting.

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    env_file: .env
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/appdb
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  # runs on first boot
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  pg_data:
  redis_data:
```

## Background worker (same image, different command)

A worker (queue consumer, cron processor) usually runs the *same code* as your API but with a different entry command. Reuse the same `build:` and just override `command:` — no second image needed.

```yaml
services:
  api:
    build: .
    command: gunicorn app:app
    ports:
      - "8000:8000"

  worker:
    build: .
    command: celery -A app.celery worker --loglevel=info
    depends_on:
      - cache
```

## Nginx reverse proxy in front of app

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app

  app:
    build: .
    expose:
      - "3000"       # internal only, nginx reaches it; host cannot
```

`nginx/nginx.conf`:
```nginx
events {}

http {
  upstream app {
    server app:3000;
  }

  server {
    listen 80;

    location / {
      proxy_pass         http://app;
      proxy_set_header   Host $host;
      proxy_set_header   X-Real-IP $remote_addr;
    }
  }
}
```

## Profiles — optional services

Profiles let you keep optional, non-default services (admin UIs, mail catchers, debug tools) in the same file but **off by default**. They only start when you explicitly ask for their profile, so the everyday `docker compose up` stays lean.

```yaml
services:
  app:
    build: .

  db:
    image: postgres:16-alpine
    # no `profiles:` key at all → always started (a service with no profile is in every run)

  pgadmin:
    image: dpage/pgadmin4
    profiles: [tools]        # only when: docker compose --profile tools up

  mailhog:
    image: mailhog/mailhog
    profiles: [dev]
```

```bash
docker compose --profile tools up
docker compose --profile dev --profile tools up
```

---

## Healthchecks

A **healthcheck** is a command Docker runs periodically inside the container to confirm the app is genuinely working — not just that the process started. A database process can be "up" for a second or two before it's actually ready to accept connections. Healthchecks turn "is the container running?" into "is the app *ready to serve*?", which is what `depends_on: condition: service_healthy` (below) waits on.

### In Dockerfile

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

> The check command must exist *inside* the image. `curl` is **not** in `-slim`, scratch, or distroless bases (Alpine has only busybox `wget`), so a `curl` healthcheck on those silently fails forever. Pick a probe your base actually ships, or install one — see the gotcha note in [07-stacks.md](07-stacks.md).

### In Compose (overrides Dockerfile)

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s     # grace period before first check
```

What each field buys you, and why the defaults aren't enough:

- **`interval`** — how often to re-run the check once it's going. Too frequent wastes CPU; too sparse means slow detection of a hang. 10–30s is typical.
- **`timeout`** — how long to wait for one check before calling it a failure. Must exceed your endpoint's worst-case response time, or a momentarily slow-but-healthy app gets marked unhealthy.
- **`retries`** — consecutive failures before the container flips to `unhealthy`. Set above 1 so a single transient blip (a GC pause, a brief lock) doesn't trigger a false alarm.
- **`start_period`** — the crucial one for slow starters. During this initial window, failing checks **don't count** against `retries` — they just don't mark the container healthy yet. Without it, an app that takes 20s to warm up (JVM, large migrations) would burn through its retries and be declared `unhealthy` before it ever finished booting, and anything gated on `service_healthy` would give up. Size it to your real cold-start time.

## `depends_on` conditions

`depends_on` controls **start order**. By default it only waits for the dependency's container to *start* — which often isn't enough, because "started" ≠ "ready." Pair it with a healthcheck and `service_healthy` to wait for genuine readiness and avoid the classic "app crashes on boot because the database wasn't accepting connections yet" race.

**What the race looks like:** with a plain `depends_on: [db]` (no healthcheck), Postgres's container starts in ~0.5s but doesn't accept connections for another 2–3s while it initializes. Your app, which started the instant the container did, fires its first query into that gap and dies:

```
app-1  | Error: connect ECONNREFUSED 172.18.0.3:5432
app-1  | Postgres is starting up — the database system is not yet accepting connections
app-1 exited with code 1
```

Switching to `condition: service_healthy` makes Compose hold the app back until `pg_isready` passes, and the error disappears.

```yaml
depends_on:
  db:
    condition: service_healthy    # waits for healthcheck to pass (truly ready)
  cache:
    condition: service_started    # just waits for container to start
  migrations:
    condition: service_completed_successfully   # waits for a one-shot job to finish 0-exit
```

## One-shot migration container

Database migrations should run **once**, to completion, *before* the app starts — not on every app replica. Model the migration as its own service that runs and exits, then gate the app on `service_completed_successfully` so it only boots after migrations finish cleanly.

```yaml
services:
  migrations:
    build: .
    command: python manage.py migrate
    depends_on:
      db:
        condition: service_healthy
    restart: "no"        # critical: a migration must NOT auto-restart

  app:
    build: .
    depends_on:
      migrations:
        condition: service_completed_successfully
```

Why `restart: "no"` is essential here: a migration container is *meant* to exit as soon as it finishes (exit 0). Under the usual `unless-stopped`/`always` policy, Docker would see that exit as "the container stopped" and dutifully restart it — re-running your migrations in an endless loop, and `service_completed_successfully` would never settle. `"no"` tells Docker the clean exit is the desired end state, not a failure to recover from.

> One nuance worth knowing: Compose's *default* restart policy (no `restart:` key at all) is already `no`, so a migration with no policy won't loop. The trap is teams that apply one restart policy to **every** service — `restart: unless-stopped` on a migration is what loops it. Set `restart: "no"` explicitly so intent is clear and a shared/templated policy can't override it.

---

## Compose hygiene

A few small things that quietly date a Compose file or undercut its isolation.

### Drop the obsolete `version:` key

```yaml
version: "3.8"     # ❌ delete this line
services:
  ...
```

The top-level `version:` mattered in the old Compose V1 (the Python tool). In **Compose V2** — the Go rewrite that's been the default since Docker Compose v2.0 — it is **silently ignored**: it does not gate features, validate a schema, or change behaviour. Its only effect today is to mislead readers into thinking they must bump it to use newer syntax (they don't). Remove it.

### Don't use `links:` — services already have DNS

```yaml
app:
  links:              # ❌ legacy, deprecated
    - db
```

`links:` predates Docker networking. It injected connection details as environment variables and `/etc/hosts` entries, creating tight coupling and leaking the linked service's env. Modern Compose puts every service on a shared network with **automatic service-name DNS** — `app` can already reach `db:5432` with no `links:` at all. For start ordering, use `depends_on:` (above). For *who can reach whom*, segment with networks — see [04-volumes-networking.md](04-volumes-networking.md#custom-network-isolate-groups-of-services).
