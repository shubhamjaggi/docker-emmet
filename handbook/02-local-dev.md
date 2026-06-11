# Local Development Patterns

When you ship an image to production it's a sealed, immutable artifact. But while *developing*, you want the opposite: edit a file on your laptop and have the running container pick it up instantly, without rebuilding. The patterns below make that happen, mostly by mounting your live source code into the container and pointing it at a dev-mode command.

## Hot-reload with bind mount

A **bind mount** maps a folder on your host directly into the container, so the container sees your edits in real time. Pair it with the framework's watch/reload command (`npm run dev`, `flask run --debug`, `bootRun`) and saves on your machine trigger reloads inside the container.

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: builder       # dev stage with devDependencies
    volumes:
      - ./src:/app/src       # live source
      - /app/node_modules    # anonymous volume shadows host node_modules
    command: npm run dev
    ports:
      - "3000:3000"
```

Why the second volume? The `./src` bind mount makes your host folder "win" over the container's. If you also bind-mounted the project root, your host's (possibly empty or OS-specific) `node_modules` would shadow the ones installed *inside* the image and break the app. The anonymous volume `- /app/node_modules` reserves that path so the container keeps its own copy.

**Concretely**, imagine you `npm ci` inside the image (Linux binaries) but ran `npm install` on a Mac, and you bind-mount the whole project:

```
- ./:/app           # host project root mounted over the container's /app
```

The container now sees your Mac's `node_modules`, and on first request it crashes with something like:

```
Error: \\?\...\node_modules\bcrypt\bcrypt_lib.node is not a valid Win32/Mach-O binary
```

— because a native module compiled for your laptop's OS can't load inside Linux. Reserving `/app/node_modules` with the anonymous volume keeps the image's correctly-compiled copy in place while still live-mounting your `./src`.

## Override file for dev extras (don't commit secrets)

The goal is to keep **one** `docker-compose.yml` that's valid in every environment, and layer machine-specific or dev-only tweaks on top without editing it. Compose supports this directly: if a `docker-compose.override.yml` exists, `docker compose up` **auto-merges it over** the base file (override values win). So the base file stays production-shaped and committed, while the override holds the things that should *not* ship — exposed DB ports for your local GUI, `DEBUG` flags, mounted fixtures. Teams typically commit the base file and git-ignore the override (or commit a `.override.example`), which is why "don't commit secrets" is in the heading: the override is exactly where local credentials tend to land.

`docker-compose.override.yml` is auto-merged when you run `docker compose up`:

```yaml
services:
  app:
    environment:
      DEBUG: "true"
    volumes:
      - ./fixtures:/app/fixtures
  db:
    ports:
      - "5432:5432"    # expose db port locally for DBeaver/psql
```

## Shell into a running container

```bash
docker exec -it <container_name> bash
docker exec -it <container_name> sh          # alpine fallback (busybox sh; alpine has no bash)
```

Note on distroless: these images ship **no shell at all**, so neither `bash` nor `sh` will exec — you'll get `exec: "sh": executable file not found`. To debug one interactively, build with the `:debug` tag (e.g. `gcr.io/distroless/static:debug`), which bundles a busybox shell, or inspect it from the outside with `docker cp` / `docker inspect` instead.

## Debugging a crashed container

If a container exits immediately, you can't `exec` into it (there's nothing running). Instead, start a fresh one but **override the entrypoint** with a shell, so you land at a prompt *before* the broken app runs and can inspect the filesystem, env, and config by hand:

```bash
docker run --rm -it --entrypoint bash myapp:latest
```

`--rm` auto-removes the container on exit; `-it` gives you an interactive terminal.
