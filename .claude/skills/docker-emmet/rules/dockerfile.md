# Dockerfile rules

_Loaded on demand by the docker-emmet skill. 24 rules (DF-01–DF-24); each lists a trigger, a production rationale, and a concrete fix._

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
