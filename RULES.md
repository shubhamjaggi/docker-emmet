# Rule Catalog

Every rule the [`/docker-emmet`](.claude/skills/docker-emmet/SKILL.md) linter enforces, mapped to the [handbook](handbook/) chapter that explains it in depth. The linter tells you *what* and *where*; the chapter tells you *why*.

Severities: 🔴 **CRITICAL** (container escape / security emergency) · ❌ **ERROR** (bug, data loss, or vulnerability) · ⚠️ **WARN** (fix before production) · ℹ️ **INFO** (improvement).

## Dockerfile rules

| ID | Sev | Rule | Deep dive |
|---|---|---|---|
| DF-01 | ❌ | Secret in `ARG`/`ENV` | [05 · Config & Secrets](handbook/05-config-secrets.md) · [08 · build secrets](handbook/08-security.md) |
| DF-02 | ⚠️ | Container runs as root | [06 · Non-root user](handbook/06-production.md) |
| DF-03 | ⚠️ | `sudo` inside the container | [08 · Security](handbook/08-security.md) |
| DF-04 | ⚠️ | Shell-form `CMD`/`ENTRYPOINT` (PID 1 signal trap) | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-05 | ⚠️ | Binary downloaded without checksum | [08 · Security](handbook/08-security.md) |
| DF-06 | ⚠️ | Source copied before dependency install | [01 · Layer caching](handbook/01-dockerfile.md) |
| DF-07 | ⚠️ | Package cache not cleaned in same `RUN` | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-08 | ⚠️ | `apt-get update`/`install` in separate `RUN` | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-09 | ℹ️ | `apt-get install` without `--no-install-recommends` | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-10 | ℹ️ | Multiple separate package-manager `RUN`s | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-11 | ⚠️ | No `.dockerignore` | [01 · .dockerignore](handbook/01-dockerfile.md) |
| DF-12 | ℹ️ | `ADD` used for a local file copy | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-13 | ℹ️ | Full base image in runtime stage | [01 · Base Image Choices](handbook/01-dockerfile.md) |
| DF-14 | ⚠️ | JDK in runtime stage (JRE suffices) | [01 · Base Image Choices](handbook/01-dockerfile.md) |
| DF-15 | ⚠️ | Unpinned base image (`latest`/no tag) | [06 · Tag conventions](handbook/06-production.md) |
| DF-16 | ❌ | Healthcheck probe binary absent in base | [03 · Healthchecks](handbook/03-compose.md) |
| DF-17 | ⚠️ | `npm install` instead of `npm ci` | [01 · Anatomy](handbook/01-dockerfile.md) |
| DF-18 | ℹ️ | `pip install` without `--no-cache-dir` | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-19 | ❌ | Node runtime stage ships devDependencies | [01 · Multi-Stage Builds](handbook/01-dockerfile.md) |
| DF-20 | ⚠️ | Go build missing `CGO_ENABLED=0` for scratch/distroless | [01 · Go](handbook/01-dockerfile.md) |
| DF-21 | ℹ️ | `COPY` without `--chown` before non-root `USER` | [01 · Build Hygiene](handbook/01-dockerfile.md) |
| DF-22 | ℹ️ | Long-running service with no `HEALTHCHECK` | [03 · Healthchecks](handbook/03-compose.md) |
| DF-23 | ℹ️ | No `WORKDIR` | [01 · Anatomy](handbook/01-dockerfile.md) |
| DF-24 | ⚠️ | Multiple `CMD`/`ENTRYPOINT` (all but last ignored) | [01 · Build Hygiene](handbook/01-dockerfile.md) |

## Compose rules

| ID | Sev | Rule | Deep dive |
|---|---|---|---|
| CM-01 | 🔴 | Docker socket mounted | [08 · The Docker socket](handbook/08-security.md) |
| CM-02 | 🔴 | `privileged: true` | [08 · privileged](handbook/08-security.md) |
| CM-03 | ❌ | Dangerous Linux capability added | [08 · Capabilities](handbook/08-security.md) · [06 · Drop capabilities](handbook/06-production.md) |
| CM-04 | ❌ | Secret hardcoded in `environment` | [05 · Config & Secrets](handbook/05-config-secrets.md) |
| CM-05 | ⚠️ | `pid: host` / `network_mode: host` | [08 · Shared namespaces](handbook/08-security.md) · [04 · Host networking](handbook/04-volumes-networking.md) |
| CM-06 | ⚠️ | `seccomp:unconfined` | [08 · Seccomp](handbook/08-security.md) |
| CM-07 | ⚠️ | `depends_on` without `service_healthy` | [03 · depends_on conditions](handbook/03-compose.md) |
| CM-08 | ❌ | One-shot service set to auto-restart | [03 · One-shot migration container](handbook/03-compose.md) |
| CM-09 | ⚠️ | No healthcheck on database/cache | [03 · Healthchecks](handbook/03-compose.md) |
| CM-10 | ⚠️ | No restart policy on long-running service | [06 · Restart policy](handbook/06-production.md) |
| CM-11 | ⚠️ | Unpinned image tag | [06 · Tag conventions](handbook/06-production.md) |
| CM-12 | ⚠️ | Database without a named volume | [04 · Volumes](handbook/04-volumes-networking.md) |
| CM-13 | ℹ️ | No resource limits | [06 · Resource limits](handbook/06-production.md) |
| CM-14 | ℹ️ | No logging limits | [06 · Log rotation](handbook/06-production.md) |
| CM-15 | ℹ️ | Database/cache port exposed to host | [04 · Networking](handbook/04-volumes-networking.md) |
| CM-16 | ℹ️ | Obsolete `version:` key | [03 · Compose hygiene](handbook/03-compose.md) |
| CM-17 | ⚠️ | `links:` (deprecated) | [03 · Compose hygiene](handbook/03-compose.md) |
| CM-18 | ℹ️ | No network segmentation | [04 · Custom networks](handbook/04-volumes-networking.md) |

---

To propose a new rule, see [Contributing](README.md#contributing). Every rule must cite the handbook chapter that justifies it — if no chapter covers it, the rule adds the chapter content too, so the catalog and the handbook never drift apart.
