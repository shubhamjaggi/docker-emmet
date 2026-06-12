# Dockerfile Fundamentals, Multi-Stage Builds & Base Images

> **Linter rules explained in this chapter:** DF-04, DF-06–DF-14, DF-17–DF-21, DF-23, DF-24. Each is tagged inline at the section that covers it.

A **Dockerfile** is a plain-text recipe. Each line is an instruction that Docker runs top-to-bottom to assemble an image. The single most important thing to understand is **layer caching**:

> Every instruction creates a **layer** (a cached snapshot of the filesystem at that point). When you rebuild, Docker reuses a layer as long as that instruction *and everything above it* is unchanged. The moment one layer changes, every layer below it is rebuilt from scratch.

This is why you'll see dependency files (`package.json`, `requirements.txt`, `pom.xml`) copied and installed **before** the application source: dependencies change rarely, source changes constantly. Put the slow, stable steps high up so they stay cached, and the fast, churny steps low down.

**Example — installing dependencies before vs. after copying source (DF-06):**

```dockerfile
# ❌ Bad: source copied before install
# changes on every edit → cache busts here
COPY . .
# ...so deps reinstall every build (~40s)
RUN npm ci

# ✅ Good: deps installed before source
# changes only when deps change
COPY package*.json ./
# cached until package.json changes (~0s on rebuild)
RUN npm ci
# only this cheap layer rebuilds on a code edit
COPY . .
```

Now picture editing one line in `src/server.js` and rebuilding:

```
Layer                          Bad order        Good order
-----------------------------  ---------------  ---------------
COPY package*.json ./          —                CACHED
RUN npm ci                     REBUILD (~40s)   CACHED
COPY . .  (your edit landed)   REBUILD          REBUILD (~1s)
-----------------------------  ---------------  ---------------
Total rebuild time             ~40s             ~1s
```

A one-character change costs 40 seconds in the bad ordering and ~1 second in the good one — purely from instruction order.

## Dockerfile Anatomy

```dockerfile
# syntax=docker/dockerfile:1

# base image
FROM node:20-slim

# all subsequent paths are relative to this
WORKDIR /app

# copy dependency manifests first (cache layer)
COPY package*.json ./
# install deps in a separate layer
RUN npm ci --omit=dev

# copy source after deps (avoids reinstall on src change)
COPY . .

# baked-in env var
ENV NODE_ENV=production

# documentation only — does not publish the port
EXPOSE 3000

# fixed executable
ENTRYPOINT ["node"]
# default argument (overridable at runtime)
CMD ["server.js"]
```

Instruction-by-instruction:

- **`# syntax=...`** — opts into the modern BuildKit Dockerfile frontend (enables features like cache mounts). Optional but recommended as the first line.
- **`FROM`** — the starting point: a pre-built image you build on top of. Everything else is layered onto this.
- **`WORKDIR`** — sets the "current directory" inside the image. Created if it doesn't exist; all later `COPY`/`RUN` paths are relative to it.
- **`COPY <src> <dst>`** — copies files from your project (the *build context*) into the image. Copy dependency manifests *before* source so the install step stays cached (see the caching note above).
- **`RUN`** — executes a command *at build time* and bakes the result into a layer (e.g. installing packages). Contrast with `CMD`, which runs at *container start*. Here it's `npm ci --omit=dev`, and both flags are deliberate: `ci` (vs `install`) installs the *exact* versions pinned in `package-lock.json` and errors if the lockfile is out of sync — so the image is reproducible instead of silently drifting to newer patch versions on each build (using `install` here is **DF-17**). `--omit=dev` skips devDependencies (test runners, linters, type definitions) because they have no business in a production runtime — leaving them in bloats the image and adds vulnerabilities for code you'll never run (shipping them is **DF-19**, covered under Multi-Stage Builds below).
- **`ENV`** — sets an environment variable that persists into the running container.
- **`EXPOSE`** — pure documentation of which port the app listens on. It does **not** actually publish the port — you still need `-p 3000:3000` or Compose `ports:` to reach it from the host. Example: an image with `EXPOSE 3000` but run as `docker run myapp` is *unreachable* from your browser; run it as `docker run -p 3000:3000 myapp` and `localhost:3000` works. The `EXPOSE` line changed nothing — the `-p` flag did the real work.
- **`ENTRYPOINT` / `CMD`** — define what runs when the container starts (see table below).

**CMD vs ENTRYPOINT:**

| | `CMD` alone | `ENTRYPOINT` + `CMD` |
|---|---|---|
| Override at runtime | `docker run img node other.js` | `docker run img other.js` (arg only) |
| Use when | Script with a default command | Binary that always runs, args vary |

Rule of thumb: use `ENTRYPOINT` for the thing that *always* runs (your binary), and `CMD` for the *default arguments* you might want to swap. Together they read as "always run X, with these default args unless overridden."

## .dockerignore (DF-11)

```
node_modules/
.git/
.env
.env.*
*.log
dist/
__pycache__/
.venv/
.pytest_cache/
coverage/
.DS_Store

# Java / Kotlin
target/
build/
.gradle/
*.class
*.jar
*.war
.idea/
*.iml
```

A `.dockerignore` works like `.gitignore`, but for the **build context** — the snapshot of your folder that Docker reads before building. Always create one before building: without it, every `COPY . .` ships your entire repo (including `node_modules/`, `.git/`, and secrets in `.env`) to the Docker daemon. That makes builds slow, bloats images, and risks baking credentials into a layer.

---

## Multi-Stage Builds

**The problem:** the tools you need to *build* an app (compilers, full SDKs, dev dependencies) are large and you do **not** want them in the image you ship — they bloat size and widen the attack surface.

**The fix:** a multi-stage build uses several `FROM` blocks in one Dockerfile. Early "builder" stages do the heavy compiling; the final stage starts from a tiny base and `COPY --from=<stage>` pulls in *only the finished artifact*. Everything else (the whole toolchain) is discarded. You get a small, clean runtime image with the convenience of a single build.

**To picture the payoff**, here's what a Go service looks like single-stage vs. multi-stage:

```
Single-stage (FROM golang:1.22)        Multi-stage (final: FROM scratch)
- Go compiler, stdlib sources          - your compiled binary
- git, gcc, build cache                 (nothing else)
- your source code
- your compiled binary
≈ 1.2 GB, has a shell + 200 packages    ≈ 12 MB, no shell, ~0 CVEs
```

Same running program, ~100× smaller image, and an attacker who breaks in finds no shell, no package manager, and no compiler to work with.

### Pattern: build artifact → slim runtime (Node.js)

```dockerfile
# syntax=docker/dockerfile:1

# --- Stage 1: build ---
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# outputs to /app/dist
RUN npm run build

# --- Stage 2: runtime ---
FROM node:20-slim AS runtime
WORKDIR /app
COPY package*.json ./
# reinstall PROD deps only — devDependencies built the app but must not ship
RUN npm ci --omit=dev
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

> **(DF-19)** Note the runtime does a fresh `npm ci --omit=dev` rather than copying `node_modules` from the builder. The builder's `node_modules` contains devDependencies (the TypeScript compiler, test tooling) needed to *build*; copying it wholesale would drag all of that into the shipped image. Installing prod-only deps in the final stage keeps it lean.

### Python

```dockerfile
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
CMD ["python", "main.py"]
```

Why `--prefix=/install`? Python has no single "compiled artifact" to copy out — it has installed packages scattered across the interpreter's directories. `--prefix=/install` corrals every installed file under one folder, so the runtime stage can grab the whole dependency set with a single `COPY --from=builder /install /usr/local`. The payoff is that the *build* stage (which may pull in `gcc` and dev headers to compile C-extension wheels like `numpy` or `psycopg2`) is thrown away, and only the finished packages land in the slim runtime — no compiler left behind.

### Go (produces a near-zero-dependency binary)

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o server ./cmd/server

FROM gcr.io/distroless/static:nonroot AS runtime
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

Every flag here earns its place:

- **`CGO_ENABLED=0`** (DF-20) — disables cgo so Go links nothing against the system C library. The result is a *fully static* binary that depends on no `.so` files, which is the only reason it can run on `scratch` or `distroless/static` (those images have no libc to link against). Leave cgo on and the binary would crash on startup looking for `libc.so` that isn't there.
- **`GOOS=linux GOARCH=amd64`** — pins the target OS/architecture so a build on a Mac or Windows machine still produces a Linux/x86 binary the container can run. Without it you'd get a binary for *your* laptop, which won't execute inside the image.
- **`-ldflags="-s -w"`** — strips the symbol table (`-s`) and DWARF debug info (`-w`) from the binary, shrinking it by ~25–30%. You give up `gdb`-style debugging of the production binary, which you wouldn't do in a container anyway.
- **`distroless/static:nonroot` + `USER nonroot`** — pairs the smallest possible base with a built-in unprivileged user, so the container is both tiny *and* not running as root (see [06-production.md](06-production.md) for why that matters).

### Java — Maven (Spring Boot)

```dockerfile
# syntax=docker/dockerfile:1

# --- Stage 1: build ---
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
# cache deps layer
RUN ./mvnw dependency:go-offline -q
COPY src ./src
RUN ./mvnw package -DskipTests -q

# --- Stage 2: extract layers (Spring Boot 2.3+) ---
FROM eclipse-temurin:21-jdk-alpine AS layers
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# --- Stage 3: runtime ---
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=layers --chown=app:app /app/dependencies ./
COPY --from=layers --chown=app:app /app/spring-boot-loader ./
COPY --from=layers --chown=app:app /app/snapshot-dependencies ./
COPY --from=layers --chown=app:app /app/application ./
USER app
EXPOSE 8080
HEALTHCHECK --interval=15s --timeout=5s --start-period=30s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

Two build choices in stage 1 are worth calling out:

- **`COPY pom.xml` then `dependency:go-offline` *before* `COPY src`** — this is the layer-caching rule applied to Maven. `go-offline` downloads every dependency into the local `.m2` repo as its own cached layer, keyed only on `pom.xml`. As long as you don't touch `pom.xml`, that ~50–200 MB download is reused across builds and only your changed source recompiles. Copy `src` first and every code edit would re-download the internet.
- **`-DskipTests`** — the image build *packages* the app; it shouldn't be the thing that *runs the test suite*. Tests belong in an earlier CI step where a failure stops the pipeline cleanly. Running them here would make image builds slow and would conflate "did it compile?" with "did the tests pass?" — two failures you want reported separately. (Note: `-DskipTests` compiles tests but skips running them; `-Dmaven.test.skip=true` skips compiling them too.)

**Why layered JARs?** Spring Boot splits the JAR into `dependencies` (rarely changes), `snapshot-dependencies`, `spring-boot-loader`, and `application` (changes every build). Docker caches each as a separate layer — rebuilds only re-push the `application` layer instead of the full 50–200 MB fat JAR. Without this, a one-line code change forces Docker to re-store and re-transfer the *entire* fat JAR (all your dependencies included) on every build and deploy.

**Key JVM container flags:**

| Flag | Effect |
|---|---|
| `-XX:+UseContainerSupport` | Reads cgroup memory/CPU limits (on by default in JDK 11+, explicit is safe) |
| `-XX:MaxRAMPercentage=75.0` | Heap = 75% of container memory limit — avoids OOMKill |
| `-XX:InitialRAMPercentage=50.0` | Pre-allocates heap upfront; reduces GC pressure at startup |
| `-XX:+ExitOnOutOfMemoryError` | Crash fast instead of limping — lets orchestrator restart cleanly |

**Why this matters — the classic OOMKill:** an older JVM ignores the container limit and sizes its heap off the *host's* total RAM. On a 32 GB host with a 512 MB container limit, the JVM happily decides it may use ~8 GB of heap, grows past 512 MB, and Linux kills the container:

```
Host RAM: 32 GB │ Container limit: 512 MB
Without the flags →  JVM sees 32 GB, sets max heap ≈ 8 GB
                     heap grows past 512 MB → OOMKilled (exit 137), restart loop
With -XX:MaxRAMPercentage=75 →  JVM sees 512 MB, sets max heap ≈ 384 MB
                                stays under the limit → runs stably
```

That `exit 137` crash-loop on a "perfectly fine" app is almost always this. The percentage flags tie the heap to what the *container* was given, not the host.

### Java — Gradle

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle/ gradle/
# cache deps
RUN ./gradlew dependencies -q
COPY src ./src
RUN ./gradlew bootJar -x test -q

FROM eclipse-temurin:21-jdk-alpine AS layers
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=layers --chown=app:app /app/dependencies ./
COPY --from=layers --chown=app:app /app/spring-boot-loader ./
COPY --from=layers --chown=app:app /app/snapshot-dependencies ./
COPY --from=layers --chown=app:app /app/application ./
USER app
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

### Kotlin — Spring Boot (Gradle KTS)

Kotlin compiles to JVM bytecode, so the Dockerfile is almost identical to Java. The only difference is the build command — Gradle KTS handles Kotlin compilation transparently.

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle/ gradle/
RUN ./gradlew dependencies -q
COPY src ./src
# same as Java — Kotlin compiles to .jar
RUN ./gradlew bootJar -x test -q

FROM eclipse-temurin:21-jdk-alpine AS layers
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=layers --chown=app:app /app/dependencies ./
COPY --from=layers --chown=app:app /app/spring-boot-loader ./
COPY --from=layers --chown=app:app /app/snapshot-dependencies ./
COPY --from=layers --chown=app:app /app/application ./
USER app
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

### GraalVM native image (instant startup, tiny footprint)

Compiles Java/Kotlin to a native binary — no JVM at runtime. Startup drops from ~3 s to ~50 ms. Requires native-image-compatible code (Spring Boot 3+ has AOT support).

```dockerfile
FROM ghcr.io/graalvm/native-image:ol9-java21 AS builder
WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle/ gradle/
RUN ./gradlew dependencies -q
COPY src ./src
# Spring Boot: nativeCompile task
RUN ./gradlew nativeCompile -x test -q

# distroless/base also works
FROM debian:12-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder --chown=app:app /app/build/native/nativeCompile/myapp /app/myapp
USER app
EXPOSE 8080
ENTRYPOINT ["/app/myapp"]
```

> GraalVM native builds are slow (~5–15 min) and need at least 8 GB of RAM. Gate them to CI/CD; use the JRE image for fast local iteration.

### Targeting a specific stage

```bash
docker build --target builder -t myapp:dev .
docker build --target runtime -t myapp:prod .
```

---

## Base Image Choices (DF-13)

The base image (the `FROM` line) is a tradeoff between **size/security** and **convenience/debuggability**. Smaller images pull faster, cost less to store, and expose fewer vulnerabilities — but strip away tools you might want when something breaks. Key terms:

- **`alpine`** — a minimal Linux distro (~5 MB). Tiny, but uses `musl` libc instead of the usual `glibc`, which occasionally breaks pre-compiled binaries.
- **`slim`** — a stripped-down Debian: small but still has a normal shell and `glibc`. A safe default.
- **`distroless`** — Google images containing *only* your app and its runtime — no shell, no package manager. Very secure and small, but you can't `exec` a shell into them to poke around.
- **`scratch`** — literally empty. Only works for fully static binaries (typical Go/Rust) that need nothing from the OS.

| Image | Size | Use case |
|---|---|---|
| `ubuntu` / `debian` | ~80–120 MB | Maximum compatibility, debugging |
| `debian:slim` | ~30 MB | General purpose, has a shell |
| `alpine` | ~5 MB | Tiny; uses musl libc — watch for glibc-linked binaries |
| `distroless/base` | ~20 MB | No shell, no package manager — production hardened |
| `distroless/static` | ~2 MB | Go/Rust static binaries only |
| `scratch` | 0 MB | Fully static binaries (Go, Rust) |
| `eclipse-temurin:21-jdk-alpine` | ~200 MB | Java/Kotlin build stage (JDK needed) |
| `eclipse-temurin:21-jre-alpine` | ~80 MB | Java/Kotlin runtime (JRE only) |
| `eclipse-temurin:21-jre-jammy` | ~160 MB | JRE on Ubuntu — better glibc compatibility |
| `ghcr.io/graalvm/native-image:ol9-java21` | ~1.5 GB | GraalVM native compile stage only |

**JVM rule of thumb:** build stage uses `-jdk-`, runtime stage uses `-jre-`. Alpine variants are smaller; Jammy (Ubuntu) variants are safer for native libs (e.g., bcrypt, SSL).

**General rule of thumb:** develop on `debian:slim`, ship on `distroless` or `alpine`.

---

## Build Hygiene & Common Pitfalls

The rules above make images small and fast to rebuild. These make them *correct* — each is a quiet footgun that builds successfully but misbehaves at runtime or wastes space.

### Exec form vs shell form — the PID 1 signal trap (DF-04)

```dockerfile
# ❌ shell form — sh is PID 1, signals not forwarded
CMD node server.js
# ✅ exec form — node is PID 1 and receives signals directly
CMD ["node", "server.js"]
```

Shell form runs your command as `/bin/sh -c "node server.js"`, which makes **`sh` the PID 1** of the container, with `node` as its child. When Docker stops the container it sends `SIGTERM` to PID 1 — and the default `sh` does **not** forward signals to its children. So `node` never hears the shutdown, never drains connections or flushes writes, and is `SIGKILL`ed after the stop timeout (10s by default). The symptom is deploys that always take ten seconds and drop in-flight requests.

Exec form makes your process PID 1 directly, so it receives signals. If you genuinely need shell features (a pipe, variable expansion), use `exec` so your app still replaces the shell:

```dockerfile
CMD ["/bin/sh", "-c", "exec node server.js"]
```

> Related: a container's PID 1 also has to **reap zombie processes**. If your app spawns children and doesn't reap them, add a tiny init: `ENTRYPOINT ["/sbin/tini", "--"]` or run with `docker run --init` / Compose `init: true`.

### One CMD, one ENTRYPOINT (DF-24)

If a stage has **two `CMD`s** (or two `ENTRYPOINT`s), Docker silently uses only the **last** one — no error, no warning. The usual cause is a refactor that left an old `CMD` above the new one; the dropped command simply never runs. Keep exactly one of each. (One `ENTRYPOINT` *plus* one `CMD` is the correct pairing — see the CMD-vs-ENTRYPOINT table above.)

### apt/apk in a single, clean layer (DF-07, DF-08, DF-09, DF-10)

```dockerfile
# ❌ three separate problems
# cached independently → goes stale
RUN apt-get update
# may install against weeks-old package lists
RUN apt-get install -y curl
# second layer, second cache
RUN apt-get install -y jq

# ✅ one atomic, recommends-free, cleaned-up layer
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl jq \
 && rm -rf /var/lib/apt/lists/*
```

Four things are happening here, each its own rule:

- **`update` and `install` in the *same* `RUN` (DF-08).** Layers are cached by content. If `RUN apt-get update` is its own layer and gets cached, a later `RUN apt-get install foo` reuses *stale* package indexes — causing "Unable to locate package" or installs of outdated, CVE-bearing versions. Chaining them means the indexes are always fresh for the install.
- **`--no-install-recommends` (DF-09).** Skips "recommended" extras the maintainer considers optional — typically 20–60 MB you never use.
- **Clean the cache *in the same layer* (DF-07).** `apt` leaves package lists in `/var/lib/apt/lists/`. Because each layer is immutable, deleting them in a *later* `RUN` doesn't shrink the image — the bytes still live in the earlier layer. Remove them in the same `RUN` that created them. (Alpine: `apk add --no-cache …` does this in one step.)
- **One install layer, not many (DF-10).** Merge multiple installs so there's a single update/clean cycle and fewer layers.

### `pip install --no-cache-dir` (DF-18)

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

pip caches downloaded wheels under `~/.cache/pip` — useful on your laptop, useless in an image (there's no second install to benefit), and often 50–150 MB of dead weight. `--no-cache-dir` skips writing it.

### `COPY`, not `ADD`, for local files (DF-12)

`ADD` has two magic behaviours `COPY` doesn't: it auto-extracts local tar archives and it can fetch URLs. That magic makes intent ambiguous — a reader has to recall the rules to know whether a tarball will be unpacked. Use `COPY` for plain local files (it *only* copies), and reserve `ADD` for the one case it's actually meant for: extracting a local `.tar.*`. For URLs, prefer `RUN curl …` so you can verify a checksum in the same layer (see [08-security.md](08-security.md#secrets-at-build-time-done-right-df-01-df-03-df-05)).

### Own files as the runtime user (DF-21)

When you copy files in and then drop to a non-root `USER`, files arrive owned by `root` — so the app can't write to them at runtime (logs, uploads, caches fail with "Permission denied"). Set ownership in the copy itself, which is also cheaper than a separate `RUN chown -R` (that adds a whole extra layer):

```dockerfile
COPY --chown=app:app . .
USER app
```

### Set a WORKDIR (DF-23)

Without a `WORKDIR`, every `RUN`, `COPY`, and `ADD` operates from `/` — the filesystem root. `COPY . .` then dumps your source straight into `/`, mixed in with `/bin`, `/etc`, and `/lib`; relative paths in your commands become ambiguous; and you're one stray `rm -rf ./build` away from deleting a system directory. `WORKDIR /app` is just `mkdir -p /app && cd /app`, applied to every later instruction in the stage — set it once near the top, before the first `COPY` or `RUN`.

```dockerfile
# creates /app and makes it the cwd for everything below
WORKDIR /app
# lands in /app, not /
COPY . .
```

### Ship a JRE, not a JDK, at runtime (DF-14)

The JDK is the *build* toolchain — `javac`, `jshell`, debug agents, source files. To *run* a compiled `.jar` you only need a JRE. Leaving a full JDK in the final stage adds ~150–250 MB and hands anyone who gains code execution a compiler and a box of tools inside your container. Build with a `-jdk-` image, ship on a `-jre-` one (the multi-stage Java examples above do exactly this):

```dockerfile
# compile here
FROM eclipse-temurin:21-jdk-alpine AS builder
# ...build the jar...
# ship this — JRE only
FROM eclipse-temurin:21-jre-alpine AS runtime
```
