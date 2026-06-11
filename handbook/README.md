# The Docker Handbook

A practical, copy-paste reference covering the Docker scenarios you'll hit in real projects. Every snippet comes with a plain-language explanation of *why* it's written that way, so it's useful whether this is your first Dockerfile or your five-hundredth.

This handbook is the companion to the [`/docker-emmet`](../README.md) linter: **every rule the linter reports links back to the chapter here that explains it in depth.** The linter tells you *what* and *where*; the handbook tells you *why*. The full rule-to-chapter map is in [RULES.md](../RULES.md).

## New to Docker? Start here

A few core ideas make everything else click:

- **Image** — a frozen, read-only snapshot of a filesystem plus the command to run. Think of it as a "class." You build it once from a `Dockerfile`.
- **Container** — a running instance of an image. Think "object." You can start many containers from one image. Containers are **disposable**: when one stops, anything it wrote inside is gone unless you stored it in a volume.
- **Dockerfile** — the recipe that describes how to build an image, step by step (`FROM`, `COPY`, `RUN`, …). See [01-dockerfile.md](01-dockerfile.md).
- **Layer** — each instruction in a Dockerfile produces one cached layer. Docker reuses unchanged layers on rebuild, which is why instruction *order* matters so much for speed.
- **Volume** — persistent storage that lives outside the container, so data (databases, uploads) survives restarts. See [04-volumes-networking.md](04-volumes-networking.md).
- **Compose** — a YAML file (`docker-compose.yml`) that runs several containers together (app + database + cache) with one command, `docker compose up`. See [03-compose.md](03-compose.md).
- **Registry** — a server (Docker Hub, GHCR, ECR) that stores images so other machines and CI can `pull` them. See [06-production.md](06-production.md).

The typical loop: **write a Dockerfile → `docker build` an image → run it as a container (often via Compose) → push the image to a registry → deploy.**

## Topic map

| File | Topics |
|---|---|
| [01-dockerfile.md](01-dockerfile.md) | Fundamentals, multi-stage builds (Node/Python/Go/Java/Kotlin/GraalVM), base image choices |
| [02-local-dev.md](02-local-dev.md) | Hot-reload, override files, exec into containers, debugging crashes |
| [03-compose.md](03-compose.md) | Core scenarios, service dependencies, healthchecks |
| [04-volumes-networking.md](04-volumes-networking.md) | Volume types, custom networks, cross-compose communication |
| [05-config-secrets.md](05-config-secrets.md) | .env files, ARG vs ENV, Docker secrets, interpolation |
| [06-production.md](06-production.md) | Hardening, log rotation, registry & image management, multi-platform builds |
| [07-stacks.md](07-stacks.md) | Full stack snippets (Node, Python, Go, Java/Kotlin) + essential commands |
| [08-security.md](08-security.md) | Container isolation model, the docker-socket/privileged/capabilities/seccomp escapes, build-time secrets, image scanning |
