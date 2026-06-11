# Common Stack Snippets

These are complete, ready-to-adapt setups that combine the building blocks from the earlier files — multi-stage Dockerfile, Compose services, healthchecks, a non-root user, and dependency ordering — into the shapes you'll actually start projects from. Each one is a sensible default for its ecosystem; copy, rename, and tweak.

## Node.js API + Postgres + Redis

A standard web API: a multi-stage build (compile with full Node, ship on `slim` as a non-root user) plus a Postgres database and Redis cache. The app waits for Postgres to be *healthy* before booting.

```dockerfile
# Dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force   # prod-only deps — devDependencies stay out of the shipped image
COPY --from=builder --chown=app:app /app/dist ./dist
USER app
EXPOSE 3000
# node:20-slim has NO curl/wget — use Node's built-in fetch so the check actually runs
HEALTHCHECK --interval=10s --timeout=3s \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
```

> **Common gotcha:** `curl` and `wget` are *not* present in `-slim` (Debian) or scratch/distroless images, and only `wget` (busybox) is present in Alpine. A `HEALTHCHECK ... curl ...` on such a base silently fails forever, leaving the container `unhealthy` and anything gated on `service_healthy` stuck. Either use a runtime you already have (Node's `fetch`, a Python one-liner, the JVM's `wget` on Alpine) or `RUN apt-get install -y --no-install-recommends curl` in the image.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["3000:3000"]
    env_file: .env
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/appdb
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  pg_data:
  redis_data:
```

---

## Python (FastAPI) + Postgres + Celery worker

The same image runs three ways via different `command:` overrides: the web API (`uvicorn`), a Celery **worker** that processes background jobs off the Redis queue, and a Celery **beat** scheduler that enqueues periodic tasks. One build, three roles.

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
COPY . .
RUN addgroup --system app && adduser --system --ingroup app app
USER app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      broker:
        condition: service_started

  worker:
    build: .
    command: celery -A app.celery worker --loglevel=info --concurrency=4
    env_file: .env
    depends_on:
      broker:
        condition: service_started

  beat:
    build: .
    command: celery -A app.celery beat --loglevel=info
    env_file: .env
    depends_on:
      broker:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      retries: 5

  broker:
    image: redis:7-alpine

volumes:
  pg_data:
```

---

## Go binary on distroless

Go compiles to a single static binary, so the runtime image needs essentially nothing else. Build with the full Go toolchain, then copy just the binary onto `distroless/static` — no shell, no OS, runs as `nonroot`. The result is a tiny, hard-to-attack image (often under 10 MB).

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

---

## Java/Kotlin Spring Boot + Postgres + Redis

Spring Boot reads config from `SPRING_*` environment variables, so the Compose file wires the database and Redis in without touching code. The `memory: 512M` limit matters here: combined with the `-XX:MaxRAMPercentage` flag in the Dockerfile (see [01-dockerfile.md](01-dockerfile.md)), the JVM sizes its heap to the container's limit instead of the host's RAM — which is what prevents Java containers from being OOM-killed. The override file adds DevTools hot-reload for local work.

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: runtime
    ports: ["8080:8080"]
    env_file: .env
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/appdb
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: secret
      SPRING_DATA_REDIS_HOST: cache
      SPRING_DATA_REDIS_PORT: 6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    deploy:
      resources:
        limits:
          memory: 512M        # JVM will use up to 75% = ~384 MB heap
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  pg_data:
  redis_data:
```

**Local dev override** — mount source for Spring DevTools hot-reload:

```yaml
# docker-compose.override.yml
services:
  app:
    build:
      target: builder         # JDK stage so DevTools can recompile
    volumes:
      - ./src:/app/src
      - ./build:/app/build
    environment:
      SPRING_DEVTOOLS_RESTART_ENABLED: "true"
    command: ["./gradlew", "bootRun"]
```

---

## Quick reference: essential commands

The commands you'll reach for daily, grouped by what you're trying to do — build images, start/stop the stack, look inside, and clean up. The `docker compose run --rm` form is especially handy: it spins up a one-off container for a single task (a migration, a seed script, a shell), then deletes it.

```bash
# build
docker build -t myapp:latest .
docker compose build --no-cache

# run
docker compose up -d
docker compose up --build            # rebuild before starting

# inspect
docker compose ps
docker compose logs -f app
docker stats                         # live CPU/memory

# exec
docker exec -it myapp-app-1 bash

# cleanup
docker compose down                  # stop + remove containers + networks
docker compose down -v               # also remove named volumes
docker system prune -af              # remove everything unused (images, containers, networks, volumes)

# one-off commands
docker compose run --rm app python manage.py createsuperuser
docker compose run --rm app npm run seed
```
