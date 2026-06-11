# Compose rules

_Loaded on demand by the docker-emmet skill. 18 rules (CM-01–CM-18); each lists a trigger, a production rationale, and a concrete fix._

---

## Security

**CM-01 | CRITICAL | Docker socket mounted — container can control the host**
Trigger: Any `volumes:` entry with the host path `/var/run/docker.sock`.
Rationale: The Docker socket is a direct, unauthenticated pipe to the Docker daemon running as root on the host. A container with socket access can: create new containers with `--privileged` or bind-mounted host filesystem, execute processes in other containers, pull and run arbitrary images, exfiltrate secrets from other containers' environments, and modify or destroy any running container or image. This is a complete container escape — functionally equivalent to giving the container unrestricted root access to the host. Even if the container runs as non-root, socket access bypasses all of that. This is the most critical Docker security misconfiguration.
Fix: Remove the socket mount. For CI runners that genuinely need Docker-in-Docker capability, use `docker:dind` with TLS, or rootless Docker, or a purpose-built solution like Kaniko or Buildah. For monitoring tools (Portainer, cAdvisor), evaluate whether read-only socket access plus a hardened, minimal image is acceptable in your threat model — but never in production application stacks.

**CM-02 | CRITICAL | privileged: true — full host access**
Trigger: Any service with `privileged: true`.
Rationale: `privileged: true` grants the container every Linux capability, disables the seccomp syscall filter, removes AppArmor/SELinux confinement, and gives the container access to all host devices. A privileged container can mount host filesystems, load kernel modules, modify network interfaces, read `/proc` for all host processes, and trivially escape to the host. There is no meaningful isolation remaining. This flag exists for specific low-level system tools (Docker-in-Docker, device drivers, VM managers) — it should never appear in an application service.
Fix: Remove `privileged: true`. If the service genuinely requires elevated access, identify the exact Linux capability needed and add only that via `cap_add:`. Common legitimate additions: `NET_BIND_SERVICE` (bind ports below 1024), `SYS_NICE` (process scheduling). Almost nothing needs `SYS_ADMIN` or `NET_ADMIN` in an application container.

**CM-03 | ERROR | Dangerous Linux capability added**
Trigger: `cap_add:` containing any of: `ALL`, `SYS_ADMIN`, `NET_ADMIN`, `SYS_PTRACE`, `SYS_MODULE`, `SYS_RAWIO`, `SYS_BOOT`, `SYS_TIME`, `DAC_READ_SEARCH`, `LINUX_IMMUTABLE`, `BPF`, `PERFMON`, `MAC_ADMIN`, `MAC_OVERRIDE`.
Do NOT trigger on capabilities that Docker grants by default (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `FSETID`, `KILL`, `SETGID`, `SETUID`, `SETPCAP`, `NET_BIND_SERVICE`, `NET_RAW`, `SYS_CHROOT`, `MKNOD`, `AUDIT_WRITE`, `SETFCAP`) — adding one of these is a redundant no-op, not an escalation, and flagging it would be a false positive.
Rationale: Containers start with a limited default capability set; the rule catches additions *beyond* that set, which expand the attack surface.
- `ALL`: adds every capability at once — effectively equivalent to `privileged: true`.
- `SYS_ADMIN`: the single most dangerous capability; enables mounting filesystems, manipulating namespaces, and dozens of privileged kernel operations. Historically called "the new root."
- `NET_ADMIN`: full network stack control — reconfigure interfaces, add routes, manipulate iptables, ARP-spoof other containers.
- `SYS_PTRACE`: inspect and inject into any process, including processes in other containers sharing the kernel.
- `SYS_MODULE`: load/unload kernel modules — a trivial path to complete kernel compromise.
- `SYS_RAWIO` / `BPF` / `PERFMON`: low-level memory, I/O port, and tracing access usable to read kernel memory or escape.
Fix: Remove the capability. The correct hardening pattern is to drop everything and add back only what the app proves it needs:
```yaml
cap_drop: [ALL]
cap_add: [NET_BIND_SERVICE]   # only if binding a port below 1024 as non-root
```
If `SYS_ADMIN` is genuinely required (e.g. FUSE mounts), isolate that function in a dedicated sidecar and apply compensating controls rather than granting it to the whole app.

**CM-04 | ERROR | Secret hardcoded in environment block**
Trigger: Under `environment:` (map or list form), a key (case-insensitive) containing `PASSWORD`, `SECRET`, `TOKEN`, `KEY`, `CREDENTIAL`, `API_`, `PRIVATE`, `AUTH`, `PASSPHRASE`, `APIKEY`, `PWD` — with a value that is a literal string, not an interpolation expression (`${VAR}`, `${VAR:-default}`, `${VAR:?error}`).
Do NOT trigger on `_FILE` suffix keys (e.g. `POSTGRES_PASSWORD_FILE`) — these are the correct pattern. Use judgment: do not flag clearly non-sensitive keys where the substring is incidental (e.g. `PUBLIC_KEY_URL`, `KEYCLOAK_REALM`, `MONKEY_PATCH`).
Rationale: Hardcoded secrets in Compose files are committed to version control (exposing them to everyone with repo access), visible in `docker inspect <container>` (no authentication needed beyond Docker socket access), logged by many container log aggregators, and inherited by every child process. A leaked Compose file equals leaked credentials.
Fix: Use one of these approaches, in order of preference:
1. Docker secrets + `_FILE` convention: `POSTGRES_PASSWORD_FILE: /run/secrets/db_password` (the value never appears in a file or env var).
2. Environment interpolation: `DATABASE_PASSWORD: ${DATABASE_PASSWORD:?must be set}` — value comes from the shell or a `.env` file that is gitignored.
3. External secret manager (Vault, AWS SSM, GCP Secret Manager) with a secrets-injector sidecar.

**CM-05 | WARN | pid: host or network_mode: host reduces isolation**
Trigger: Any service with `pid: host` or `network_mode: host`.
Rationale:
- `pid: host`: the container shares the host's PID namespace. It can see, inspect, and signal every process on the host — including processes in other containers and privileged OS processes. Tools like `ps`, `kill`, and `strace` inside the container now operate on the full host.
- `network_mode: host`: the container shares the host's full network stack. No network namespace isolation, no Compose service-name DNS, any port the container binds is immediately accessible on the host's external interfaces. Unlike `ports:` mapping, there is no NAT layer or access control.
Both have legitimate uses (network diagnostics tools, legacy apps with many dynamic ports) but significantly reduce isolation. They should be intentional, documented, and never present in production application services without a clear written justification.
Fix: Use standard networking and remove the flag if it was added out of habit. If `network_mode: host` is needed for performance (high-throughput networking, WebRTC), document it explicitly. If `pid: host` is needed for a monitoring tool, consider whether the tool can instead be given a targetted capability.

**CM-06 | WARN | Seccomp filter disabled**
Trigger: `security_opt:` containing `seccomp:unconfined` or `seccomp=unconfined`.
Rationale: Docker's default seccomp profile blocks approximately 44 syscalls that have no legitimate use in most applications but are useful for container-escape exploits — including `ptrace`, `reboot`, `kexec_load`, `mount`, `pivot_root`, and others. Disabling it (`seccomp:unconfined`) removes this kernel-level defense layer, leaving the container exposed to kernel exploit chains that the default profile would block. The common reason for disabling seccomp is that a tool (debugger, profiler) needs a specific syscall — the correct fix is to override only that syscall, not disable all filtering.
Fix: Remove `seccomp:unconfined`. If a specific tool requires a syscall blocked by the default profile, create a custom seccomp JSON profile that allows only that syscall and apply it via `security_opt: ["seccomp=custom-profile.json"]`.

---

## Reliability & startup

**CM-07 | WARN | depends_on without service_healthy when a healthcheck is defined**
Trigger: A service lists another service under `depends_on` using the simple list form (`depends_on: [db]`) or with `condition: service_started`, AND that dependency service defines a `healthcheck:` block.
Rationale: `condition: service_started` (the default) only confirms that the dependency container's main process has launched — not that the application inside is ready to accept connections. A Postgres container is "started" within ~0.5 seconds but does not accept queries for another 1–3 seconds while it runs crash recovery and initialises shared memory. During that window, any app that connects immediately receives `ECONNREFUSED` and crashes. This is the single most common Docker Compose "works sometimes, fails on first try" bug.
Fix: If the dependency defines a healthcheck, use `condition: service_healthy`:
```yaml
depends_on:
  db:
    condition: service_healthy     # waits for pg_isready to pass — actually ready
```
`condition: service_started` is appropriate only for services with no healthcheck.

**CM-08 | ERROR | One-shot service set to auto-restart**
Trigger: A service whose `command:` or `entrypoint:` contains any of: `migrate`, `migration`, `flyway`, `liquibase`, `alembic`, `db:migrate`, `knex migrate`, `prisma migrate`, `seed`, `init-db`, `setup` — AND has an active restart policy: `restart: always`, `restart: unless-stopped`, or `restart: on-failure` (including the long form under `deploy.restart_policy` with a `condition` other than `none`).
Note: Compose's default restart policy (when no `restart:` key is present) is `no`, so a one-shot service with *no* restart key will not loop — do not flag that case as an error. The dangerous case, which is common because teams apply one restart policy to every service, is an *explicit* restarting policy on a job meant to exit once.
Rationale: One-shot jobs (migrations, seeders, init scripts) run once and exit 0. With an auto-restart policy, Docker treats that clean exit as "the container stopped" and relaunches it — re-running your migrations in an endless loop. Worse, any service gated on `condition: service_completed_successfully` never settles, because the dependency never reaches a completed state. This can silently loop for hours with no obvious error.
Fix: Set `restart: "no"` explicitly on every one-shot service (even though it is the default, the explicit value documents intent and overrides any shared/templated policy):
```yaml
migrations:
  build: .
  command: python manage.py migrate
  restart: "no"
  depends_on:
    db:
      condition: service_healthy
```

**CM-09 | WARN | No healthcheck on database or cache service**
Trigger: A service using a `postgres`, `mysql`, `mariadb`, `mongo`, `mongodb`, or `redis` image with no `healthcheck:` block.
Rationale: Without a healthcheck, no dependent service can use `condition: service_healthy` — so startup ordering becomes unreliable (see CM-07). There is also no runtime signal when the database degrades (OOM kill, storage full, crash loop) short of the process exiting. The health status is visible in `docker compose ps` and exposed to orchestrators for automated remediation.
Fix — common patterns:
```yaml
# Postgres
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  interval: 5s
  timeout: 5s
  retries: 5
  start_period: 10s

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  retries: 5

# MySQL / MariaDB
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 5s
  retries: 5
```

**CM-10 | WARN | No restart policy on long-running service**
Trigger: A service that has `ports:`, `expose:`, or a `build:` key (indicating a long-running application) AND has no `restart:` key.
Do NOT trigger on services with `restart: "no"` (intentional), nor on one-shot jobs whose `command`/`entrypoint` matches the migration/seed/init patterns in CM-08 — those should stay one-shot, not be given an auto-restart policy.
Rationale: Without a restart policy, a container that crashes stays dead until someone manually intervenes — a `docker compose up` or server reboot. For a database, this means data is inaccessible; for a web service, it means downtime that persists past the incident. `restart: unless-stopped` is the standard: it restarts after crashes but respects `docker compose stop`, making controlled maintenance possible. Avoid `restart: always` — it also restarts after `docker compose stop`, making it hard to take a service offline temporarily.
Fix: Add `restart: unless-stopped` to every long-running service. Add `restart: "no"` explicitly to one-shot jobs to make intent clear.

**CM-11 | WARN | Unpinned image tag**
Trigger: `image: <name>` with no tag at all, or `image: <name>:latest`.
Rationale: `docker compose pull` or a fresh `docker compose up` in a new environment will pull whatever is currently tagged `latest`, which may be a different major version than the one tested. `latest` is not a stable reference — it is updated with every new release. Silent version changes are one of the most common "it stopped working and I don't know why" scenarios.
Fix: Pin every image to an explicit version: `postgres:16.4-alpine`, `redis:7.4-alpine`. Set up Dependabot or Renovate for automatic, reviewable version bump PRs.

---

## Data & resource safety

**CM-12 | WARN | Database service without a named volume**
Trigger: A service using a `postgres`, `mysql`, `mariadb`, `mongo`, `mongodb`, `elasticsearch`, or `cassandra` image with no `volumes:` entry whose source side is a named volume (a plain name like `pg_data`, not a path starting with `.` or `/`).
Rationale: Without a named volume, all database state lives in the container's writable layer, which is deleted when `docker compose down` removes the container. The next `docker compose up` starts with a completely empty database — no tables, no data, no accounts. This is the most common accidental data-loss scenario in development and staging. Named volumes survive `docker compose down` and are only removed when explicitly passed `-v`, making accidental wipes much harder.
Fix:
```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pg_data:/var/lib/postgresql/data   # named volume — survives down

volumes:
  pg_data:    # declare at top level
```

**CM-13 | INFO | No resource limits — container can consume unlimited host resources**
Trigger: A long-running service with `build:` or one that uses a database/cache image AND has no `deploy.resources.limits` block (neither `memory` nor `cpus`).
Note: Only flag services likely to be long-running and resource-intensive. Do NOT flag one-shot jobs (command/entrypoint matching the CM-08 migration/seed/init patterns), and do not flag every service in a clearly dev-only file.
Rationale: Without resource limits, a single misbehaving container — a memory leak, an infinite loop, a traffic spike — can consume all host memory or CPU, OOM-killing other containers or the host OS itself. `deploy.resources.limits` works with `docker compose` since Compose V2 (no Swarm required). Setting a memory ceiling prevents cascade failures; setting a CPU limit ensures fair sharing between services.
Fix:
```yaml
deploy:
  resources:
    limits:
      memory: 512M    # hard ceiling — container is OOM-killed if exceeded
      cpus: "1.0"     # max 1 CPU core
    reservations:
      memory: 256M    # minimum guaranteed allocation
```
Size based on your profiled usage, not a guess — start generous and tune down.

**CM-14 | INFO | No logging limits — disk can fill up silently**
Trigger: A long-running service (with `ports:`, `expose:`, or `build:`) with no `logging:` configuration. Do NOT flag one-shot jobs (CM-08 patterns), which produce bounded output and exit.
Rationale: Docker's default logging driver is `json-file` with no size limit. A verbose service logging every HTTP request, database query, or heartbeat can fill the host disk over days or weeks, causing all Docker containers on the host to fail when the daemon can't write logs. This failure mode is silent until it hits — there are no warnings that disk is filling.
Fix:
```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"    # rotate when file reaches 10 MB
    max-file: "3"      # keep 3 rotated files = max 30 MB total
```
In production, replace `json-file` with a centralised driver: `awslogs`, `gcplogs`, `fluentd`, or `loki` depending on your stack.

**CM-15 | INFO | Database or cache port exposed to the host**
Trigger: A service using a `postgres`, `mysql`, `mariadb`, `mongo`, `mongodb`, `redis`, or `rabbitmq` image that has a `ports:` mapping.
Rationale: A `ports:` mapping publishes the container port to the host's network interface — making it reachable from anywhere with network access to the host, not just from other containers. On a cloud VM with a public IP or a misconfigured firewall, a database exposed this way is reachable from the internet. Internal services in the same Compose project reach each other by service name on the Compose network without any `ports:` mapping.
Fix: Remove `ports:` from database and cache services. If you need local GUI access to the database during development, override in `docker-compose.override.yml` (which is gitignored) rather than putting it in the main Compose file that gets deployed.

**CM-16 | INFO | Obsolete top-level version key**
Trigger: A top-level `version:` key (e.g. `version: "3.8"` or `version: "2"`) at the start of the Compose file.
Rationale: The `version:` key was meaningful in Compose V1 (Python, deprecated). In Compose V2 (the current Go rewrite, the default since Docker Desktop 3.x and Docker Compose v2.0), it is silently ignored. It does not control which Compose features are available, does not validate the schema against a specific spec version, and has no effect on behaviour. Its presence misleads developers into thinking they need to bump it to access new features (they don't) and can confuse schema validators.
Fix: Remove the `version:` line entirely.

**CM-17 | WARN | links: is deprecated — use networks instead**
Trigger: Any service with a `links:` key.
Rationale: `links:` is a legacy Compose feature from Docker's pre-network era. It works by injecting environment variables into the linking container (e.g. `DB_PORT_5432_TCP_ADDR`, `DB_ENV_POSTGRES_PASSWORD`) and adding `/etc/hosts` entries. This creates tight coupling between services, leaks connection and environment details as env vars, and creates implicit start-order dependencies that are hard to reason about. Modern Compose automatically gives every service DNS resolution by service name on a shared network — no `links:` needed. Links were officially deprecated in Docker Engine 1.12.
Fix: Remove `links:`. Services can reach each other by service name directly (e.g. `db:5432`). If you need explicit start ordering, use `depends_on:`.

**CM-18 | INFO | All services on a single implicit network — consider segmentation**
Trigger: A Compose file with 4 or more services AND no top-level `networks:` block AND no `networks:` key on any service — AND at least one service appears to be a database or internal backend (uses a `postgres`, `mysql`, `mongo`, `redis`, `rabbitmq` image).
Rationale: By default, all services in a Compose file join one shared network where every service can reach every other service by name. For development this is fine. For production or staging, this means a compromised frontend container can directly dial the database — there is no network boundary stopping it. Defining explicit networks (`frontend`, `backend`) lets you put only the app and the reverse proxy on `frontend`, while the database lives exclusively on `backend` where only the app can reach it.
Fix:
```yaml
networks:
  frontend:
  backend:

services:
  nginx:  { networks: [frontend, backend] }
  app:    { networks: [backend] }
  db:     { networks: [backend] }    # not reachable from frontend
```
