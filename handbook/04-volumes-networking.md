# Volumes & Networking

> **Linter rules explained in this chapter:** CM-12, CM-15, CM-18 (and the networking side of CM-05). Each is tagged inline at the section that covers it.

## Volumes

Containers are **ephemeral**: anything written inside one disappears when it's removed. That's fine for the app itself, but disastrous for a database. **Volumes** are storage that lives *outside* the container's lifecycle, so data persists across restarts, rebuilds, and `docker compose down`. There are a few kinds, suited to different jobs:

| Type | Lives where | Use for |
|---|---|---|
| **Named volume** | Docker-managed area on host | Databases, persistent app data |
| **Bind mount** | A specific host folder you choose | Live source code in dev, config files |
| **Anonymous volume** | Docker-managed, no name | Shielding a container path from a bind mount |
| **tmpfs** | RAM (never touches disk) | Secrets, scratch files you *want* gone |

### Named volume — managed by Docker, persists across `down` (CM-12)

```yaml
volumes:
  pg_data:                          # Docker manages the path

services:
  db:
    volumes:
      - pg_data:/var/lib/postgresql/data
```

Data survives `docker compose down`. Removed with `docker compose down -v`.

**See it in action:**

```bash
docker compose up -d
docker compose exec db psql -U app -d appdb -c "CREATE TABLE t (id int); INSERT INTO t VALUES (1);"

docker compose down          # containers destroyed...
docker compose up -d         # ...brand-new db container starts
docker compose exec db psql -U app -d appdb -c "SELECT * FROM t;"
#  id
# ----
#   1          ← row is still there: the named volume outlived the container
```

Run the same sequence *without* the `pg_data` volume and the `SELECT` returns `relation "t" does not exist` — the data died with the container. Now run `docker compose down -v` and repeat: the `-v` wipes the volume too, so the table is gone. That's the difference between "restart" and "reset."

### Bind mount (host path ↔ container path)

```yaml
volumes:
  - ./src:/app/src                  # absolute or relative host path
  - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro   # read-only
```

The `:ro` suffix mounts the file read-only, so the container can read the config but can't modify it. Use it for anything the container should only *consume* (config files, certs, seed data): it enforces the intent, prevents a process from corrupting your host file, and means a compromised container can't tamper with the mounted config. Drop `:ro` only when the container genuinely needs to write back (like live source during development).

### Anonymous volume (shield a container dir from host overlay)

```yaml
volumes:
  - /app/node_modules               # no host path — prevents host dir from shadowing
```

### tmpfs (in-memory, not persisted)

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /run
```

---

## Networking

By default each container is isolated, but containers on the same Docker **network** can talk to each other, and Docker runs a small internal DNS so they can find each other by name. The questions networking answers are: *which containers can reach which others*, and *what's reachable from the host or outside world*.

### Compose's default network (a per-project user-defined bridge)

All services in a Compose file share one network that Compose creates automatically. They reach each other by **service name** as the hostname (e.g. the app dials `db:5432`). You only need the custom setups below when you want to *restrict* who can talk to whom.

> Worth knowing precisely: this auto-created network is a **user-defined bridge**, and that's exactly why name resolution works — Docker runs an embedded DNS server on user-defined networks. Docker's *other* bridge, the legacy `docker0` "default bridge" you land on with a plain `docker run` and no network flag, does **not** provide name-based DNS; there you'd have to use `--link` or raw IPs. So "containers find each other by name" is a property of user-defined networks (which Compose always gives you), not of bridge networking in general.

### Publishing ports vs. internal-only — keep databases off the host (CM-15)

There are two ways a service's port can be reachable, and the difference is a security boundary:

- **`ports: ["5432:5432"]`** *publishes* the container port to the **host's network interface**. Anyone who can reach the host — another machine on the LAN, or the whole internet if the host has a public IP and an open firewall — can now hit that port directly. There's no NAT allow-list or Compose-network boundary in front of it.
- **`expose: ["5432"]`** (or simply omitting both) keeps the port reachable **only from other containers on the same Compose network**. Services already find each other by service name there — the app dials `db:5432` — with no published port at all.

So publishing a database or cache port is almost always a mistake: your app reaches `db:5432` over the internal network regardless, and the only thing a `ports:` mapping adds is exposure. A `postgres` or `redis` container with `ports:` on a cloud VM is one firewall slip away from being an open datastore on the internet — and these images often ship with weak or well-known default credentials.

```yaml
db:
  image: postgres:16-alpine
  # ❌ ports: ["5432:5432"]   # reachable from the host network — and maybe the internet
  # ✅ no ports: at all — the app still reaches db:5432 on the Compose network
```

**If you need local GUI access** (psql, TablePlus, RedisInsight) while developing, don't put the mapping in the committed Compose file that also gets deployed. Add it in `docker-compose.override.yml` — which Compose loads automatically and which you keep gitignored — so the port is published on your laptop but never in production:

```yaml
# docker-compose.override.yml  (gitignored, local only)
services:
  db:
    ports: ["5432:5432"]
```

### Custom networks: isolating groups of services (CM-18)

```yaml
networks:
  frontend:
  backend:

services:
  nginx:
    networks: [frontend, backend]

  app:
    networks: [backend]

  db:
    networks: [backend]
```

`nginx` can reach `app` and `db`; `app` can reach `db`; external clients cannot reach `db` directly.

Picture it as two rooms, where a service can only talk to others sharing a room:

```
          frontend network          backend network
        ┌───────────────────┐   ┌───────────────────────┐
        │      nginx  ◄──────┼───┼──►  app  ◄──►  db      │
        └───────────────────┘   └───────────────────────┘
nginx ∈ {frontend, backend}   app ∈ {backend}   db ∈ {backend}
```

`nginx` straddles both, so it can talk to everyone. `db` lives only in `backend`, so `nginx` reaches it but it has no path to the outside. Concretely, from inside the `nginx` container `ping db` resolves and connects; if you put a service in `frontend` only, `ping db` from there fails with "name does not resolve" — the DNS entry simply doesn't exist on a network it isn't a member of.

### Cross-compose-file communication

Each `docker compose` project normally creates its *own* private network, so services in two separate compose files can't see each other by default — useful isolation, but a problem when (say) a shared database stack and an app stack live in different files. The fix is to have both join one network that exists independently of either project. One file *creates* it; the other declares `external: true`, meaning "don't create this — it already exists, just attach to it." Without `external: true` the second project would try to create a second network of the same name and the services still wouldn't share one.

Both compose files must declare the same network *and* attach their services to it:

```yaml
# compose A — creates and owns the network
networks:
  shared:
    name: myproject_shared
services:
  db:
    image: postgres:16-alpine
    networks: [shared]

# compose B — attaches to the network A already made
networks:
  shared:
    external: true          # "don't create it — it exists; just join it"
    name: myproject_shared
services:
  app:
    build: .
    networks: [shared]      # can now reach `db` by service name
```

Start `compose A` first (so the network exists), then `compose B`. The `networks: [shared]` line on each service is essential — declaring the network at the top level alone doesn't attach anything to it.

### Host networking, Linux only (CM-05)

```yaml
services:
  app:
    network_mode: host    # app binds directly on host ports, no NAT
```

Normally Docker puts each container behind a virtual network with its own IP, and `ports:` maps host ports to container ports through NAT. `network_mode: host` removes that layer — the container shares the host's network stack directly. Use it when the NAT hop costs measurable throughput/latency, or when the app opens many unpredictable ports (e.g., media streaming, WebRTC) that you can't enumerate in a `ports:` list. The tradeoff is real, which is why it's not the default: you lose network isolation (the container can bind any host port and reach anything the host can), service-name DNS no longer applies, and it only works on Linux — so reserve it for cases that genuinely need it.
