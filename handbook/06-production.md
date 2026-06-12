# Production Hardening, Registry & Multi-Platform Builds

> **Linter rules explained in this chapter:** DF-02, DF-15, CM-03, CM-10, CM-11, CM-13, CM-14. Each is tagged inline at the section that covers it.

## Production Hardening

The theme of this section is **least privilege**: give the container only what it needs, so that if your app is ever compromised, the blast radius is small. Each setting below closes off one avenue an attacker could otherwise use.

### Non-root user (DF-02)

By default a container runs as `root`. If an attacker breaks out of your app, they're root inside the container — and a container root that escapes is root-adjacent on the host. Running as an unprivileged user is the single highest-value hardening step.

**The difference, made concrete** — suppose an attacker gets command execution via your app and tries to tamper with a system file:

```bash
# Container running as root (default):
$ id
uid=0(root) gid=0(root)
$ echo 'evil' >> /etc/passwd        # succeeds — full control of the container

# Container running as USER app:
$ id
uid=101(app) gid=101(app)
$ echo 'evil' >> /etc/passwd
sh: can't create /etc/passwd: Permission denied   # blocked
```

Same exploit, very different outcome. The unprivileged user can't modify system files, install packages, or bind privileged ports — so the foothold is far less useful.

```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

The `--system` flag creates a service account (low UID, no password, no home dir, no login shell) rather than a human-login account — appropriate for something that only ever runs a process.

**Or pin a fixed UID** — important the moment you mount a volume:

```dockerfile
RUN addgroup --gid 1001 app && adduser --uid 1001 --gid 1001 --disabled-password app
USER 1001:1001
```

Why pin it? Files on a mounted host volume are owned by a *numeric* UID, and that number must line up on both sides. If you let the system auto-assign (say it picks `100` today, `101` after a base-image bump), a volume written by yesterday's container can become unreadable to today's — the classic "Permission denied" on a bind mount. A fixed `1001` you also `chown` the host directory to makes ownership deterministic across rebuilds and machines. `--disabled-password` just removes the ability to log in as the account, since a service never needs to.

### Read-only filesystem

Most apps never need to write to their own filesystem at runtime. Making it read-only means an attacker can't drop a malicious binary or tamper with your code. Carve out a writable `tmpfs` only for the few paths that genuinely need it (like `/tmp`).

```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp         # app can still write temp files
```

### Resource limits (CM-13)

Caps on CPU and memory prevent one container from starving the rest of the host — whether from a memory leak, a runaway loop, or a denial-of-service attack. Without limits, a single bad container can take down everything on the machine.

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          memory: 256M
```

> `deploy.resources` works with `docker compose up` — no Swarm required.

### Drop Linux capabilities (CM-03)

Linux splits root's powers into ~40 fine-grained "capabilities" (mount filesystems, change ownership, etc.). Containers get a broad default set, but a typical web app needs none of them. Drop `ALL` and add back only the rare one you actually use (e.g. `NET_BIND_SERVICE` to bind a port below 1024).

```yaml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE    # only if binding port < 1024
```

### No new privileges

This flag stops a process from *gaining* more privileges than it started with (e.g. via `setuid` binaries). It closes a common privilege-escalation path, so a compromised process can't promote itself to root.

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
```

### Restart policy (CM-10)

Tells Docker what to do when a container exits. `unless-stopped` restarts it on crash or host reboot, but respects a manual stop — a sensible default for long-running services. Use `on-failure` for jobs that should only retry on error, and `no` for one-shot tasks.

```yaml
services:
  app:
    restart: unless-stopped    # or: always | on-failure | no
```

> Prefer `unless-stopped` over `always`: both survive crashes and reboots, but `always` also restarts a container you deliberately `docker compose stop`, making it awkward to take a service offline. And always set `restart: "no"` *explicitly* on migrations/seeders — see the one-shot trap in [03-compose.md](03-compose.md#one-shot-migration-container-cm-08).

### Log rotation (CM-14)

Docker's default logging driver is `json-file` with **no size limit**. A chatty service — one log line per request, query, or heartbeat — will grow that file until it fills the host disk, at which point *every* container on the machine fails because the daemon can't write. The failure is silent right up until it happens. Cap it:

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"     # rotate at 10 MB
        max-file: "3"       # keep 3 files → 30 MB ceiling per container
```

In real production, point the driver at a centralised sink instead (`awslogs`, `gcplogs`, `fluentd`, `loki`) so logs survive the container and are searchable.

---

## Registry & Image Management

A **registry** is the warehouse for your images (Docker Hub, GitHub Container Registry, AWS ECR, etc.). You `push` an image after building so that other machines, teammates, and your deployment pipeline can `pull` the exact same artifact. The `name:tag` you push *is* the image's identity, so tagging discipline matters.

### Tag conventions (DF-15, CM-11)

A **tag** is the label after the colon (`app:1.2.3`). The trap to avoid: `latest` is just a default name, not a guarantee — it silently moves to whatever was pushed last. For anything you deploy, prefer an **immutable** tag (a semver release or a git-SHA) so a given tag always means the exact same image. That makes rollbacks and "what's actually running?" answerable.

**The trap in practice:** say production runs `app:latest`. A teammate pushes a broken build and tags it `latest`. Now:

```
10:00  prod is running app:latest  (= the good build, sha-a1b2c3d)
10:05  teammate: docker push app:latest   (= broken build, sha-f9e8d7c)
10:10  a prod node restarts, pulls app:latest → gets the BROKEN build
10:11  you try to roll back... to what? "latest" no longer points at the good one
```

You can't roll back, because the name that *was* good now points at the bad image and the good SHA is buried. With an immutable scheme you'd deploy `app:sha-a1b2c3d`, and rolling back is just "redeploy `app:sha-a1b2c3d`" — the tag can never have meant anything else.

```bash
myregistry.io/org/app:1.2.3          # semver release
myregistry.io/org/app:1.2            # minor alias
myregistry.io/org/app:latest          # latest stable
myregistry.io/org/app:sha-a1b2c3d    # immutable git-sha tag (preferred in CD)
```

### Build and push

```bash
docker build -t myregistry.io/org/app:1.2.3 .
docker push myregistry.io/org/app:1.2.3

# retag and push alias
docker tag myregistry.io/org/app:1.2.3 myregistry.io/org/app:latest
docker push myregistry.io/org/app:latest
```

### Build with layer cache from registry (CI speedup)

**The problem this solves:** layer caching (the whole basis of [01-dockerfile.md](01-dockerfile.md)) lives in the *local* Docker daemon. CI runners are usually fresh, ephemeral machines with an empty cache, so every CI build starts stone cold — reinstalling all dependencies from scratch even when nothing about them changed. `--cache-from` lets the build seed its cache by *pulling layers from an image already in the registry*, restoring the speedup on a machine that has never built this project before.

```bash
docker build \
  --cache-from myregistry.io/org/app:latest \
  --tag myregistry.io/org/app:$SHA \
  .
```

### BuildKit inline cache (pushes cache metadata with image)

For `--cache-from` to actually hit, the pulled image must *carry* the cache metadata that maps each layer to the instruction that built it. A normal `docker push` strips that metadata. `BUILDKIT_INLINE_CACHE=1` embeds it into the pushed image, so the *next* CI run's `--cache-from` can reuse those layers instead of treating them as opaque. In short: this flag is what makes the `--cache-from` above effective build-over-build.

```bash
DOCKER_BUILDKIT=1 docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --cache-from myregistry.io/org/app:latest \
  -t myregistry.io/org/app:latest .
```

### Useful image commands

```bash
docker images                          # list local images
docker rmi myapp:old                   # remove image
docker image prune -f                  # remove dangling (untagged) images
docker image prune -a                  # remove all unused images
docker inspect myapp:latest            # full metadata JSON
docker history myapp:latest            # layer sizes
```

---

## Multi-Platform Builds

An image built on an Apple-Silicon (ARM) Mac won't run on a typical x86 (amd64) cloud VM, and vice versa — the CPU architecture is baked in. `buildx` builds for several architectures at once and publishes them under a single tag as a *manifest list*, so each machine automatically pulls the variant that matches its CPU.

Build for `linux/amd64` and `linux/arm64` in one shot (M-series Mac ↔ cloud VMs):

```bash
# one-time setup
docker buildx create --use --name multibuilder

# build and push both platforms
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag myregistry.io/org/app:latest \
  --push \
  .
```

The registry stores a manifest list; `docker pull` picks the right arch automatically.
