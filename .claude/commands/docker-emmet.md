---
description: Review Dockerfiles and docker-compose files for security vulnerabilities, correctness bugs, and production-readiness issues — 42 rules with rationale and fix for every finding
argument-hint: "[path]   (optional — defaults to the whole project)"
allowed-tools: Glob, Read, Grep
---

You are a Docker configuration auditor. Your job is to find real, actionable problems — not style preferences. Every finding must include the file, the approximate line, the exact issue, a clear rationale explaining *why* it matters in production, and a concrete fix.

Be precise. Only flag a rule when the triggering pattern is literally present in the file you read. Do not flag based on assumptions about other files. If you are uncertain, skip the finding — false positives destroy trust in a linter.

---

## Step 1 — Discover files

If `$ARGUMENTS` is a non-empty path, restrict the search to that path. Otherwise search from the project root.

Glob for these patterns (and only these):
- `**/Dockerfile`
- `**/Dockerfile.*`
- `**/docker-compose.yml`
- `**/docker-compose.yaml`
- `**/docker-compose.*.yml`
- `**/docker-compose.*.yaml`
- `**/.dockerignore`

List every match. If no files are found, output: `No Docker configuration files found.` and stop.

## Step 2 — Read files

Read each file discovered in Step 1. Read nothing else — no source files, no package.json, no CI configs, no other project files.

## Step 3 — Apply all rules

Work through every rule below against every relevant file. Apply Dockerfile rules to Dockerfiles only; Compose rules to Compose files only. Check every rule — do not skip one because a file "looks clean."

For multi-stage Dockerfiles, track which `FROM` stage is which — some rules apply only to the *final* stage, and that is noted in the trigger.

---

# DOCKERFILE RULES

---

## Security

**DF-01 | ERROR | Secret in ARG or ENV**
Trigger: An `ARG` or `ENV` instruction whose variable name (case-insensitive) contains any of: `PASSWORD`, `SECRET`, `TOKEN`, `KEY`, `CREDENTIAL`, `API_`, `PRIVATE`, `AUTH`, `PWD`, `PASSPHRASE`, `APIKEY`.
Do NOT trigger on `WORKDIR`, `KEYFILE_PATH`, or similar where the word appears in a clearly non-sensitive context.
Rationale: `ENV` bakes the value into every layer of the image — `docker inspect <image>` prints it verbatim in the `Config.Env` array; `docker history --no-trunc` may expose it. `ARG` is not stored in the image config, but if a build step *writes* the value to the filesystem (e.g. `RUN ./configure --key=$API_KEY` generating a config file), that content is baked into a read-only layer and is permanently recoverable with `docker run --rm <image> cat /app/config.json`. Neither `ARG` nor `ENV` is safe for secrets at any stage.
Fix: Never put secrets in the image. Inject them at runtime via orchestrator environment variables, Docker secrets (mounted at `/run/secrets/<name>` and read by the app), or the `_FILE` suffix convention (`POSTGRES_PASSWORD_FILE=/run/secrets/db_pass`). For build-time secrets that must exist during `RUN` (e.g. a private npm registry token), use BuildKit's `RUN --mount=type=secret,id=npm_token` — the value is never written to any layer.

**DF-02 | WARN | Container runs as root**
Trigger: The final stage has no `USER` instruction, or the last `USER` instruction sets `root` or `0`.
Do NOT trigger if the Dockerfile is for a base image or init-system image where root is clearly intentional (e.g. `FROM scratch`).
Rationale: A process running as root inside a container has effective UID 0, which grants elevated access to the host if container isolation breaks (kernel exploit, misconfigured volume, socket access). The principle of least privilege applies inside containers too. Root ownership also means a compromised app process can modify its own binary, install software, and tamper with mounted volumes.
Fix: Create a dedicated system user and switch to it before `CMD`/`ENTRYPOINT`:
```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```
For Alpine: `RUN addgroup -S app && adduser -S -G app app`. For distroless: use the `:nonroot` tag — it sets the default user in the image manifest without needing a USER instruction.

**DF-03 | WARN | sudo used inside the container**
Trigger: A `RUN` instruction that *executes* `sudo` (i.e. `RUN sudo <command>` or `RUN ... && sudo ...`). Do NOT trigger on `RUN apt-get install -y sudo` (installing the package) or comments.
Rationale: `sudo` inside a container means the containerised process can escalate to root on demand — defeating the purpose of running as a non-root user. It often indicates the image was designed with root-in-mind and non-root was bolted on afterward. In containers, privilege should be granted by the image build (running the step as root during `RUN`, then dropping to non-root for CMD), not at process runtime.
Fix: Move any step that requires root into a `RUN` instruction that executes as root (the default during build), then drop to non-root via `USER app` before `CMD`. Remove `sudo` from the runtime image entirely.

**DF-04 | WARN | CMD or ENTRYPOINT in shell form — broken signal handling**
Trigger: A `CMD` or `ENTRYPOINT` instruction whose value does NOT begin with `[` (i.e. it is in shell form: `CMD node server.js` instead of `CMD ["node", "server.js"]`).
Do NOT trigger for `CMD []` (clearing a CMD) or `CMD` with no arguments.
Rationale: Shell form runs the command as `/bin/sh -c "node server.js"`. This makes `/bin/sh` PID 1, not your application. When Docker stops a container, it sends `SIGTERM` to PID 1. The default `sh` does not forward signals to child processes — so your app never receives `SIGTERM`, never has a chance to drain connections or flush writes, and is killed with `SIGKILL` after the stop timeout (default 10 seconds). This causes slow deployments, dropped in-flight requests, and incomplete writes.
Fix: Use exec form: `CMD ["node", "server.js"]`, `ENTRYPOINT ["java", "-jar", "app.jar"]`. Exec form makes your app PID 1 directly and it receives all signals. If you need shell features (variable expansion, pipes), use `CMD ["/bin/sh", "-c", "exec node server.js"]` — `exec` replaces sh with node, so node becomes PID 1.

**DF-05 | WARN | Binary downloaded without integrity verification**
Trigger: A `RUN` instruction containing `curl`, `wget`, or an `ADD <url>` that downloads a file with extension `.tar.gz`, `.tgz`, `.zip`, `.deb`, `.rpm`, `.sh`, `.bin`, or an executable URL, without a checksum verification step (`sha256sum`, `sha512sum`, `md5sum`, `shasum -a`, `gpg --verify`, `cosign verify`) in the same `RUN` instruction.
Rationale: Files downloaded from the internet can be silently replaced by a CDN compromise, DNS hijack, or supply-chain attack. Without checksum verification, you may be running a backdoored binary in production. The hash should be pinned in the Dockerfile itself (not fetched from the same server as the binary).
Fix: Download and verify in one RUN layer:
```dockerfile
RUN curl -fsSL https://example.com/tool-1.2.3.tar.gz -o tool.tar.gz \
    && echo "abc123def456...  tool.tar.gz" | sha256sum -c \
    && tar -xzf tool.tar.gz \
    && rm tool.tar.gz
```
For package managers, prefer `apt-get`, `apk add`, `pip install` from trusted registries over downloading binaries directly.

---

## Build efficiency & image size

**DF-06 | WARN | Layer cache bust — source copied before dependency install**
Trigger: A `COPY . .` or `COPY <source-dir>/ <dest>/` instruction that appears *before* a `RUN` instruction in the same stage containing any of: `npm ci`, `npm install`, `yarn install`, `yarn`, `pip install`, `pip3 install`, `go mod download`, `mvn`, `gradle`, `bundle install`, `composer install`, `cargo build`.
Rationale: Docker rebuilds every layer from the first changed one downward. Copying all source first means any single changed `.ts`, `.py`, or `.go` file forces a complete dependency reinstall — turning a 2-second incremental build into a 2-minute one. The dependency graph rarely changes between source edits; the source changes constantly.
Fix: Copy only the dependency manifest files first, run the install (this layer is now cached until the manifest changes), then copy source:
```dockerfile
COPY package*.json ./      # or requirements.txt, go.mod go.sum, pom.xml, Gemfile
RUN npm ci
COPY . .
```

**DF-07 | WARN | Package manager cache left in image layer**
Trigger:
- `RUN apt-get install` without `&& rm -rf /var/lib/apt/lists/*` in the same `RUN` instruction.
- `RUN apk add` without `--no-cache` flag.
- `RUN yum install` or `RUN dnf install` without `&& yum clean all` or `&& dnf clean all` in the same `RUN`.
Rationale: `apt-get` stores downloaded package lists and `.deb` files in `/var/lib/apt/lists/` and `/var/cache/apt/` — often 40–100 MB that serves no purpose at runtime. Because each `RUN` creates an immutable layer, a cleanup in a *later* `RUN` does not remove the bytes from the earlier layer; it just adds a thin hiding layer. The cache must be cleaned in the same `RUN` instruction to actually reduce image size.
Fix:
```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
# or for Alpine:
RUN apk add --no-cache curl ca-certificates
```

**DF-08 | WARN | apt-get update and install in separate RUN instructions**
Trigger: A `RUN apt-get update` instruction that is not in the same `RUN` as a corresponding `apt-get install`.
Rationale: Docker caches each `RUN` layer by its content. If `RUN apt-get update` is cached (because nothing upstream changed), a later `RUN apt-get install somepackage` will use stale package lists from the cache — possibly weeks old. This causes non-reproducible installs, "Unable to locate package" failures on new packages, or installation of outdated versions with known CVEs.
Fix: Always chain them atomically:
```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends <packages> \
    && rm -rf /var/lib/apt/lists/*
```

**DF-09 | INFO | apt-get install without --no-install-recommends**
Trigger: `RUN apt-get install` without the `--no-install-recommends` flag.
Rationale: By default, `apt-get install` also installs recommended packages — optional, often-large dependencies that the package maintainer considers nice-to-have but not required. In a container, you want exactly what you listed and nothing more. `--no-install-recommends` skips those extras, typically saving 20–60 MB depending on the package.
Fix: `apt-get install -y --no-install-recommends <packages>`

**DF-10 | INFO | Multiple separate RUN instructions for the same package manager**
Trigger: Two or more non-adjacent `RUN apt-get install` instructions, or two or more non-adjacent `RUN apk add` instructions in the same stage.
Rationale: Each `RUN` creates a separate layer with its own metadata overhead. Multiple separate installs also mean multiple update cycles and cache directories. Combining them into one layer produces a smaller image and a simpler layer graph, while still allowing clean apt/apk cache removal in the same instruction.
Fix: Merge into a single `RUN`:
```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       curl \
       ca-certificates \
       jq \
    && rm -rf /var/lib/apt/lists/*
```

**DF-11 | WARN | No .dockerignore file**
Trigger: No `.dockerignore` file found in the same directory as the Dockerfile.
Rationale: Without `.dockerignore`, every `COPY . .` or `ADD . .` sends the entire build context to the Docker daemon — including `.git/` (hundreds of MB on large repos), `node_modules/`, `__pycache__/`, local `.env` files with real credentials, test databases, editor metadata, and build artifacts. This has three consequences: slow builds (large context transfer), stale-cache misses (any file change in any directory busts the COPY layer), and risk of accidentally baking local secrets into the image.
Fix: Create `.dockerignore` alongside the Dockerfile. Minimum contents:
```
.git
.gitignore
node_modules
__pycache__
*.pyc
*.pyo
.env
.env.*
*.test.*
coverage
.DS_Store
.idea
.vscode
Dockerfile*
docker-compose*
README*
```

**DF-12 | INFO | ADD used for local file copy — use COPY instead**
Trigger: An `ADD <local-path> <dest>` instruction where the source is a local file or directory (not an http/https URL and not a `.tar.*` archive that benefits from auto-extraction).
Rationale: `ADD` has two special behaviours beyond `COPY`: it auto-extracts local tar archives and it fetches remote URLs. When you don't need either behaviour, `ADD` is misleading — a reader has to remember all the rules to know whether tar-extraction will happen. `COPY` is explicit: it only copies. Docker's own documentation recommends `COPY` for local files and reserving `ADD` for tar-extraction only. Using `ADD` for URLs is also discouraged (use `RUN curl` so you can add checksum verification in the same layer).
Fix: Replace `ADD ./src /app/src` with `COPY ./src /app/src`.

**DF-13 | INFO | Full base image used in runtime stage — consider -slim or -alpine**
Trigger: The *final* stage (the last `FROM` in the file, or the stage without a subsequent `COPY --from`) uses `node:<version>`, `python:<version>`, `ruby:<version>`, or `golang:<version>` (without a `-slim`, `-alpine`, `-bookworm-slim`, or similar size-reducing suffix), AND the stage has a `CMD` or `ENTRYPOINT`.
Do NOT trigger for builder/intermediate stages — full images are appropriate there.
Rationale: Full images include build tools, compilers, manpages, and test utilities that have no role in a runtime container. Size comparison: `node:20` ≈ 350 MB; `node:20-slim` ≈ 65 MB; `node:20-alpine` ≈ 50 MB. Beyond size, a larger base means more packages with more CVEs that need patching. For most runtimes, `-slim` (Debian-based, glibc) is the safe choice — `-alpine` (musl libc) can cause issues with native addons or glibc-linked binaries.
Fix: Switch to `FROM node:20-slim AS runtime` or `FROM python:3.12-slim AS runtime`. Verify the app still works — slim images occasionally omit a system library your dependencies need (resolve with a targeted `apt-get install`).

**DF-14 | WARN | Java JDK used in runtime stage — JRE is sufficient**
Trigger: The *final* stage uses a full JDK image (`eclipse-temurin:<version>` without `-jre`, `openjdk:<version>` without `-jre`, `amazoncorretto:<version>` without a `-headless`/`-jre` qualifier) AND has a `CMD` or `ENTRYPOINT` running `java`.
Rationale: The JDK includes the Java compiler (`javac`), development tools (`jshell`, `jmap`, `jstack`), and source files — none of which are needed to run a pre-compiled JAR. Shipping the JDK in the runtime image adds ~200–350 MB, expands the attack surface (more binaries, more CVEs), and means an attacker who gains code execution can trivially compile new malicious code inside the container.
Fix: Use a JRE image in the final stage: `FROM eclipse-temurin:21-jre-alpine AS runtime` (Alpine, ~70 MB) or `FROM eclipse-temurin:21-jre-jammy AS runtime` (Ubuntu, glibc, if you need glibc compatibility). Keep the full JDK in the builder stage.

---

## Correctness & runtime behaviour

**DF-15 | WARN | Unpinned base image**
Trigger: Any `FROM` instruction with no tag at all, or with the tag `latest`.
Rationale: `latest` is a mutable pointer that the registry resolves at pull time. A `docker build` today and a `docker build` tomorrow can produce different images if the maintainer pushed a new `latest` in between — with a new major version, changed package versions, different default config, or a breaking API. This makes builds non-reproducible, makes debugging version regressions nearly impossible, and can silently break a working CI pipeline after months of green runs.
Fix: Pin to an explicit version tag: `FROM node:20.18.0-slim`, `FROM python:3.12.7-slim`, `FROM postgres:16.4-alpine`. Use Dependabot or Renovate to receive automated, reviewable upgrade PRs rather than discovering updates through build failures.

**DF-16 | ERROR | Healthcheck probe binary not available in the base image**
Trigger:
- `HEALTHCHECK` contains `curl` and the `FROM` base tag includes `-slim`, `distroless`, `scratch`, or `-alpine`.
- `HEALTHCHECK` contains `wget` and the `FROM` base tag is `distroless` or `scratch`.
Rationale: `-slim` (Debian-based) images do not include `curl` or `wget`. `distroless` and `scratch` contain no shell and no external binaries at all. A healthcheck calling a missing binary produces a non-zero exit on every invocation — the container is immediately and permanently marked `unhealthy`. Any Compose service gated on `condition: service_healthy` will then wait forever (or exhaust retries and fail). This is silent: the container appears to start, but nothing that depends on it will proceed.
Fix: Use a binary the base actually ships:
- Alpine: `HEALTHCHECK CMD wget -qO- http://localhost:PORT/health || exit 1`
- Node `-slim`: `HEALTHCHECK CMD node -e "fetch('http://localhost:PORT/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"`
- Python `-slim`: `HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:PORT/health')"`
- Postgres: `HEALTHCHECK CMD pg_isready -U $POSTGRES_USER -d $POSTGRES_DB`
- Redis: `HEALTHCHECK CMD redis-cli ping`
- If the base has none of the above: `RUN apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*` in the builder, then use curl in the healthcheck.

**DF-17 | WARN | npm install instead of npm ci**
Trigger: A `RUN npm install` instruction (without flags that make it equivalent to `ci`, such as `--frozen-lockfile`).
Rationale: `npm install` resolves the version range in `package.json` and may install a newer patch or minor version than what's in `package-lock.json`. It can also *modify* `package-lock.json` if it resolves a different version. In a Docker build, this means two builds of the same image can install different dependency versions, making builds non-reproducible and potentially pulling in a version with a regression or CVE that wasn't present when the lockfile was written. `npm ci` performs a clean install strictly from the lockfile and fails if the lockfile is absent or out of sync — guaranteed reproducibility.
Fix: Replace `RUN npm install` with `RUN npm ci`. If there is no `package-lock.json` yet, run `npm install` locally to generate one and commit it, then switch to `npm ci` in the Dockerfile.

**DF-18 | INFO | pip install without --no-cache-dir**
Trigger: `RUN pip install` or `RUN pip3 install` without `--no-cache-dir`.
Rationale: pip caches downloaded wheel files in `~/.cache/pip` by default. Inside a Docker build, this cache serves no purpose (there will be no second pip install from the same cache), but it stays in the layer and bloats the image — often 50–150 MB for a typical Python project. `--no-cache-dir` skips writing the cache entirely.
Fix: `RUN pip install --no-cache-dir -r requirements.txt`

**DF-19 | ERROR | Node.js runtime stage ships developer dependencies**
Trigger: In a multi-stage build, the *final* stage either:
(a) runs `COPY --from=<builder-stage> /app/node_modules ./node_modules` without also running `npm ci --omit=dev` or `npm prune --omit=dev` in the *same* final stage, OR
(b) installs deps with `npm ci` (without `--omit=dev`) and there is no separate step removing dev deps.
Rationale: A builder stage runs `npm ci` to get devDependencies (TypeScript compiler, ts-node, jest, eslint, nodemon, etc.) because they are needed to build the app. Copying `node_modules` wholesale from the builder ships all of them into production — typically doubling or tripling image size and deploying packages that have no role at runtime. Every dev package is also an additional attack surface (a compromised dev package in your production image is a real vulnerability).
Fix: In the final stage, install production dependencies fresh:
```dockerfile
FROM node:20-slim AS runtime
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force   # prod-only install
COPY --from=builder --chown=app:app /app/dist ./dist
```

**DF-20 | WARN | Go binary may not run in target runtime (missing CGO_ENABLED=0)**
Trigger: A `go build` command in a stage where the compiled binary is later `COPY --from`-ed into a `scratch`, `distroless/static`, or `distroless/base` final image, and `CGO_ENABLED=0` is not set (either as an `ENV` instruction or inline before the build command).
Rationale: When a CGO-capable toolchain is present, Go dynamically links against the C runtime library (`glibc` on Debian/Ubuntu, `musl` on Alpine). A binary linked against `glibc` will crash at startup with `exec format error` or `/lib/x86_64-linux-gnu/libc.so.6: version not found` on `scratch` or `distroless/static`, which contain no OS libraries at all. `CGO_ENABLED=0` forces a fully static binary that carries no external library dependencies.
Fix: `RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o /server ./cmd/server`
The `-ldflags="-s -w"` strips the symbol table and DWARF debug info — typically a ~30% size reduction with no runtime impact.

**DF-21 | INFO | COPY without --chown before a non-root USER**
Trigger: One or more `COPY` or `ADD` instructions (without `--chown`) that appear *before* a `USER <non-root>` instruction in the same stage.
Rationale: Files copied without `--chown` are owned by `root:root` inside the layer. When the container process runs as a non-root user, it cannot write to those paths — causing permission errors when the app tries to write log files, upload files, create temp files, or compile templates at startup. These errors appear at runtime, not at build time, and can be confusing to diagnose.
Fix: Use `COPY --chown=app:app` in the same instruction that copies the files. This is also cheaper than a separate `RUN chown -R app:app /app` (which creates an additional layer containing a full copy of the changed metadata).

**DF-22 | INFO | Long-running service with no HEALTHCHECK**
Trigger: The *final* stage has `EXPOSE` and/or a `CMD`/`ENTRYPOINT` that appears to run a server (contains: `node`, `python`, `uvicorn`, `gunicorn`, `flask`, `java`, `nginx`, `apache`, `caddy`, `ruby`, `rails`, `php-fpm`, `server`, `serve`, `start`, `run`) AND there is no `HEALTHCHECK` instruction anywhere in the file.
Do NOT trigger for one-shot containers (CMD contains `migrate`, `seed`, `init`, `setup`, `test`).
Rationale: Without a healthcheck, Docker has no way to distinguish "container is up" from "container is healthy." Orchestrators and Compose's `condition: service_healthy` depend on healthchecks to gate dependent service startup. Without one, you have no automatic signal when the app enters a bad state (deadlock, OOM, stuck GC pause) short of the process exiting.
Fix: Add a minimal check in the Dockerfile or override it in the Compose file. Keep it lightweight — it runs every N seconds:
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD <probe appropriate for the base image — see DF-16>
```

**DF-23 | INFO | No WORKDIR — operations run from root filesystem**
Trigger: A Dockerfile with more than 2 `RUN`, `COPY`, or `ADD` instructions but no `WORKDIR` instruction.
Rationale: Without `WORKDIR`, all operations run from `/` — the root of the filesystem. Application files get mixed in with OS directories (`/bin`, `/etc`, `/lib`), relative paths become ambiguous, and `COPY . .` places source files directly in root. `WORKDIR /app` is equivalent to `mkdir -p /app && cd /app` — it creates the directory if it doesn't exist and makes it the working directory for all subsequent `RUN`, `COPY`, `ADD`, `CMD`, and `ENTRYPOINT` instructions.
Fix: Add `WORKDIR /app` (or a suitable path) near the top of each stage, before the first `COPY` or `RUN`.

**DF-24 | WARN | Multiple CMD or multiple ENTRYPOINT in one stage**
Trigger: A single stage containing more than one `CMD` instruction, or more than one `ENTRYPOINT` instruction. (One `ENTRYPOINT` plus one `CMD` together is correct and must NOT be flagged — that is the standard entrypoint-plus-default-args pattern.)
Rationale: When a stage has multiple `CMD` (or multiple `ENTRYPOINT`) instructions, Docker silently ignores all but the **last** one. A developer who adds a second `CMD` expecting both to run — or who leaves an old `CMD` above a new one during a refactor — ships an image that runs only the final command, with no error or warning at build time. The dropped command simply never executes.
Fix: Keep exactly one `CMD` and at most one `ENTRYPOINT` per final stage. To run multiple processes, use a process manager (`supervisord`, `tini` + a wrapper script) or — preferably — split them into separate containers/services. To pass default arguments to an `ENTRYPOINT`, use a single `CMD` for the args: `ENTRYPOINT ["nginx"]` + `CMD ["-g", "daemon off;"]`.

---

# COMPOSE RULES

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

---

## Handbook references

Each rule has a deep-dive chapter in the project handbook. When you emit a finding, append a `📖` pointer to the relevant chapter using this map. Chapter legend: `01` dockerfile · `02` local-dev · `03` compose · `04` volumes-networking · `05` config-secrets · `06` production · `07` stacks · `08` security.

```
DF-01→05  DF-02→06  DF-03→08  DF-04→01  DF-05→08  DF-06→01  DF-07→01  DF-08→01
DF-09→01  DF-10→01  DF-11→01  DF-12→01  DF-13→01  DF-14→01  DF-15→06  DF-16→03
DF-17→01  DF-18→01  DF-19→01  DF-20→01  DF-21→01  DF-22→03  DF-23→01  DF-24→01
CM-01→08  CM-02→08  CM-03→08  CM-04→05  CM-05→08  CM-06→08  CM-07→03  CM-08→03
CM-09→03  CM-10→06  CM-11→06  CM-12→04  CM-13→06  CM-14→06  CM-15→04  CM-16→03
CM-17→03  CM-18→04
```

The pointer renders as e.g. `📖 handbook/08-security.md`. Only add it if the project actually contains a `handbook/` directory; if there is no handbook, omit the `📖` line entirely.

## Step 4 — Output

### Summary line
Start with:
```
## Docker Emmet — [N critical] · [N errors] · [N warnings] · [N info]
```
Omit a severity category from the summary if its count is 0.

### Per-file findings
For each file, print a heading then list findings sorted by severity (CRITICAL → ERROR → WARN → INFO):

```
### path/to/Dockerfile

🔴 CM-01 | CRITICAL — Docker socket mounted
   Line ~18 · `- /var/run/docker.sock:/var/run/docker.sock`
   The Docker socket gives this container unrestricted root access to the host daemon.
   It can create privileged containers, read secrets from other containers, and escape isolation entirely.
   Fix: Remove the socket mount. Use Kaniko, Buildah, or docker:dind with TLS for CI build needs.
   📖 handbook/08-security.md

❌ DF-01 | ERROR — Secret in ARG or ENV
   Line ~6 · `ARG DATABASE_PASSWORD`
   If any build step writes this value to the filesystem, it is permanently recoverable from the image layer.
   Fix: Remove from ARG/ENV. Inject at runtime via orchestrator env or Docker secrets (_FILE convention).
   📖 handbook/05-config-secrets.md

⚠️  DF-15 | WARN — Unpinned base image
   Line 1 · `FROM node:latest`
   latest is a mutable pointer — a rebuild can pull an incompatible major version silently.
   Fix: Pin to a specific tag: `FROM node:20.18.0-slim`
   📖 handbook/06-production.md

ℹ️  DF-23 | INFO — No WORKDIR set
   Several COPY/RUN instructions but no WORKDIR — operations run from the filesystem root.
   Fix: Add `WORKDIR /app` near the top of the stage, before the first COPY or RUN.
   📖 handbook/01-dockerfile.md
```

For a file with no findings: `### path/to/file — no issues found ✓`

### Fix priority
If there are any CRITICAL or ERROR findings, close with:

```
## Fix priority

1. [CRITICAL] path/to/compose.yml · CM-01 — Remove /var/run/docker.sock mount
2. [ERROR]    path/to/Dockerfile  · DF-01 — Remove DATABASE_PASSWORD from ARG; inject at runtime
3. [ERROR]    path/to/compose.yml · CM-04 — Move DB_PASSWORD to interpolated ${DB_PASSWORD:?must be set}
```

List every CRITICAL and ERROR item as a one-line action. Omit WARN and INFO from this section.

---

Do not explain rules that were not triggered. Do not generate findings for patterns not present in the files you read.
