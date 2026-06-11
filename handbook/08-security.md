# Container Security & Isolation

A container is **not** a virtual machine. Every container on a host shares that host's single Linux kernel — isolation is enforced by kernel features (namespaces, cgroups, capabilities, seccomp, LSMs like AppArmor/SELinux), not by a hardware boundary. That has one blunt consequence: **anything that weakens those kernel features moves you from "isolated process" toward "process running on the host."** This chapter covers the misconfigurations that erase that boundary — the ones [`/docker-emmet`](../README.md) flags as `CRITICAL` and `ERROR` — and how to harden against them.

The positive, day-to-day hardening controls (non-root user, read-only filesystem, resource limits, restart policy) live in [06-production.md](06-production.md). This chapter is about the **dangerous anti-patterns** and the runtime security model underneath them.

---

## The Docker socket — the keys to the kingdom (CM-01)

```yaml
# ☠️ NEVER do this in an application stack
services:
  app:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

`/var/run/docker.sock` is the API endpoint of the Docker daemon, which runs as **root on the host**. It has no authentication: anything that can write to that socket can tell the daemon to do anything. A container with the socket mounted can:

- launch a *new* container with `--privileged` and the host's entire filesystem bind-mounted at `/host`, then `chroot` into it — full host takeover;
- read the environment (and thus the secrets) of every other running container;
- pull and run any image from anywhere.

This is why it's the single worst misconfiguration the linter knows about: it's a complete container escape, and **running the container as non-root does not help** — the socket bypasses all of that. The classic one-liner that demonstrates total compromise:

```bash
# from inside a container that has the socket mounted:
docker run -v /:/host --privileged alpine chroot /host sh   # now you ARE the host
```

**What to do instead.** Whatever made you reach for the socket has a safer answer:

- *Building images in CI* → [Kaniko](https://github.com/GoogleContainerTools/kaniko) or [BuildKit](https://github.com/moby/buildkit) build without a daemon; or use `docker:dind` with TLS.
- *Monitoring/management UIs (Portainer, cAdvisor)* → if you truly must, mount the socket **read-only** (`:ro`) into a single, minimal, well-audited container — and never in the same stack as internet-facing app code. Even read-only socket access is powerful; treat it as a privileged admin plane.

---

## `privileged: true` — turning isolation off (CM-02)

```yaml
services:
  app:
    privileged: true     # ☠️
```

One flag, and the container gets: **every** Linux capability, **no** seccomp filter, **no** AppArmor/SELinux confinement, and access to **all host devices** under `/dev`. At that point the container can mount host filesystems, load kernel modules, and reach host hardware — escaping is trivial. `privileged` exists for a tiny set of system-level tools (Docker-in-Docker, some device drivers). It should never appear on an application service.

**Instead of `privileged`, grant the one specific thing you need.** Almost always that's a single capability or device:

```yaml
# need to bind port 80 as non-root? that's one capability, not "privileged":
cap_drop: [ALL]
cap_add: [NET_BIND_SERVICE]

# need one hardware device? map just that device:
devices:
  - /dev/snd:/dev/snd
```

---

## Linux capabilities — drop all, add back what you prove you need (CM-03)

Linux chops root's power into ~40 distinct **capabilities** (mount filesystems, change file ownership, send raw packets, …). Docker gives every container a **default set** of about 14 — enough for normal programs, far short of full root. Hardening means dropping the lot and re-adding only what the app demonstrably uses:

```yaml
cap_drop: [ALL]
cap_add: [NET_BIND_SERVICE]    # only if you bind a port below 1024
```

The linter flags `cap_add` of capabilities **beyond** the default set, because those genuinely expand what an attacker can do. The ones to never add casually:

| Capability | Why it's dangerous |
|---|---|
| `ALL` | Adds every capability at once — equivalent to `privileged`. |
| `SYS_ADMIN` | "The new root." Mount filesystems, manipulate namespaces, dozens of privileged calls. |
| `NET_ADMIN` | Reconfigure interfaces, routes, iptables; ARP-spoof neighbouring containers. |
| `SYS_PTRACE` | Inspect/inject into any process, including ones in other containers. |
| `SYS_MODULE` | Load kernel modules — direct path to full kernel compromise. |
| `SYS_RAWIO` / `BPF` / `PERFMON` | Raw I/O port and kernel-tracing access — read kernel memory, escape. |

> **A subtlety the linter gets right:** adding a capability that's *already in the default set* (`CHOWN`, `SETUID`, `SETGID`, `FOWNER`, `NET_RAW`, `SYS_CHROOT`, …) is a redundant no-op, not an escalation — so it is **not** flagged. The real risk is the genuinely extra ones above. The right posture is always `cap_drop: [ALL]` first, which removes even the defaults, then add back the minimum.

---

## Seccomp — keep the syscall filter on (CM-06)

```yaml
services:
  app:
    security_opt:
      - seccomp:unconfined     # ⚠️ removes a whole layer of defense
```

Docker ships a **default seccomp profile** that blocks ~44 syscalls no normal app needs but which are staples of kernel-exploit chains (`ptrace`, `mount`, `pivot_root`, `kexec_load`, `reboot`, …). `seccomp:unconfined` switches that filter off entirely, re-exposing the kernel's full syscall surface.

People usually disable seccomp because one tool (a debugger, a profiler) needs one blocked syscall. The fix is to allow **that one syscall**, not all of them:

```yaml
# allow exactly what's needed, keep everything else filtered:
security_opt:
  - seccomp=./profiles/allow-ptrace.json
```

Generate a custom profile by starting from Docker's [default profile](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json) and adding only the syscalls your tool requires.

---

## Shared namespaces — `pid: host`, `network_mode: host` (CM-05)

Namespaces are *how* containers get their private view of processes, the network, mounts, and so on. Sharing the host's namespace removes that privacy:

- **`pid: host`** — the container sees and can signal **every process on the host**, including other containers and privileged daemons. `ps aux` inside the container now lists the whole machine; `kill` reaches host processes.
- **`network_mode: host`** — the container uses the host's network stack directly. No network namespace, no Compose service-name DNS, and any port it binds is immediately live on the host's external interfaces with **no NAT and no `ports:` allow-list** in between.

Both have legitimate niche uses (a host-level monitoring agent; a high-throughput or WebRTC app with many dynamic ports). Both should be deliberate, documented, and absent from ordinary application services. For the networking trade-offs in depth, see [04-volumes-networking.md](04-volumes-networking.md#host-networking-linux-only).

---

## Secrets at build time, done right (DF-01, DF-03, DF-05)

[05-config-secrets.md](05-config-secrets.md) covers the cardinal rule — **never bake a secret into an image** via `ARG` or `ENV`, because it stays recoverable from the layer history forever. But sometimes a build step *genuinely* needs a secret (a token to pull from a private package registry). The wrong fix is `ARG NPM_TOKEN`; the right one is a **BuildKit secret mount**, which exposes the value to a single `RUN` and never writes it to any layer:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```
```bash
docker build --secret id=npm_token,env=NPM_TOKEN .
```

Two related build-time hygiene rules:

- **No `sudo` in the image (DF-03).** During `RUN`, the build already executes as root, so `sudo` is pointless there; leaving it in the *runtime* image just hands an attacker an escalation tool. Do privileged work in `RUN` as root, then `USER app` before `CMD`.
- **Verify what you download (DF-05).** A `RUN curl … | sh` or an unverified tarball trusts the network completely — a CDN compromise or DNS hijack becomes code execution in your image. Pin and check a hash in the *same* layer:

```dockerfile
RUN curl -fsSL https://example.com/tool-1.2.3.tar.gz -o tool.tgz \
 && echo "abc123…  tool.tgz" | sha256sum -c \
 && tar -xzf tool.tgz && rm tool.tgz
```

---

## A hardened service, all together

Everything from this chapter and [06-production.md](06-production.md) combined into one reference service — a useful target to harden toward:

```yaml
services:
  app:
    build: .
    user: "1001:1001"            # non-root (see 06)
    read_only: true              # immutable root filesystem (see 06)
    tmpfs:
      - /tmp                      # the one writable path the app needs
    cap_drop: [ALL]              # drop every capability...
    cap_add: [NET_BIND_SERVICE]  # ...add back only the one used (omit if port ≥ 1024)
    security_opt:
      - no-new-privileges:true   # block setuid-based escalation
      # default seccomp profile stays ON (we don't unconfine it)
    deploy:
      resources:
        limits: { memory: 512M, cpus: "1.0" }
    restart: unless-stopped
    # no docker.sock, no privileged, no host namespaces, no exposed admin ports
```

---

## Scan images before you ship them

Hardening the *configuration* doesn't patch a vulnerable dependency baked into the base image. Make image scanning part of CI so known CVEs are caught before deploy:

```bash
docker scout cves myapp:latest        # Docker's built-in scanner
trivy image myapp:latest              # popular open-source alternative
```

And keep base images pinned and updated (see [06-production.md](06-production.md) on tagging) so a scan result is reproducible and a fix is a deliberate version bump — not a silent `latest` drift. For signing and verifying provenance, [`cosign`](https://github.com/sigstore/cosign) lets a deploy step prove an image came from your pipeline and wasn't swapped in the registry.
