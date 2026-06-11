# Config & Secrets

The golden rule: **keep configuration out of your image.** The same image should run in dev, staging, and prod, with only the injected config differing. That means no hardcoded URLs, hostnames, or — especially — passwords baked into the Dockerfile. This file covers how to feed config in from outside, and how to handle the sensitive bits safely.

## .env file (default, auto-loaded by Compose)

The simplest approach: put `KEY=value` pairs in a `.env` file. Compose reads it automatically and can inject the values into your containers. Great for non-secret config and local dev.

```
# .env
DATABASE_URL=postgres://user:pass@db:5432/mydb
SECRET_KEY=changeme
```

```yaml
services:
  app:
    env_file: .env                    # inject all vars from file
    environment:
      EXTRA_VAR: "hardcoded"          # additional inline vars
```

**Never commit `.env`.** Commit `.env.example` instead.

## Build-time ARG vs runtime ENV

These two are easy to confuse. `ARG` exists **only while the image is being built** and is gone afterward — use it for build-time choices like a version number or base-image tag. `ENV` is **baked into the image and visible to the running container** — use it for runtime configuration.

```dockerfile
ARG BUILD_VERSION=unknown       # only available during build
RUN echo $BUILD_VERSION > /app/version.txt

ENV RUNTIME_VAR=value           # available to running container
```

```bash
docker build --build-arg BUILD_VERSION=$(git rev-parse --short HEAD) .
```

**Never put secrets in either** — whatever they touch gets baked into the image, where it stays recoverable. The two leak in different ways:

- **`ENV` is the obvious case.** Its value is stored in the image's config and printed verbatim by `docker inspect` (the `Config.Env` array) and `docker history`. There's no hiding it.
- **`ARG` is subtler but still unsafe.** The variable itself isn't kept in the final image, but if a build step *writes it into the filesystem*, that value lives forever in the read-only layer — even though the arg is "gone afterward."

Suppose a build step bakes the key into a config file:

```dockerfile
ARG API_KEY
RUN ./configure --key=$API_KEY    # writes the key into /app/config.json
```

Anyone with the image can read it straight back out of that layer — no build args or history needed:

```bash
docker run --rm myapp:latest cat /app/config.json
# { "apiKey": "sk_live_9f3a8b2c7d..." }   ← your "secret", in plain text, forever
```

That's why real secrets must be injected at *runtime* (env from the orchestrator, or the file-based secrets below) and never appear in any build instruction.

## Docker secrets (Swarm / Compose v2.x)

For genuinely sensitive values (DB passwords, API keys), env vars are leaky — they show up in `docker inspect`, in logs, and in child-process environments. **Docker secrets** instead mount the value as a file at `/run/secrets/<name>`. Your app reads the file; the secret never becomes an environment variable. Many official images support a `_FILE` suffix (e.g. `POSTGRES_PASSWORD_FILE`) for exactly this.

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt   # or `external: true` for Swarm

services:
  db:
    secrets: [db_password]
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
```

Secrets are mounted as files at `/run/secrets/<name>` — not exposed as env vars (safer, not in `docker inspect`).

## Interpolation and defaults

```yaml
environment:
  PORT: ${PORT:-3000}              # use $PORT or fall back to 3000
  LOG_LEVEL: ${LOG_LEVEL:?LOG_LEVEL must be set}   # fail if unset
```

These two forms encode two different intents, and choosing the right one is the point:

- **`${PORT:-3000}`** — *optional with a sane default.* If `PORT` isn't set, use `3000`. Use this for config that has a reasonable fallback, so a fresh checkout runs with zero setup.
- **`${LOG_LEVEL:?...}`** — *required, fail loudly.* If `LOG_LEVEL` isn't set, Compose aborts immediately with your message instead of starting. Use this for values that have *no* safe default (a production database URL, an API key) — you want a clear startup error now, not a container that boots with a blank or wrong value and fails mysteriously later. Failing fast at startup turns a silent misconfiguration into an obvious one.
