# Examples

Each fixture pairs an intentionally broken `bad/` config with a corrected `good/` twin. Every line in a `bad/` file is annotated with the rule ID it triggers; every `good/` file is written to produce **zero** findings, so it doubles as a copy-paste-ready starting point.

## Why some cases have multiple files

A few rules are **mutually exclusive within a single Dockerfile** — you can't demonstrate both at once without writing something incoherent:

- **DF-02** (runs as root) needs *no* `USER`; **DF-21** (missing `--chown` before a non-root `USER`) needs *a* `USER`.
- **DF-22** (no `HEALTHCHECK`) needs *no* healthcheck; **DF-16** (healthcheck calls a missing binary) needs *one*.

So instead of one contrived "kitchen-sink" file, a case is split into a few **themed, realistic** Dockerfiles — each a believable image someone would actually write. Running the linter on the case folder (e.g. `/docker-emmet examples/java/bad`) discovers every Dockerfile under it and reports them all in one pass.

`examples/java/` is the worked example of this pattern:

```
java/
  bad/
    image-and-secrets/Dockerfile     # wrong base + leaked secret, runs as root
    build-hygiene/Dockerfile         # a messy builder stage
    runtime-and-health/Dockerfile    # runtime-stage permission & signal bugs
  good/
    image-and-secrets/{Dockerfile, .dockerignore}
    build-hygiene/{Dockerfile, .dockerignore}
    runtime-and-health/{Dockerfile, .dockerignore}
```

## Coverage matrix

Every one of the 42 rules is triggered in at least one `bad/` fixture and resolved in its `good/` twin.

### Dockerfile rules

| Rule | Demonstrated in |
|---|---|
| DF-01 secret in ARG/ENV | `java/…/image-and-secrets`, `node/bad`, `python/bad` |
| DF-02 runs as root | `java/…/image-and-secrets`, `node/bad`, `python/bad` |
| DF-03 `sudo` in container | `java/…/build-hygiene`, `go/bad` |
| DF-04 shell-form CMD/ENTRYPOINT | `java/…/runtime-and-health`, `node/bad` |
| DF-05 download without checksum | `java/…/build-hygiene`, `go/bad` |
| DF-06 source copied before install | `java/…/image-and-secrets`, `node/bad`, `python/bad` |
| DF-07 package cache not cleaned | `java/…/build-hygiene`, `node/bad`, `go/bad` |
| DF-08 `apt-get update` split from install | `java/…/build-hygiene`, `go/bad` |
| DF-09 no `--no-install-recommends` | `java/…/build-hygiene`, `node/bad`, `go/bad` |
| DF-10 multiple package-manager RUNs | `java/…/build-hygiene`, `go/bad` |
| DF-11 no `.dockerignore` | `java/…/image-and-secrets`, `node/bad`, `python/bad`, `go/bad`, `insecure/bad` |
| DF-12 `ADD` for a local copy | `java/…/build-hygiene`, `go/bad` |
| DF-13 full base image in runtime | `node/bad`, `python/bad`, `go/bad` |
| DF-14 JDK in runtime stage | `java/…/image-and-secrets` |
| DF-15 unpinned base image | `java/…/image-and-secrets`, `node/bad`, `go/bad` |
| DF-16 healthcheck binary absent | `java/…/image-and-secrets` |
| DF-17 `npm install` not `npm ci` | `node/bad` |
| DF-18 `pip install` without `--no-cache-dir` | `python/bad` |
| DF-19 Node runtime ships devDependencies | `insecure/bad` |
| DF-20 Go build missing `CGO_ENABLED=0` | `go/bad` |
| DF-21 `COPY` without `--chown` before USER | `java/…/runtime-and-health`, `go/bad`, `insecure/bad` |
| DF-22 long-running service, no HEALTHCHECK | `java/…/runtime-and-health`, `python/bad`, `go/bad` |
| DF-23 no `WORKDIR` | `java/…/build-hygiene`, `go/bad` |
| DF-24 multiple CMD/ENTRYPOINT | `java/…/runtime-and-health`, `go/bad` |

### Compose rules

Compose rules are language-agnostic, so they live in two compose fixtures rather than being duplicated per language.

| Rule | Demonstrated in |
|---|---|
| CM-01 Docker socket mounted | `insecure/bad` |
| CM-02 `privileged: true` | `insecure/bad` |
| CM-03 dangerous capability added | `insecure/bad` |
| CM-04 secret hardcoded in `environment` | `node/bad` |
| CM-05 `pid: host` / `network_mode: host` | `insecure/bad` |
| CM-06 `seccomp:unconfined` | `insecure/bad` |
| CM-07 `depends_on` without `service_healthy` | `node/bad` |
| CM-08 one-shot service set to auto-restart | `node/bad` |
| CM-09 no healthcheck on database/cache | `insecure/bad` |
| CM-10 no restart policy | `node/bad` |
| CM-11 unpinned image tag | `node/bad` |
| CM-12 database without a named volume | `node/bad` |
| CM-13 no resource limits | `node/bad` |
| CM-14 no logging limits | `node/bad` |
| CM-15 database/cache port exposed | `node/bad` |
| CM-16 obsolete `version:` key | `insecure/bad` |
| CM-17 `links:` (deprecated) | `insecure/bad` |
| CM-18 no network segmentation | `insecure/bad` |

## Try it

```
/docker-emmet examples/java/bad        # reports findings across all three Java scenarios
/docker-emmet examples/java/good       # no issues found ✓
/docker-emmet examples/insecure/bad    # every CRITICAL / security finding
```
