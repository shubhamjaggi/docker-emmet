# docker-emmet

[![links](https://github.com/shubhamjaggi/docker-emmet/actions/workflows/links.yml/badge.svg)](https://github.com/shubhamjaggi/docker-emmet/actions/workflows/links.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**A Docker config linter and a Docker handbook in one project.**

1. **The linter** — a Claude Code skill that audits Dockerfiles and `docker-compose` files for security vulnerabilities, correctness bugs, and production-readiness issues. 42 rules. A rationale and a concrete fix for every finding.
2. **The [handbook](handbook/)** — 8 plain-language chapters explaining *why* each rule exists, from layer caching to container-escape vectors. Every finding the linter reports ends with a clickable link to the chapter (on GitHub) that explains it.

Most linters tell you a rule was violated. This one tells you why it matters and hands you the chapter to go deep. It's a [Claude Code skill](https://code.claude.com/docs/en/skills): type `/docker-emmet` in any project — or just ask Claude to review your Docker setup and it loads automatically. A structured report in seconds, with no API keys, no network calls, and no dependencies beyond your existing Claude Code session.

```
## Docker Emmet — 2 critical · 3 errors · 4 warnings · 2 info

### Dockerfile

🔴 CM-01 | CRITICAL — Docker socket mounted
   Line ~18 · `- /var/run/docker.sock:/var/run/docker.sock`
   Gives this container unrestricted root access to the host Docker daemon.
   Fix: Remove the socket mount. Use Kaniko or docker:dind with TLS for CI builds.
   📖 https://github.com/shubhamjaggi/docker-emmet/blob/main/handbook/08-security.md

❌ DF-01 | ERROR — Secret in ARG or ENV
   Line ~6 · `ARG DATABASE_PASSWORD`
   If a build step writes this value to the filesystem, it is permanently recoverable
   from the image layer with a plain docker run ... cat.
   Fix: Remove from ARG/ENV. Inject at runtime via Docker secrets or _FILE convention.
   📖 https://github.com/shubhamjaggi/docker-emmet/blob/main/handbook/05-config-secrets.md

⚠️  DF-15 | WARN — Unpinned base image
   Line 1 · `FROM node:latest`
   latest is a mutable pointer — a rebuild can silently pull an incompatible version.
   Fix: FROM node:20.18.0-slim
   📖 https://github.com/shubhamjaggi/docker-emmet/blob/main/handbook/06-production.md

...

## Fix priority
1. [CRITICAL] docker-compose.yml · CM-01 — Remove /var/run/docker.sock mount
2. [ERROR]    Dockerfile          · DF-01 — Remove DATABASE_PASSWORD from ARG
3. [ERROR]    docker-compose.yml  · CM-04 — Move DB_PASSWORD to ${DB_PASSWORD:?must be set}
```

## What's in here

```
docker-emmet/
├─ .claude/skills/docker-emmet/        # the linter skill (creates /docker-emmet)
│   ├─ SKILL.md                        #   review procedure, output format, handbook map
│   └─ rules/                          #   42 rule definitions, loaded on demand
│       ├─ dockerfile.md               #     DF-01–24
│       └─ compose.md                  #     CM-01–18
├─ handbook/                           # the companion handbook (8 chapters)
│   ├─ 01-dockerfile.md                #   fundamentals, multi-stage, base images, build hygiene
│   ├─ 02-local-dev.md                 #   hot-reload, override files, debugging
│   ├─ 03-compose.md                   #   dependencies, healthchecks, compose hygiene
│   ├─ 04-volumes-networking.md        #   volumes, networks, segmentation
│   ├─ 05-config-secrets.md            #   .env, ARG vs ENV, Docker secrets
│   ├─ 06-production.md                #   non-root, limits, logging, registry, multi-platform
│   ├─ 07-stacks.md                    #   full ready-to-adapt stack snippets
│   └─ 08-security.md                  #   isolation model + the escape vectors the linter flags
├─ examples/                           # bad/ + good/ fixtures for Node, Python, Java
├─ .github/workflows/links.yml         # CI: checks every handbook/README link & anchor
├─ RULES.md                            # every rule → its handbook chapter
├─ LICENSE                             # MIT
└─ README.md
```

The linter and the handbook are kept in lockstep by [RULES.md](RULES.md): every rule cites the chapter that justifies it, and every chapter exists because a rule needs it.

## Installation

The skill is a directory (`SKILL.md` plus its `rules/`), so install the whole folder into your personal Claude Code skills directory. The easiest way is to clone and symlink — then `git pull` keeps it current:

```bash
git clone https://github.com/shubhamjaggi/docker-emmet
ln -s "$(pwd)/docker-emmet/.claude/skills/docker-emmet" ~/.claude/skills/docker-emmet
```

**Or download the files directly**

macOS / Linux:
```bash
mkdir -p ~/.claude/skills/docker-emmet/rules
base=https://raw.githubusercontent.com/shubhamjaggi/docker-emmet/main/.claude/skills/docker-emmet
curl -fsSL $base/SKILL.md            -o ~/.claude/skills/docker-emmet/SKILL.md
curl -fsSL $base/rules/dockerfile.md -o ~/.claude/skills/docker-emmet/rules/dockerfile.md
curl -fsSL $base/rules/compose.md    -o ~/.claude/skills/docker-emmet/rules/compose.md
```

Windows (PowerShell):
```powershell
$dir = "$env:USERPROFILE\.claude\skills\docker-emmet"
New-Item -ItemType Directory -Force "$dir\rules" | Out-Null
$base = "https://raw.githubusercontent.com/shubhamjaggi/docker-emmet/main/.claude/skills/docker-emmet"
Invoke-WebRequest "$base/SKILL.md"            -OutFile "$dir\SKILL.md"
Invoke-WebRequest "$base/rules/dockerfile.md" -OutFile "$dir\rules\dockerfile.md"
Invoke-WebRequest "$base/rules/compose.md"    -OutFile "$dir\rules\compose.md"
```

If the `~/.claude/skills/` directory didn't already exist, **restart Claude Code** (or start a new session) so it watches the new directory. Then run `/docker-emmet` — or just ask Claude to review your Docker files.

## Usage

```
/docker-emmet                    # audit all Docker files in the current project
/docker-emmet ./services/api     # audit a specific subdirectory (monorepos)
/docker-emmet ./Dockerfile       # audit a single file
```

Because it's a skill, Claude also loads it **automatically** when you ask it to review, audit, or harden a Dockerfile or compose file — no slash command required.

## Demo

No project of your own needed — every rule ships with a fixture. In Claude Code, point the linter at the intentionally-broken Node fixture:

> `/docker-emmet examples/node/bad`

The report opens with a severity tally, then lists every finding grouped by file — each with a rationale, a fix, and a handbook link (abbreviated here):

```text
### examples/node/bad/Dockerfile

❌ DF-01 | ERROR — Secret in ARG or ENV
   Line 8 · `ARG API_KEY`
   If a build step writes this to the filesystem, it's recoverable from the image layer forever.
   Fix: Remove from ARG/ENV; inject at runtime via Docker secrets (_FILE convention).
   📖 https://github.com/shubhamjaggi/docker-emmet/blob/main/handbook/05-config-secrets.md

⚠️  DF-04 | WARN — Shell-form CMD (PID 1 signal trap)
   Line 24 · `CMD npm start`
   sh becomes PID 1 and won't forward SIGTERM — shutdown drops in-flight requests after a 10s kill.
   Fix: Use exec form: CMD ["npm", "start"].
   📖 https://github.com/shubhamjaggi/docker-emmet/blob/main/handbook/01-dockerfile.md

   … more Dockerfile findings …

### examples/node/bad/docker-compose.yml

❌ CM-08 | ERROR — One-shot service set to auto-restart
   Line 29 · `restart: unless-stopped` (migrations)
   A migration exits 0; an auto-restart policy re-runs it in an endless loop.
   Fix: Set restart: "no" on one-shot jobs.
   📖 https://github.com/shubhamjaggi/docker-emmet/blob/main/handbook/03-compose.md

   … more Compose findings …

## Fix priority
1. [ERROR] docker-compose.yml · CM-08 — Set restart: "no" on the migrations service
2. [ERROR] docker-compose.yml · CM-04 — Replace hardcoded passwords with Docker secrets / ${VAR:?}
3. [ERROR] Dockerfile          · DF-01 — Remove API_KEY / DATABASE_PASSWORD from ARG/ENV
```

Then run the **good** twin to confirm a clean pass:

> `/docker-emmet examples/node/good` → `no issues found ✓`

## Rules

42 rules across two file types, organized by category. Every rule includes a precise trigger condition so findings are accurate, not guessed — and links to a handbook chapter for the full explanation. The complete rule-to-chapter map is in [RULES.md](RULES.md).

### Dockerfile — Security

| ID | Severity | Rule |
|---|---|---|
| DF-01 | ERROR | Secret in `ARG` or `ENV` — baked into image, recoverable from layers |
| DF-02 | WARN | Container runs as root — no `USER` instruction in final stage |
| DF-03 | WARN | `sudo` executed inside container — runtime privilege escalation |
| DF-04 | WARN | `CMD`/`ENTRYPOINT` in shell form — `sh` becomes PID 1, signals lost |
| DF-05 | WARN | Binary downloaded without checksum verification — supply chain risk |

### Dockerfile — Build efficiency & image size

| ID | Severity | Rule |
|---|---|---|
| DF-06 | WARN | Layer cache bust — source copied before dependency install |
| DF-07 | WARN | Package cache not cleaned in same `RUN` — bloats image silently |
| DF-08 | WARN | `apt-get update` and `install` in separate `RUN` — stale cache trap |
| DF-09 | INFO | `apt-get install` without `--no-install-recommends` |
| DF-10 | INFO | Multiple separate `RUN` for same package manager — merge them |
| DF-11 | WARN | No `.dockerignore` — full context sent, risk of leaking local secrets |
| DF-12 | INFO | `ADD` used for local file copy — use `COPY` instead |
| DF-13 | INFO | Full base image in runtime stage — `-slim` or `-alpine` available |
| DF-14 | WARN | Java JDK in runtime stage — JRE is 200 MB smaller and sufficient |

### Dockerfile — Correctness & runtime behaviour

| ID | Severity | Rule |
|---|---|---|
| DF-15 | WARN | Unpinned base image (`latest` or no tag) |
| DF-16 | ERROR | Healthcheck probe binary not available in base image |
| DF-17 | WARN | `npm install` instead of `npm ci` — non-reproducible installs |
| DF-18 | INFO | `pip install` without `--no-cache-dir` — wheel cache bloats image |
| DF-19 | ERROR | Node runtime stage ships devDependencies |
| DF-20 | WARN | Go build missing `CGO_ENABLED=0` for scratch/distroless target |
| DF-21 | INFO | `COPY` without `--chown` before non-root `USER` — runtime permission errors |
| DF-22 | INFO | Long-running service with no `HEALTHCHECK` |
| DF-23 | INFO | No `WORKDIR` — operations run from filesystem root |
| DF-24 | WARN | Multiple `CMD` or `ENTRYPOINT` — all but the last silently ignored |

### Compose — Security

| ID | Severity | Rule |
|---|---|---|
| CM-01 | CRITICAL | Docker socket mounted — container controls the host daemon |
| CM-02 | CRITICAL | `privileged: true` — all capabilities, no isolation |
| CM-03 | ERROR | Dangerous Linux capability added (`SYS_ADMIN`, `NET_ADMIN`, `ALL`, etc.) |
| CM-04 | ERROR | Secret hardcoded in `environment` block |
| CM-05 | WARN | `pid: host` or `network_mode: host` without justification |
| CM-06 | WARN | `seccomp:unconfined` — syscall filter disabled |

### Compose — Reliability & startup

| ID | Severity | Rule |
|---|---|---|
| CM-07 | WARN | `depends_on` without `service_healthy` when a healthcheck is defined |
| CM-08 | ERROR | One-shot service (migrate/seed) set to auto-restart — loops forever |
| CM-09 | WARN | No healthcheck on database or cache service |
| CM-10 | WARN | No restart policy on long-running service |
| CM-11 | WARN | Unpinned image tag |

### Compose — Data & resource safety

| ID | Severity | Rule |
|---|---|---|
| CM-12 | WARN | Database service without a named volume — data lost on `down` |
| CM-13 | INFO | No resource limits — container can exhaust host memory/CPU |
| CM-14 | INFO | No logging limits — json-file driver can fill host disk |
| CM-15 | INFO | Database or cache port exposed to host |
| CM-16 | INFO | Obsolete `version:` key — ignored by Compose V2 |
| CM-17 | WARN | `links:` used — deprecated, use networks |
| CM-18 | INFO | All services on implicit network — no segmentation |

## How it works

The skill is a lean `SKILL.md` (the review procedure and output format) plus two bundled rule files it loads on demand. Claude Code loads `SKILL.md` when you invoke `/docker-emmet` (or when it auto-activates on a Docker-related request). It then:

1. **Globs** for Docker-related files — nothing is read until files are confirmed to exist
2. **Reads** only the discovered files — no other project files are loaded into context
3. **Loads** only the rule file(s) it needs — `rules/dockerfile.md` (24 rules) and/or `rules/compose.md` (18 rules) — and applies every rule, each with a precise trigger condition to minimise false positives
4. **Outputs** findings grouped by file, ordered by severity (CRITICAL → ERROR → WARN → INFO), with rationale, a fix, and a `📖` link to the relevant handbook chapter

All rule knowledge lives in the skill's own files — no network calls, no external dependencies, no configuration required. It pulls in only the rule set it needs (Dockerfile and/or Compose), and never loads the handbook into context (that would bloat every review); it just emits a clickable link to the chapter (on GitHub) for you to open afterward.

## Examples

The `examples/` directory pairs an intentionally broken `bad/` config with a corrected `good/` config for Node.js, Python, and Java. Each line in the `bad/` files is annotated with the rule ID it triggers, and every `good/` config is written to produce **zero** findings — so it doubles as a copy-paste-ready starting point.

```
examples/
  node/   bad/{Dockerfile, docker-compose.yml}   good/{Dockerfile, docker-compose.yml, .dockerignore}
  python/ bad/Dockerfile                          good/{Dockerfile, .dockerignore}
  java/   bad/Dockerfile                          good/{Dockerfile, .dockerignore}
```

Verify the skill against them:

```
/docker-emmet examples/node/bad     # surfaces the full set of annotated findings
/docker-emmet examples/node/good    # should report: no issues found ✓
```

## Severity levels

| Severity | Meaning |
|---|---|
| CRITICAL | Container escape or immediate security emergency — fix before anything else |
| ERROR | Bug, data loss, exploitable vulnerability, or silent failure |
| WARN | Bad practice that should be resolved before production |
| INFO | Improvement opportunity — consider fixing, not urgent |

## Contributing

Bug reports, new rules, and improved rationale are welcome.

To propose a new rule, open an issue describing:
- The problem it catches and why it matters in production
- The precise trigger condition (to minimise false positives)
- A minimal example that triggers it (add to `examples/`)
- The recommended fix
- The handbook chapter that justifies it — **if none covers it, the PR adds that chapter content too.** The linter and handbook are kept in lockstep ([RULES.md](RULES.md)); a rule with no chapter, or a chapter with no rule, is the one thing we don't merge.

To test your changes, run `/docker-emmet examples/node/bad` and verify the annotated rules fire, then `/docker-emmet examples/node/good` and verify it reports no issues.

## License

MIT
