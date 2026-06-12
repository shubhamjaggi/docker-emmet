# Examples

Each fixture pairs an intentionally broken `bad/` config with a corrected `good/` twin. Every line in a `bad/` file is annotated with the rule ID it triggers; every `good/` file is written to produce **zero** findings, so it doubles as a copy-paste-ready starting point.

## Why each language has multiple files

A few rules are **mutually exclusive within a single Dockerfile** — you can't demonstrate both at once without writing something incoherent:

- **DF-02** (runs as root) needs *no* `USER`; **DF-21** (missing `--chown` before a non-root `USER`) needs *a* `USER`.
- **DF-22** (no `HEALTHCHECK`) needs *no* healthcheck; **DF-16** (healthcheck calls a missing binary) needs *one*.

So instead of one contrived "kitchen-sink" file, each language is split into three **themed, realistic** Dockerfiles — each a believable image someone would actually write. Running the linter on a case folder (e.g. `/docker-emmet examples/go/bad`) discovers every Dockerfile under it and reports them all in one pass.

```
<language>/
  bad/
    image-and-secrets/Dockerfile      # wrong/full base, a leaked secret, runs as root
    build-hygiene/Dockerfile          # a messy builder stage
    runtime-and-health/Dockerfile     # runtime-stage permission, signal & healthcheck bugs
  good/
    image-and-secrets/{Dockerfile, .dockerignore}
    build-hygiene/{Dockerfile, .dockerignore}
    runtime-and-health/{Dockerfile, .dockerignore}
```

`node/` additionally carries `bad/docker-compose.yml` + `good/docker-compose.yml` for the everyday Compose rules, and `insecure/` is a standalone deliberately-dangerous stack for the container-escape Compose rules.

## Coverage matrix

Every one of the 42 rules is triggered in at least one `bad/` fixture and resolved in its `good/` twin.

### Dockerfile rules

The three themes are consistent across languages. The table shows which theme demonstrates each rule, and in which languages.

| Rule | Theme · languages |
|---|---|
| DF-01 secret in ARG/ENV | image-and-secrets · java, node, python, go |
| DF-02 runs as root | image-and-secrets · java, node, python, go |
| DF-03 `sudo` in container | build-hygiene · java, node, python, go |
| DF-04 shell-form CMD/ENTRYPOINT | image-and-secrets · node, python, go · runtime-and-health · java |
| DF-05 download without checksum | build-hygiene · java, node, python, go |
| DF-06 source copied before install | image-and-secrets · java, node, python, go |
| DF-07 package cache not cleaned | build-hygiene · java, node, python, go |
| DF-08 `apt-get update` split from install | build-hygiene · java, node, python, go |
| DF-09 no `--no-install-recommends` | build-hygiene · java, node, python, go |
| DF-10 multiple package-manager RUNs | build-hygiene · java, node, python, go |
| DF-11 no `.dockerignore` | image-and-secrets · java, node, python, go · also insecure |
| DF-12 `ADD` for a local copy | build-hygiene · java, node, python, go |
| DF-13 full base image in runtime | image-and-secrets · node, python, go |
| DF-14 JDK in runtime stage | image-and-secrets · java |
| DF-15 unpinned base image | image-and-secrets · java, node, python, go |
| DF-16 healthcheck binary absent | image-and-secrets · java · runtime-and-health · node, python, go |
| DF-17 `npm install` not `npm ci` | image-and-secrets · node |
| DF-18 `pip install` without `--no-cache-dir` | image-and-secrets · python |
| DF-19 Node runtime ships devDependencies | runtime-and-health · node · also insecure |
| DF-20 Go build missing `CGO_ENABLED=0` | build-hygiene · go |
| DF-21 `COPY` without `--chown` before USER | runtime-and-health · java, node, python, go · also go-build-hygiene, insecure |
| DF-22 long-running service, no HEALTHCHECK | image-and-secrets · node, python, go · runtime-and-health · java |
| DF-23 no `WORKDIR` | build-hygiene · java, node, python, go |
| DF-24 multiple CMD/ENTRYPOINT | runtime-and-health · java, node, python, go |

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
/docker-emmet examples/go/bad          # reports findings across all three Go scenarios
/docker-emmet examples/java/good       # no issues found ✓
/docker-emmet examples/insecure/bad    # every CRITICAL / security finding
```
