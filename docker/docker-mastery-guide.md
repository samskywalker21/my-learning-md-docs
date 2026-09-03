# Docker Mastery Guide — A Hands-On Reference

## About This Document

**Spec captured from the original request, so future updates match style without re-explaining:**

- **Framing:** Curiosity-driven — no specific upcoming task drives this. Structured as a guided, hands-on tutorial (runnable examples + exercises) that builds durable technical fluency, not a project-specific how-to. Theory/history appears only at the point it's needed to explain a mechanism (e.g., why `version:` in Compose is obsolete, why containers boot faster than VMs) — never as a standalone narrative section.
- **Confirmed scope (all four, in one document):** (1) Core engine — images/containers/Dockerfiles/networking/volumes, (2) Compose & multi-container apps, (3) Production/orchestration concerns — registries, CI/CD builds, hardening, resource limits, healthchecks, and how Docker relates to Swarm/Kubernetes, (4) Internals — namespaces, cgroups, storage drivers, BuildKit.
- **Depth tiers:** Every major Part is structured as four progressive, explicitly labeled tiers — **Beginner → Working Knowledge → Advanced → Mastery** — so you can stop at whichever level you need. A tier is collapsed where it would be trivial or redundant for that topic (this happens most for Compose, which gets Beginner → Working Knowledge → Advanced only, since "Mastery" there is really just Part 5's internals applied).
- **Assumed background:** Basic Linux CLI/networking is *not* assumed — Part 0 is a primer. No prior Docker knowledge assumed.
- **Per-section ingredients (used where they add value):** plain-language explanation; a "wrong vs. right" snippet pair for gotchas; a runnable "Try It" exercise (curiosity-driven default) or a "Real Scenario" where a production-style incident illustrates the concept better than a toy example; an ASCII diagram where the concept is spatial/structural.
- **Sourcing rule:** Official Docker docs cited first; Stack Overflow used only for real-world gotchas and flagged as such; official docs win whenever they disagree with a popular but outdated blog convention (called out explicitly wherever that happens, e.g. the Compose `version:` key).
- **Format:** Clickable TOC, "back to top" links after each Part, cheat-sheet tables at the end, fenced code blocks with language tags.
- **To update this doc later:** keep this section, keep the tiered structure, keep the wrong-vs-right and Try-It patterns, re-verify anything version-specific against current docs before changing it, and prefer adding a new Part/tier over reverting to freeform prose.

---

## Table of Contents

- [Part 0: CLI & Networking Primer](#part-0-cli--networking-primer)
- [Part 1: Images, Containers & the Engine](#part-1-images-containers--the-engine)
  - [Beginner](#part-1-beginner)
  - [Working Knowledge](#part-1-working-knowledge)
  - [Advanced](#part-1-advanced)
  - [Mastery](#part-1-mastery)
- [Part 2: Storage & Networking](#part-2-storage--networking)
  - [Beginner](#part-2-beginner)
  - [Working Knowledge](#part-2-working-knowledge)
  - [Advanced](#part-2-advanced)
  - [Mastery](#part-2-mastery)
- [Part 3: Docker Compose](#part-3-docker-compose)
  - [Beginner](#part-3-beginner)
  - [Working Knowledge](#part-3-working-knowledge)
  - [Advanced](#part-3-advanced)
- [Part 4: Production & Orchestration](#part-4-production--orchestration)
  - [Beginner](#part-4-beginner)
  - [Working Knowledge](#part-4-working-knowledge)
  - [Advanced](#part-4-advanced)
  - [Mastery](#part-4-mastery)
- [Part 5: Internals Deep Dive](#part-5-internals-deep-dive)
- [Cheat Sheets](#cheat-sheets)
- [Suggested Learning Order](#suggested-learning-order)
- [Quick Self-Check](#quick-self-check)

---

## Part 0: CLI & Networking Primer

A few concepts get referenced constantly below — five minutes here saves confusion later.

**Shell basics:**

- A "shell" (bash, zsh, PowerShell) reads the commands you type and runs them.
- `sudo` (Linux/macOS) runs a command with administrator privileges — Docker commands need this unless your user is in the `docker` group. On Windows/macOS with Docker Desktop, you generally don't need it.
- `--flag value` (double dash) is a full word (`--name web`); `-f value` (single dash) is a short letter (`-p 8080:80`). Both are just ways of parameterizing a command.

**Networking basics:**

- An **IP address** identifies a machine (or container) on a network; a **port** identifies a specific service on that machine (port 80 for HTTP, 5432 for Postgres, etc.).
- **Port publishing** (`-p host_port:container_port`) connects a port on your real machine to a port inside a container — containers are network-isolated from the host by default.
- **DNS** lets you use names instead of IP addresses. Docker gives containers on the same user-defined network automatic name-based DNS — central to [Part 2](#part-2-storage--networking).
- **`localhost` inside a container refers to that container**, not your host machine or any other container — the single most common source of "why can't my app reach the database" confusion, revisited in [Part 2](#part-2-storage--networking).

**Try It — confirm your install:**

```bash
docker version        # client + server (daemon) versions
docker run hello-world # pulls a tiny image and runs it — proves the whole chain works
```

If `docker run hello-world` prints its welcome message, the client, the daemon, and image pulling are all working end to end.

[↑ back to top](#table-of-contents)

---

## Part 1: Images, Containers & the Engine

<a id="part-1-beginner"></a>

### Beginner

**What it is.** An **image** is a read-only template: a filesystem snapshot plus metadata (default command, exposed ports, env vars). A **container** is a running (or stopped) *instance* of an image, with a thin writable layer on top for runtime changes. One image → many containers, the same way one class can produce many objects.

The `docker` command you type is a **client**; it talks to the **Docker daemon** (`dockerd`), a background service that does the actual work. This is why `docker ps` still shows containers after you close the terminal that started them — the daemon and its containers run independently of any one client session.

```
   docker CLI (client)  ──REST API over a socket──▶  dockerd (daemon)
                                                          │
                                          pulls images, creates containers,
                                          manages networks & volumes
```

**Try It:**

```bash
docker pull nginx:1.27          # pull an image, no container yet
docker images                    # list images you have locally
docker run -d -p 8080:80 --name web nginx:1.27
curl http://localhost:8080       # nginx's default page
docker ps                        # see "web" running
docker stop web && docker rm web # stop, then remove the container
docker images                    # the image is still there — images and containers are separate
```

**Real Scenario:** A team notices `docker run myapp` behaves differently from what's on a colleague's machine. Root cause: the colleague had been using `docker exec` to patch files *inside a running container*, then kept reusing that container instead of rebuilding the image. The image on disk never changed. Containers are disposable; images are the source of truth — if it's not in the Dockerfile, it doesn't really exist.

[↑ back to top](#table-of-contents)

<a id="part-1-working-knowledge"></a>

### Working Knowledge

**Writing a Dockerfile.** A Dockerfile is a recipe: a sequence of instructions that produce an image.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

- `FROM` — base image. Pin a specific tag (`node:22-bookworm-slim`), never rely on the implicit `latest`, or a moved tag silently changes your build.
- `WORKDIR` — sets (and creates) the working directory for subsequent instructions.
- `COPY` — copies files from the build context into the image. (`ADD` does this too, plus auto-extracts archives and can fetch URLs — [official guidance](https://docs.docker.com/reference/dockerfile/#add) is to prefer `COPY` unless you specifically need that extra behavior.)
- `RUN` — executes a command *at build time*, producing a new layer.
- `CMD` — the default command when a container *starts*. Prefer the exec form `["node", "server.js"]` over the shell form `node server.js` — the exec form doesn't wrap your process in a shell, so signals like `SIGTERM` reach it directly for clean shutdowns.

**Wrong vs. Right — dependency install order:**

```dockerfile
# WRONG — copies everything before installing deps, so ANY source
# change invalidates the npm install layer on the very next build
FROM node:22-bookworm-slim
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

```dockerfile
# RIGHT — copy only the dependency manifests first; npm ci only
# re-runs when dependencies actually change
FROM node:22-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

**Try It — watch the cache work:**

```bash
docker build -t myapp .          # first build: everything runs
touch README.md                  # change a file npm ci doesn't care about
docker build -t myapp .          # COPY . . reruns, but "npm ci" layer shows CACHED
```

**Build context & `.dockerignore`.** `docker build .` sends the entire `.` directory to the daemon as the **build context** before any instruction runs — even if the Dockerfile only `COPY`s a fraction of it. A huge context (an accidental `node_modules` or `.git`) slows every build, and anything in the context could leak into a layer. A `.dockerignore` (same syntax as `.gitignore`) fixes both:

```
.git
node_modules
*.md
.env*
Dockerfile*
docker-compose*
```

**Layer caching, visually.** Docker caches each layer and reuses it on the next build *if and only if* the instruction and everything before it are unchanged — this is why instruction order matters.

```
Dockerfile order:              Cache behavior on a source-only change:
FROM base        ─┐
WORKDIR /app       │ cached    (unchanged)
COPY package*.json │ cached    (unchanged)
RUN npm ci          ┘ cached    (unchanged) ← the expensive step is skipped
COPY . .           ─┐ invalidated (this is what changed)
CMD [...]           ┘ invalidated (everything after a cache miss reruns)
```

**Real Scenario:** A CI pipeline's Docker build step that used to take 40 seconds started taking 6 minutes on every commit. The Dockerfile copied the whole repo *before* installing dependencies, so even a one-line README fix invalidated the dependency-install layer and forced a full reinstall. Reordering to copy manifests first brought it back to seconds for anything that didn't touch dependencies.

Source: [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) · [Build cache invalidation](https://docs.docker.com/build/cache/invalidation/)

[↑ back to top](#table-of-contents)

<a id="part-1-advanced"></a>

### Advanced

**BuildKit is the default builder.** Since Docker Engine 23.0, `docker build` uses **BuildKit** by default (previously opt-in via `DOCKER_BUILDKIT=1`) — it builds independent stages in parallel, skips unused stages entirely, and supports build-time secret mounts. If you see a tutorial exporting `DOCKER_BUILDKIT=1` before every build, that's a sign it predates the switch; you don't need it on current Docker. ([BuildKit docs](https://docs.docker.com/build/buildkit/))

**Multi-stage builds.** A single-stage Dockerfile that compiles code ships the compiler, build caches, and dev dependencies in the final image. Multi-stage builds use more than one `FROM`; each starts a fresh stage, and you selectively `COPY --from=<stage>` only the finished artifacts forward.

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: build
FROM node:22-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

# Stage 2: runtime — only build output + prod deps come along
FROM node:22-bookworm-slim
ENV NODE_ENV=production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The final image never contains the TypeScript compiler, dev dependencies, or intermediate build files.

**Try It — see the size difference:**

```bash
docker build -t myapp:multistage .
docker history myapp:multistage    # inspect each layer and its size
docker images myapp:multistage      # compare against a single-stage version
```

**Build-time secrets, done right.** `ENV`/`ARG` values can end up visible in image history or metadata — never put real secrets there. BuildKit's `--mount=type=secret` mounts a secret only for the duration of one `RUN`, never persisting it into a layer:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

**Wrong vs. Right — leaking a secret at build time:**

```dockerfile
# WRONG — token ends up baked into a layer, recoverable via `docker history`
# or by anyone who can pull the image
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc && npm ci
```

```dockerfile
# RIGHT — secret is mounted only for this RUN, never written to a layer
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```

Source: [Build secrets](https://docs.docker.com/build/building/secrets/)

[↑ back to top](#table-of-contents)

<a id="part-1-mastery"></a>

### Mastery

**What an image actually is on disk.** An image is not one big file — it's a manifest (JSON) referencing a stack of content-addressed layer tarballs plus a config blob. Each layer is identified by a SHA256 digest of its *contents*, which is why two images sharing an ancestor (e.g., both `FROM node:22-bookworm-slim`) can share the actual bytes on disk instead of duplicating them — `docker pull` for the second image only needs to fetch the layers it doesn't already have.

```bash
docker inspect nginx:1.27 --format '{{json .RootFS.Layers}}'  # the layer digest stack
```

**BuildKit's real advantage: a DAG, not a script.** Legacy `docker build` executed a Dockerfile as a strictly linear sequence. BuildKit instead builds a dependency graph (DAG) of stages and only executes what a given target actually depends on — this is why an unused multi-stage build stage (say, a `test` stage nobody targets) never runs at all, and why independent stages (e.g., building a frontend and backend in parallel) execute concurrently rather than serially.

```bash
docker build --target build -t myapp:build-stage .   # stop at an intermediate stage
```

**Cache export/import across machines.** Layer cache is local to the machine that built it by default — a fresh CI runner starts cold every time. BuildKit can export/import cache explicitly, which is how CI systems get warm-cache speed on ephemeral runners:

```bash
docker buildx build \
  --cache-to type=registry,ref=myregistry/myapp:cache \
  --cache-from type=registry,ref=myregistry/myapp:cache \
  -t myapp .
```

**`buildx` and multi-architecture images.** `docker buildx` is BuildKit's CLI frontend, and it's what makes cross-platform builds possible — building an `arm64` image on an `amd64` laptop via QEMU emulation, or producing a single tag that resolves to the right architecture on any machine (a **manifest list**):

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
```

Source: [Multi-platform images](https://docs.docker.com/build/building/multi-platform/) · [BuildKit cache backends](https://docs.docker.com/build/cache/backends/)

[↑ back to top](#table-of-contents)

---

## Part 2: Storage & Networking

<a id="part-2-beginner"></a>

### Beginner

Containers are ephemeral by default — delete the container, lose its writable layer. Two mechanisms persist or share data:

- **Volumes** — storage Docker creates and manages (under `/var/lib/docker/volumes` on Linux), referenced by name. [Official guidance](https://docs.docker.com/engine/storage/volumes/) treats these as the preferred mechanism for data generated by containers.
- **Bind mounts** — a specific host path mounted directly into the container. You own the path; Docker doesn't manage it.

```bash
# Named volume — Docker manages the storage
docker run -d -v db_data:/var/lib/postgresql/data postgres:17

# Bind mount — you point at an exact host path (common in local dev,
# so live source code shows up inside the container)
docker run -d -v "$(pwd)":/app -w /app node:22-bookworm-slim npm run dev
```

**Try It:**

```bash
docker run -d --name db -v db_data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=pw postgres:17
docker rm -f db                          # remove the container
docker run -d --name db2 -v db_data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=pw postgres:17    # new container, same volume
docker exec db2 psql -U postgres -c '\l' # data from before is still there
docker volume ls                          # the volume outlived two containers
```

[↑ back to top](#table-of-contents)

<a id="part-2-working-knowledge"></a>

### Working Knowledge

**Networking between containers.** Every container attaches to Docker's built-in `bridge` network by default — but containers on that default network can only reach each other by IP (fragile; IPs change). The fix, and the officially recommended pattern, is a **user-defined bridge network**: containers on it get automatic DNS resolution by container name.

```bash
docker network create app-network

docker run -d --name db --network app-network postgres:17
docker run -d --name api --network app-network \
  -e DB_HOST=db myapp:latest
# inside "api", the hostname "db" resolves automatically — no IP needed
```

**Wrong vs. Right:**

```bash
# WRONG — both on the default bridge network; "api" has no
# reliable way to find "db" by name
docker run -d --name db postgres:17
docker run -d --name api -e DB_HOST=db myapp:latest   # DNS lookup for "db" fails
```

```bash
# RIGHT — shared user-defined network gives automatic DNS
docker network create app-network
docker run -d --name db --network app-network postgres:17
docker run -d --name api --network app-network -e DB_HOST=db myapp:latest
```

**Try It — reproduce and fix the failure yourself:**

```bash
docker run -d --name db postgres:17 -e POSTGRES_PASSWORD=pw
docker run --rm --network container:db alpine ping -c1 db   # fails: no DNS on default bridge
docker network create app-net
docker rm -f db
docker run -d --name db --network app-net -e POSTGRES_PASSWORD=pw postgres:17
docker run --rm --network app-net alpine ping -c1 db         # works
```

**Real Scenario:** "It works when I run it manually but the app can't reach the database" is one of the most common Docker support questions on Stack Overflow — almost always two containers started separately without a shared user-defined network, so the app is trying to resolve a hostname that has no DNS entry anywhere. ([Stack Overflow: Docker networking name resolution](https://stackoverflow.com/questions/tagged/docker-networking))

Source: [Bridge network driver](https://docs.docker.com/engine/network/drivers/bridge/)

[↑ back to top](#table-of-contents)

<a id="part-2-advanced"></a>

### Advanced

**The other network drivers.** `bridge` is the default for a single host, but it's one of several:

| Driver    | Use case                                                                 |
| --------- | ------------------------------------------------------------------------- |
| `bridge`  | Default; isolated network on one host, needed for most local setups       |
| `host`    | Container shares the host's network stack directly — no port mapping, no isolation, lowest latency |
| `none`    | No networking at all — total isolation for a batch job that never needs the network |
| `overlay` | Spans **multiple hosts** — used by Swarm/Kubernetes-style multi-node setups, not relevant to single-host work |

```bash
docker run --rm --network host nginx:1.27   # binds directly to host's port 80, no -p needed
```

**Volume drivers and mount options.** Beyond the local driver, volumes support `read_only`, and named volumes can be pre-populated from an image's own filesystem at that path the first time they're used — useful for shipping default config into a volume without a separate init step. `tmpfs` mounts are a third option: in-memory only, never written to disk, gone when the container stops — right for secrets or scratch space that should never persist.

```bash
docker run -d --tmpfs /app/cache:size=64m myapp:latest
```

**Wrong vs. Right — `localhost` inside a container:**

```bash
# WRONG — app.js tries to connect to "localhost:5432" for Postgres,
# assuming "localhost" means the same thing it does on your host
const client = new Client({ host: 'localhost', port: 5432 });
```

```bash
# RIGHT — inside a container, "localhost" is that container. Use the
# service/container name on a shared network instead
const client = new Client({ host: 'db', port: 5432 });
```

Source: [Network drivers overview](https://docs.docker.com/engine/network/drivers/) · [tmpfs mounts](https://docs.docker.com/engine/storage/tmpfs/)

[↑ back to top](#table-of-contents)

<a id="part-2-mastery"></a>

### Mastery

**What a bridge network actually is.** A Docker bridge network is a Linux virtual bridge device (visible via `ip link` on the host, typically `br-xxxxxxxx` for a user-defined network, `docker0` for the default one). Each container gets a `veth` (virtual ethernet) pair — one end inside the container's network namespace, the other attached to the bridge — and Docker runs an embedded DNS server (at `127.0.0.11` inside each container) that resolves container/service names to their bridge IP.

```
   Host
   ┌───────────────────────────────────────────────┐
   │   br-app (bridge)                              │
   │   ┌─────────┐        ┌─────────┐               │
   │   │ veth0   │        │ veth1   │               │
   │   └────┬────┘        └────┬────┘               │
   └────────┼──────────────────┼────────────────────┘
            │ (netns)          │ (netns)
      ┌─────▼─────┐      ┌─────▼─────┐
      │ container │      │ container │
      │    db     │      │    api    │
      │ eth0      │      │ eth0      │
      │ resolves  │      │ resolves  │
      │ "api" via │      │ "db" via  │
      │ 127.0.0.11│      │ 127.0.0.11│
      └───────────┘      └───────────┘
```

**Try It — see the bridge from the host (Linux):**

```bash
docker network create demo-net
docker run -d --name c1 --network demo-net alpine sleep 3600
ip link show type bridge          # a new br-xxxxxxxx device exists
docker exec c1 cat /etc/resolv.conf  # nameserver 127.0.0.11 — Docker's embedded DNS
```

**Port publishing under the hood.** `-p 8080:80` isn't magic routing — on Linux, the daemon (via `docker-proxy` or, more commonly today, `iptables`/`nftables` DNAT rules) rewrites incoming traffic on the host's port 8080 to the container's internal IP on port 80. This is why a firewall rule blocking the host's iptables chain can silently break `-p` mappings even though `docker ps` shows the port as published — the rule and the actual packet path are two different layers.

**Storage drivers.** The writable container layer and the way image layers stack aren't implemented as literal file copies — Linux uses a **union filesystem** driver, most commonly `overlay2` today, which presents multiple directories (lower read-only image layers + one upper writable layer) as a single merged view without copying files between layers until they're actually modified (copy-on-write). `docker info` shows which storage driver is active.

```bash
docker info --format '{{.Driver}}'   # typically "overlay2" on modern Linux
```

Source: [overlayfs storage driver](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/) · [Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

[↑ back to top](#table-of-contents)

---

## Part 3: Docker Compose

<a id="part-3-beginner"></a>

### Beginner

**Why Compose.** Manually running `docker network create`, then multiple `docker run` commands with matching flags, doesn't scale past a couple of containers and isn't reproducible. **Compose** describes an entire multi-container application — services, networks, volumes — in one YAML file, then brings it all up or down with a single command.

```yaml
services:
  api:
    build: .
    ports:
      - '3000:3000'
    environment:
      - DB_HOST=db
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:17
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      retries: 5

volumes:
  db_data:
```

**Try It:**

```bash
docker compose up -d          # build (if needed), create, and start everything, detached
docker compose ps              # see both services
docker compose logs -f api     # stream one service's logs
docker compose down            # stop and remove containers + the project's network
```

Every top-level entry under `services` gets its own container, and Compose automatically creates a shared user-defined network for the project — services can already reach each other by service name (`db`, `api`) with no manual network setup, unlike the raw `docker run` case in [Part 2](#part-2-storage--networking).

[↑ back to top](#table-of-contents)

<a id="part-3-working-knowledge"></a>

### Working Knowledge

**The `version:` key is obsolete.** Older tutorials show Compose files starting with `version: "3.8"`. **Current official guidance is to omit this entirely.** The old 2.x/3.x versioned formats were merged into a single "Compose Specification" that the modern CLI always parses with the latest schema regardless of `version:` — the field is now purely informational and triggers an obsolete-field warning if present. This is exactly the kind of thing an outdated blog post gets wrong. ([Version and name top-level elements](https://docs.docker.com/reference/compose-file/version-and-name/))

**`depends_on` with a healthcheck condition.** `depends_on: db: condition: service_healthy` waits for the dependency's `HEALTHCHECK` to pass, not just for the container to *start* — see [Part 4](#part-4-production--orchestration) for why "started" and "actually ready" are different things.

**Day-to-day commands:**

```bash
docker compose up -d          # start everything, detached
docker compose logs -f api    # stream logs for one service
docker compose exec api sh    # shell into a running service's container
docker compose down           # stop and remove containers + network
docker compose down -v        # ...and also remove named volumes (destructive!)
```

Note the command is `docker compose` (a space, built into the Docker CLI) rather than the older standalone `docker-compose` binary — the hyphenated form is the legacy v1 tool and shouldn't be your starting point for a new setup.

**Wrong vs. Right — waiting on "started" instead of "ready":**

```yaml
# WRONG — api starts as soon as the db CONTAINER starts, not once
# Postgres is actually accepting connections; the app crashes on boot
services:
  api:
    build: .
    depends_on:
      - db
  db:
    image: postgres:17
```

```yaml
# RIGHT — api waits for db's healthcheck to report healthy
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:17
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      retries: 5
```

[↑ back to top](#table-of-contents)

<a id="part-3-advanced"></a>

### Advanced

**Dev/prod parity via override files.** Your Compose setup for local development should exercise the *same* image and topology as production, differing only in configuration. A base Compose file plus an override file for local-only additions (extra debug ports, bind-mounted source for hot reload) avoids duplicating the whole stack:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

**Profiles — opt-in services.** A service tagged with `profiles:` only starts when that profile is explicitly requested — useful for optional tooling (a debug UI, a seed job) you don't want running by default:

```yaml
services:
  adminer:
    image: adminer
    profiles: ['debug']
```

```bash
docker compose up -d                     # adminer is skipped
docker compose --profile debug up -d     # adminer starts too
```

**Try It — build the same stack two ways:**

```bash
docker compose config    # renders the fully-merged, resolved config — great for debugging
                          # "why isn't my override taking effect" questions
```

Source: [Compose file reference](https://docs.docker.com/reference/compose-file/) · [Compose profiles](https://docs.docker.com/compose/how-tos/profiles/)

[↑ back to top](#table-of-contents)

---

## Part 4: Production & Orchestration

<a id="part-4-beginner"></a>

### Beginner

**Registries.** An **image registry** (Docker Hub, GitHub Container Registry, AWS ECR, etc.) stores and distributes images by name+tag. `docker pull`/`docker push` talk to whichever registry a tag's prefix points to (no prefix = Docker Hub).

```bash
docker tag myapp:latest ghcr.io/yourname/myapp:1.0.0
docker push ghcr.io/yourname/myapp:1.0.0
```

**Try It:** create a free account on any registry, tag a local image with `<registry>/<namespace>/<name>:<tag>`, log in (`docker login`), and push it — then `docker pull` it on a different machine (or after `docker rmi` locally) to see the round trip.

[↑ back to top](#table-of-contents)

<a id="part-4-working-knowledge"></a>

### Working Knowledge

**Image size and attack surface.** Every package in a final image is something that can carry a vulnerability. Current guidance favors minimal base image variants:

- `-slim` variants (Debian-based, no build tooling) — a reasonable default for most apps.
- `alpine` variants — smaller still, but based on musl libc rather than glibc, which occasionally breaks native dependencies that assume glibc.
- **Distroless** images — no shell, no package manager at all; smallest attack surface, but harder to debug (no `docker exec ... sh`).

Avoid `latest` and unpinned generic images (`ubuntu:latest`) in anything you'd call production — pin a specific tag, or a digest for full reproducibility.

**Running as non-root.** By default, a process in a container that doesn't specify otherwise runs as **root**. If an attacker breaks out of the application, running as root inside the container makes any container-escape vector far more dangerous.

```dockerfile
# WRONG — no USER set; process runs as root by default
FROM node:22-bookworm-slim
WORKDIR /app
COPY . .
CMD ["node", "server.js"]
```

```dockerfile
# RIGHT — drop to a non-root user before running the app
FROM node:22-bookworm-slim
WORKDIR /app
COPY . .
USER node
CMD ["node", "server.js"]
```

(Many official images, including `node`, already ship a non-root user you can switch to — check before creating one with `useradd`.)

**Health checks.** By default, Docker only knows whether a container's main *process* is alive, not whether the service inside it can actually serve traffic. A `HEALTHCHECK` closes that gap:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD ["node", "healthcheck.js"]
```

Use the exec form here too, especially for minimal/shell-less images, so the check doesn't depend on a shell being present.

**Real Scenario:** A service opens its port immediately but takes 15 seconds to load a large in-memory cache before it can answer requests correctly. Without a healthcheck, `depends_on` or a load balancer sees "port is open" and routes traffic immediately, causing a burst of errors on every deploy. A healthcheck that pings a real readiness endpoint — not just "is the process alive" — makes dependents wait for `healthy`, not just `started`.

**Config and secrets.** Bake as little environment-specific config into the image as possible; inject at runtime instead (`environment:`/`env_file:` in Compose, `-e` on `docker run`). This is what makes "build once, promote the same image everywhere" possible (below). For real secrets, avoid `ENV`/`ARG` in a Dockerfile — prefer runtime injection or BuildKit's secret mounts ([Part 1 Advanced](#part-1-advanced)).

**Logging.** Write to `stdout`/`stderr` and let `docker logs` (or `docker compose logs`) capture it — don't have the application manage its own log files inside the container. This keeps the container's filesystem stateless and lets you swap the logging backend without touching application code.

[↑ back to top](#table-of-contents)

<a id="part-4-advanced"></a>

### Advanced

**Resource limits.** Without limits, one runaway container can starve every other process on the host. `docker run` and Compose both support CPU/memory ceilings:

```bash
docker run -d --memory=512m --cpus=1.0 myapp:latest
```

```yaml
services:
  api:
    image: myapp:latest
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
```

**Wrong vs. Right — no limits on a memory leak:**

```yaml
# WRONG — a slow memory leak in "api" can consume all host memory,
# eventually taking down unrelated containers too (OOM killer picks victims)
services:
  api:
    image: myapp:latest
```

```yaml
# RIGHT — a leak in api is contained to its own limit; the OOM killer
# restarts just this container, not the whole host
services:
  api:
    image: myapp:latest
    deploy:
      resources:
        limits:
          memory: 512M
    restart: unless-stopped
```

**Reverse proxy pattern.** A common single-host production pattern: one reverse proxy container (Nginx, Caddy, Traefik) is the only thing with a port published to the outside world; every application container sits behind it on the internal network, reached by service name.

```
        Internet
            │
            ▼
   ┌─────────────────┐
   │  reverse proxy   │  (port 80/443 published)
   │  (nginx/caddy)   │
   └────────┬─────────┘
            │  internal network, by service name
   ┌────────┼─────────┐
   ▼                  ▼
┌────────┐       ┌────────┐
│  api   │       │  web   │   (no ports published directly)
└────────┘       └────────┘
```

Application containers themselves never need a port published to the host — only the proxy does — reducing actual attack surface.

**Build-once, promote-everywhere.** Rebuilding an image separately per environment (dev image, staging image, prod image) invites "works in staging, breaks in prod," because the *artifact* differs, not just its configuration. Build **one** image, tag it, and promote that same tag through staging into production; only environment configuration (env vars, secrets, replica count) changes between environments, never the image contents.

**CI/CD image builds.** A typical pipeline: build → tag with the commit SHA (and optionally a semantic version) → push to a registry → the deployment step pulls that exact tag, never `latest`. Tagging by commit SHA makes every running container traceable back to an exact source snapshot — a `latest` tag alone can't tell you that.

[↑ back to top](#table-of-contents)

<a id="part-4-mastery"></a>

### Mastery

**Where Docker ends and orchestration begins.** Everything above (`docker run`, Compose) targets **one host**. The moment you need a container rescheduled automatically onto a *different* healthy machine when one host dies, or need to run replicas of a service across many machines behind a single virtual IP, you've left Docker Engine's job and entered **orchestration** territory — Kubernetes (the dominant choice today) or Docker Swarm (built into the Engine, much lower adoption now). The core concepts you already know transfer directly:

| Single-host concept (this doc) | Multi-host equivalent (Kubernetes)              |
| ------------------------------- | ------------------------------------------------ |
| `docker run` / a Compose service | a **Pod** (one or more co-located containers)     |
| user-defined bridge network + DNS | a **Service** (stable DNS name, load-balanced across pod replicas) |
| named volume                     | a **PersistentVolume**                            |
| `HEALTHCHECK`                    | **liveness/readiness probes**                     |
| `restart: unless-stopped`        | a **Deployment**'s replica/self-healing controller |

You do not need Kubernetes to be productive with Docker — most single-service or small-team workloads never need it — but recognizing this mapping means nothing you learned in this document is wasted if a project later requires multi-host orchestration.

**Container runtime layers below `dockerd`.** `dockerd` itself doesn't create containers directly — it delegates to **containerd** (an independent, CNCF-graduated container runtime daemon), which in turn spawns **runc** (the low-level OCI-compliant process that actually calls `clone()`/`unshare()` to set up namespaces and cgroups, covered in depth in [Part 5](#part-5-internals-deep-dive)) via a per-container **shim** process. This layering is why containers survive a `dockerd` restart on modern Docker — `containerd` and the shims keep running independently of the higher-level daemon.

```
   docker CLI
       │
       ▼
   dockerd  ──(restart doesn't kill running containers)──┐
       │                                                   │
       ▼                                                   │
   containerd                                               │
       │                                                    │
       ▼                                                    │
   containerd-shim (one per container)  ◀────────────────────┘
       │
       ▼
   runc (creates namespaces/cgroups, then exits)
       │
       ▼
   your process (PID 1 inside the container)
```

Source: [containerd](https://containerd.io/) · [runc / OCI runtime spec](https://github.com/opencontainers/runtime-spec) · [Kubernetes concepts](https://kubernetes.io/docs/concepts/)

[↑ back to top](#table-of-contents)

---

## Part 5: Internals Deep Dive

Everything above treated "container" as a black box that isolates a process. This Part opens the box. It's pure Mastery-tier — skip it on a first pass and come back once the mechanisms above feel routine.

**Namespaces — the isolation primitive.** A Linux **namespace** wraps a global kernel resource so a process sees its own private instance of it. A container is just a process (or process tree) started inside a specific combination of namespaces:

| Namespace | What it isolates                                | Why it matters                                              |
| --------- | ------------------------------------------------ | -------------------------------------------------------------- |
| `pid`     | Process IDs                                       | Your process can be PID 1 inside the container, invisible to other containers |
| `net`     | Network interfaces, routing, ports                | Each container gets its own `eth0`, IP, and port space — the basis of everything in [Part 2](#part-2-storage--networking) |
| `mnt`     | Filesystem mount points                            | A container sees its own root filesystem, not the host's |
| `uts`     | Hostname                                           | `docker run --hostname` works because of this |
| `ipc`     | Inter-process communication (shared memory, semaphores) | Prevents one container's IPC from colliding with another's |
| `user`    | UID/GID mapping                                    | Lets a process be "root" (UID 0) inside the container while mapping to an unprivileged UID on the host — the mechanism behind rootless container runtimes |

**Try It — see a container's PID namespace from the host (Linux):**

```bash
docker run -d --name c1 alpine sleep 3600
docker inspect -f '{{.State.Pid}}' c1     # the PID as seen from the HOST
docker exec c1 ps aux                      # inside the container, that same process is PID 1
```

Two different PID numbers for the exact same process — that's the `pid` namespace in action.

**cgroups — the resource-limiting primitive.** Namespaces control what a process can *see*; **control groups (cgroups)** control what it can *use* — CPU shares, memory ceilings, block I/O bandwidth. The `--memory`/`--cpus` flags from [Part 4](#part-4-production--orchestration) are a friendly interface over writing directly to cgroup pseudo-files.

```bash
docker run -d --name limited --memory=256m alpine sleep 3600
# on a Linux host with cgroup v2:
cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.max 2>/dev/null | head -1
```

**Namespaces + cgroups + a union filesystem = a container.**

```
                     ┌───────────────────────────────────────┐
                     │            "the container"              │
                     │                                          │
   namespaces  ──────▶  what this process can SEE               │
   (pid/net/mnt/...)   (its own PIDs, network, filesystem)       │
                     │                                          │
   cgroups     ──────▶  what this process can USE                │
                     │   (CPU shares, memory ceiling, I/O)        │
                     │                                          │
   overlay2 (union fs)▶  WHERE its filesystem comes from          │
                     │   (read-only image layers + one writable   │
                     │    layer, merged via copy-on-write)         │
                     └───────────────────────────────────────┘
```

A container is not a distinct kernel-level object at all — it's an ordinary Linux process, just started with an unusual combination of these three mechanisms already in place. This is *the* fact that explains nearly every other behavior in this document: why containers boot in milliseconds (no kernel to boot — [Part 1](#part-1-images-containers--the-engine)), why `docker top` on the host can see container processes (they're real host processes), and why a kernel-level vulnerability can, in principle, be exploited to escape a container (there's no hypervisor boundary, only these three mechanisms).

**Try It — watch overlay2 from the host (Linux):**

```bash
docker run -d --name c2 alpine sleep 3600
CID=$(docker inspect -f '{{.Id}}' c2)
docker inspect -f '{{json .GraphDriver.Data}}' c2   # shows LowerDir / UpperDir / MergedDir paths
```

`LowerDir` is the stack of read-only image layers, `UpperDir` is this container's writable layer, and `MergedDir` is what the container actually sees as `/` — a single merged view assembled by the kernel's `overlay` filesystem driver, with copy-on-write meaning a file is only ever copied into `UpperDir` the first time the container modifies it.

Source: [Namespaces (man7.org)](https://man7.org/linux/man-pages/man7/namespaces.7.html) · [cgroups (man7.org)](https://man7.org/linux/man-pages/man7/cgroups.7.html) · [Docker overlay2 driver](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)

[↑ back to top](#table-of-contents)

---

## Cheat Sheets

### Core CLI

| Command                                      | What it does                                                          |
| --------------------------------------------- | ------------------------------------------------------------------------ |
| `docker build -t name:tag .`                 | Build an image from the Dockerfile in the current directory              |
| `docker run -d -p 8080:80 name:tag`          | Run a container, detached, mapping host port 8080 → container port 80    |
| `docker ps` / `docker ps -a`                 | List running / all containers                                           |
| `docker logs -f <container>`                 | Stream a container's logs                                                |
| `docker exec -it <container> sh`             | Open a shell inside a running container                                  |
| `docker stop` / `docker rm`                  | Stop / remove a container                                                |
| `docker image prune` / `docker system prune` | Reclaim disk space from unused images/objects                            |
| `docker network create <name>`               | Create a user-defined bridge network                                     |
| `docker inspect <object>`                    | Full JSON detail on any container/image/network/volume                   |
| `docker history <image>`                     | Show each layer in an image and its size                                 |

### Compose CLI

| Command                            | What it does                                        |
| ------------------------------------ | ------------------------------------------------------ |
| `docker compose up -d`             | Build (if needed) and start all services, detached      |
| `docker compose down`              | Stop and remove containers + project network             |
| `docker compose down -v`           | ...and also remove named volumes                          |
| `docker compose logs -f <service>` | Stream one service's logs                                  |
| `docker compose exec <service> sh` | Shell into a running service                                |
| `docker compose config`            | Print the fully-resolved, merged configuration               |
| `docker compose build --no-cache`  | Force a full rebuild, ignoring the layer cache                |

### Dockerfile Instructions

| Instruction   | Purpose                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------- |
| `FROM`        | Base image                                                                                 |
| `WORKDIR`     | Set working directory for subsequent instructions                                            |
| `COPY`        | Copy files from build context into the image                                                 |
| `RUN`         | Execute a command at build time (new layer)                                                   |
| `ENV`         | Set an environment variable (baked into the image — not for secrets)                          |
| `ARG`         | Build-time-only variable, not present in the final image's env (still visible in build history — not for secrets either) |
| `EXPOSE`      | Document which port the container listens on (informational only — doesn't publish it)         |
| `USER`        | Drop to a non-root user for subsequent instructions and at runtime                              |
| `HEALTHCHECK` | Define how Docker checks whether the container is actually healthy                             |
| `CMD`         | Default command when the container starts (exec form preferred)                                 |

[↑ back to top](#table-of-contents)

---

## Suggested Learning Order

1. **Part 0 + Part 1 Beginner/Working Knowledge** — get the mental model and write your first Dockerfile against a small app of your own.
2. **Part 2 Beginner/Working Knowledge** — persist data with a volume, connect two containers with a user-defined network via plain `docker run`.
3. **Part 3** — move to Compose once you have 2+ services that need to talk to each other.
4. **Part 1/4 Advanced** — multi-stage builds, non-root, healthchecks, secrets, resource limits — the hardening pass most tutorials skip.
5. **Part 4 Mastery** — understand where Docker's job ends and Kubernetes/Swarm's begins, so you know what you'd be signing up for if a project needs multi-host orchestration.
6. **Part 5** — once the mechanisms above feel routine, open the box: namespaces, cgroups, and the union filesystem that make all of it possible. This section pays off retroactively — re-read Parts 1–2 afterward and notice how much of the "why" now has a concrete answer.

## Quick Self-Check

- Why does editing your source code sometimes force a full dependency reinstall in a Docker build — and how do you prevent it?
- Two containers started with plain `docker run` can't reach each other by name. What's missing, and why does Compose not have this problem by default?
- What's actually wrong with using `latest` as a base image tag in production?
- Why doesn't `EXPOSE` alone make a port reachable from outside the container?
- Why is running a container process as root riskier than it sounds, even though the container is "isolated"?
- What's the practical difference between a container being "started" and being "healthy," and which one should a dependent service wait for?
- Why is removing the `version:` key from a Compose file now the recommended practice instead of a lint nitpick?
- What problem do multi-stage builds solve that a `.dockerignore` file doesn't?
- What are the three Linux kernel mechanisms that combine to make something a "container," and which one is responsible for a container starting in milliseconds instead of minutes?
- If `dockerd` restarts, why do already-running containers usually keep running? What process is actually responsible for them staying up?
- At what point does a problem stop being a Docker Engine problem and become a Kubernetes/Swarm problem?

[↑ back to top](#table-of-contents)
