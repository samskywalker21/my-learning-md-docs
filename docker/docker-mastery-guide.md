# Docker: A Working Reference (Single-Host + Compose)

## About This Document

**Spec captured from the original request, so future updates match style without re-explaining:**

- **Framing:** Goal-driven — learning Docker toward developing and deploying production apps in containers.
- **Confirmed scope:** Single-host Docker + Compose only (no Kubernetes/Swarm/orchestration).
- **Examples:** Generic / stack-agnostic (not tied to a specific framework).
- **Assumed background:** Basic Linux CLI/networking is _not_ assumed — a short primer is included.
- **Depth philosophy:** A well-chosen set of concepts covered properly, not exhaustive coverage. Not every section needs every ingredient below.
- **Per-section ingredients (used where they add value):** plain-language explanation; a "wrong vs. right" snippet pair for common gotchas; a "Real Scenario" grounding the concept in an actual incident/bug; an ASCII diagram where the concept is spatial/structural.
- **Sourcing rule:** Official Docker docs preferred over blog conventions when they disagree; that disagreement is called out explicitly.
- **Format:** Clickable TOC, "back to top" links, cheat-sheet tables at the end of major parts, fenced code blocks with language tags.
- **To update this doc later:** keep this section, keep the structure below, keep the sourcing/accuracy discipline (re-verify anything version-specific before changing it).

---

## Table of Contents

- [Part 0: CLI & Networking Primer](#part-0-cli--networking-primer)
- [Part 1: Fundamentals](#part-1-fundamentals)
    - [1.1 Containers vs. VMs](#11-containers-vs-vms)
    - [1.2 Images, Containers, and Layers](#12-images-containers-and-layers)
    - [1.3 The Client/Daemon Model](#13-the-clientdaemon-model)
- [Part 2: Core Workflow](#part-2-core-workflow)
    - [2.1 Writing a Dockerfile](#21-writing-a-dockerfile)
    - [2.2 Build Context and .dockerignore](#22-build-context-and-dockerignore)
    - [2.3 Layer Caching](#23-layer-caching)
    - [2.4 Volumes and Bind Mounts](#24-volumes-and-bind-mounts)
    - [2.5 Networking Between Containers](#25-networking-between-containers)
- [Part 3: Docker Compose](#part-3-docker-compose)
    - [3.1 Why Compose](#31-why-compose)
    - [3.2 Anatomy of a Compose File](#32-anatomy-of-a-compose-file)
    - [3.3 The `version:` Key Is Obsolete](#33-the-version-key-is-obsolete)
    - [3.4 Day-to-Day Compose Commands](#34-day-to-day-compose-commands)
- [Part 4: Production Concerns](#part-4-production-concerns)
    - [4.1 Multi-Stage Builds](#41-multi-stage-builds)
    - [4.2 Image Size and Attack Surface](#42-image-size-and-attack-surface)
    - [4.3 Running as Non-Root](#43-running-as-non-root)
    - [4.4 Health Checks](#44-health-checks)
    - [4.5 Configuration and Secrets](#45-configuration-and-secrets)
    - [4.6 Logging](#46-logging)
- [Part 5: Real-World Patterns](#part-5-real-world-patterns)
    - [5.1 Dev/Prod Parity](#51-devprod-parity)
    - [5.2 Reverse Proxy in Front of Containers](#52-reverse-proxy-in-front-of-containers)
    - [5.3 Build-Once, Promote-Everywhere](#53-build-once-promote-everywhere)
- [Cheat Sheets](#cheat-sheets)
- [Suggested Learning/Reference Order](#suggested-learningreference-order)
- [Quick Self-Check](#quick-self-check)

---

## Part 0: CLI & Networking Primer

You don't need to be a Linux expert, but a few concepts get referenced constantly below, so here they are up front.

**Shell basics you'll actually use:**

- A "shell" (bash, zsh, PowerShell) is just the program that reads the commands you type and runs them.
- `sudo` (Linux/macOS) runs a command with administrator privileges — Docker commands often need this unless your user is in the `docker` group.
- Flags like `-d` (detached) or `-p 8080:80` (port mapping) modify what a command does; `--` (double dash) usually precedes a full word (`--name`), a single `-` a short letter (`-p`).

**Networking basics you'll actually use:**

- An **IP address** identifies a machine (or container) on a network; a **port** identifies a specific service on that machine (e.g., port 80 for HTTP).
- **Port mapping/publishing** (`-p host_port:container_port`) connects a port on your real machine to a port inside a container, since containers are otherwise isolated.
- **DNS** lets you use names instead of IP addresses. Docker gives containers on the same user-defined network automatic name-based DNS — this becomes important in [2.5](#25-networking-between-containers).
- **localhost** inside a container refers to _that container_, not your host machine or other containers — a extremely common source of confusion covered in the Real Scenario below.

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Part 1: Fundamentals

### 1.1 Containers vs. VMs

A **virtual machine** virtualizes an entire computer, including its own kernel — it's heavy but fully isolated. A **container** instead shares the host machine's kernel and uses OS-level isolation (Linux namespaces and cgroups) to make a process _believe_ it has its own filesystem, network stack, and process tree. This is why containers start in milliseconds instead of the minutes a VM takes, and why a single host can run dozens of containers with modest overhead.

```
VM MODEL                          CONTAINER MODEL
┌──────────────┐ ┌──────────────┐  ┌──────────────┐ ┌──────────────┐
│   App A      │ │   App B      │  │   App A      │ │   App B      │
│  Bins/Libs   │ │  Bins/Libs   │  │  Bins/Libs   │ │  Bins/Libs   │
│  Guest OS    │ │  Guest OS    │  ├──────────────┤ ├──────────────┤
├──────────────┤ ├──────────────┤  │        Docker Engine          │
│         Hypervisor            │  ├────────────────────────────────┤
├────────────────────────────────┤  │          Host OS Kernel        │
│          Host OS               │  ├────────────────────────────────┤
├────────────────────────────────┤  │           Hardware             │
│           Hardware              │  └────────────────────────────────┘
└────────────────────────────────┘
```

The practical consequence: containers are **not** a lightweight VM. Anything that assumes kernel-level isolation (running a different OS, or treating a container as a security boundary against a fully hostile workload) needs extra hardening — see [4.2](#42-image-size-and-attack-surface).

### 1.2 Images, Containers, and Layers

An **image** is a read-only template — a filesystem snapshot plus metadata (default command, exposed ports, env vars). A **container** is a running (or stopped) _instance_ of an image, with a thin writable layer on top for any changes made at runtime.

- One image → many containers, the same way one class can produce many objects.
- Each instruction in a Dockerfile (`RUN`, `COPY`, `ADD`) typically creates a new, cached **layer**. Layers are stacked and shared between images that have common ancestry, which is why pulling a second image based on the same parent is often fast.

**Real Scenario:** A team notices `docker run myapp` behaves differently from what's on a colleague's machine. Root cause: the colleague had been iterating with `docker exec` to patch files _inside a running container_, then just kept using that container instead of rebuilding the image. The image on disk never changed. Containers are disposable; images are the source of truth — if it's not in the Dockerfile, it doesn't really exist.

### 1.3 The Client/Daemon Model

The `docker` command you type is a **client**. It talks (usually over a local Unix socket, `/var/run/docker.sock`) to the **Docker daemon** (`dockerd`), a background service that does the actual work: pulling images, creating containers, managing networks and volumes. This is why `docker ps` still works even after you close your terminal — the daemon, and the containers it manages, keep running independently of the client that issued the command.

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Part 2: Core Workflow

### 2.1 Writing a Dockerfile

A Dockerfile is a recipe: a sequence of instructions that produce an image.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

Key instructions:

- `FROM` — the base image to build on. Pin a specific tag (`node:22-bookworm-slim`), never rely on the implicit `latest`, or you'll get surprise breakage when the tag moves.
- `WORKDIR` — sets the working directory for subsequent instructions (and creates it if needed).
- `COPY` — copies files from the build context into the image. (`ADD` does this too, plus auto-extracts archives and can fetch URLs — official guidance is to prefer `COPY` unless you specifically need `ADD`'s extra behavior, since the "magic" can surprise you.)
- `RUN` — executes a command _at build time_, producing a new layer.
- `CMD` — the default command run when a container _starts_. Prefer the exec form (`["node", "server.js"]`) over the shell form (`node server.js`) — the exec form doesn't wrap your process in a shell, so signals like `SIGTERM` reach it directly for clean shutdowns.

**Wrong vs. Right — dependency installation:**

```dockerfile
# WRONG — copies everything before installing deps,
# so any source change invalidates the npm install layer
FROM node:22-bookworm-slim
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

```dockerfile
# RIGHT — copies only the dependency manifests first,
# so npm ci is only re-run when dependencies actually change
FROM node:22-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

The difference is entirely about layer caching order — see [2.3](#23-layer-caching).

Source: [Dockerfile reference (Docker Docs)](https://docs.docker.com/reference/dockerfile/)

### 2.2 Build Context and .dockerignore

When you run `docker build .`, that `.` is the **build context** — the entire directory gets sent to the daemon before any instruction runs, even if your Dockerfile only `COPY`s a fraction of it. Two consequences:

1. A huge context (e.g., an accidentally-included `node_modules` or `.git` folder) slows every build down.
2. Anything in the context _could_ leak into an image layer, even if a later step doesn't reference it directly — a good reason to keep secrets and local-only files out entirely.

A `.dockerignore` file (same syntax as `.gitignore`) fixes both:

```
.git
node_modules
*.md
.env*
Dockerfile*
docker-compose*
```

### 2.3 Layer Caching

Docker caches each layer and reuses it on the next build **if and only if** the instruction and everything before it are unchanged. This is why instruction _order_ matters: put rarely-changing steps (installing dependencies) before frequently-changing steps (copying application source), so an edit to your source code doesn't force a full dependency reinstall.

```
Dockerfile order:           Cache behavior on a source-only change:
FROM base        ─┐
WORKDIR /app       │ cached  (unchanged)
COPY package*.json │ cached  (unchanged)
RUN npm ci          ┘ cached  (unchanged) ← the expensive step is skipped
COPY . .           ─┐ invalidated (this is what changed)
CMD [...]           ┘ invalidated (everything after a cache miss reruns)
```

**Real Scenario:** A CI pipeline's Docker build step that used to take 40 seconds started taking 6 minutes on every single commit. The Dockerfile copied the whole repo _before_ installing dependencies, so every commit — even a one-line README fix — invalidated the dependency-install layer and forced a full reinstall. Reordering to copy manifests first (per [2.1](#21-writing-a-dockerfile)) brought it back down to seconds for anything that didn't touch dependencies.

### 2.4 Volumes and Bind Mounts

Containers are ephemeral by default — delete the container, lose its writable layer. Two mechanisms persist or share data:

- **Volumes** — storage that Docker creates and manages (under `/var/lib/docker/volumes` on Linux), referenced by name. Official guidance treats these as the preferred mechanism for persisting data generated by containers.
- **Bind mounts** — a specific host path mounted directly into the container. You own the path; Docker doesn't manage it.

|             | Volumes                                        | Bind mounts                                                   |
| ----------- | ---------------------------------------------- | ------------------------------------------------------------- |
| Managed by  | Docker                                         | You (the host filesystem)                                     |
| Good for    | Databases, app state you want Docker to own    | Injecting host-owned files: source code in dev, config, certs |
| Portability | High — doesn't depend on host directory layout | Low — depends on an exact host path existing                  |

```bash
# Named volume — Docker manages the storage
docker run -d -v db_data:/var/lib/postgresql/data postgres:17

# Bind mount — you point at an exact host path (common in local dev,
# so your live source code shows up inside the container)
docker run -d -v "$(pwd)":/app -w /app node:22-bookworm-slim npm run dev
```

Source: [Volumes (Docker Docs)](https://docs.docker.com/engine/storage/volumes/)

### 2.5 Networking Between Containers

By default, every container attaches to Docker's built-in `bridge` network — but containers on that default network can only reach each other by IP address, which is fragile (IPs change). The fix, and the officially recommended pattern, is a **user-defined bridge network**: containers on it get automatic DNS resolution by container name.

```bash
docker network create app-network

docker run -d --name db --network app-network postgres:17
docker run -d --name api --network app-network \
  -e DB_HOST=db myapp:latest
# inside `api`, the hostname "db" resolves automatically — no IP needed
```

**Wrong vs. Right:**

```bash
# WRONG — both containers on the default bridge network;
# "api" has no reliable way to find "db" by name
docker run -d --name db postgres:17
docker run -d --name api -e DB_HOST=db myapp:latest   # DNS lookup for "db" fails
```

```bash
# RIGHT — shared user-defined network gives automatic DNS
docker network create app-network
docker run -d --name db --network app-network postgres:17
docker run -d --name api --network app-network -e DB_HOST=db myapp:latest
```

**Real Scenario:** "It works when I run it manually but the app can't reach the database" is one of the most common Docker support questions — almost always traced to two containers started separately without a shared user-defined network, so the app is trying to resolve a hostname that has no DNS entry anywhere.

Source: [Bridge network driver (Docker Docs)](https://docs.docker.com/engine/network/drivers/bridge/)

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Part 3: Docker Compose

### 3.1 Why Compose

Manually running `docker network create`, then multiple `docker run` commands with matching flags, doesn't scale past a couple of containers and isn't reproducible. **Compose** lets you describe an entire multi-container application — services, networks, volumes — in one YAML file, then bring it all up or down with a single command.

### 3.2 Anatomy of a Compose File

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

Notes:

- Every top-level entry under `services` gets its own container, and Compose automatically creates a shared user-defined network for the project — services can already reach each other by service name (`db`, `api`) with no manual network setup, unlike the raw `docker run` case in [2.5](#25-networking-between-containers).
- `depends_on` with `condition: service_healthy` waits for the dependency's healthcheck to pass, not just for the container to start — see [4.4](#44-health-checks) for why "started" and "actually ready" are different things.
- Named volumes declared under top-level `volumes:` behave exactly as in [2.4](#24-volumes-and-bind-mounts).

### 3.3 The `version:` Key Is Obsolete

Older tutorials show Compose files starting with `version: "3.8"` or similar. **Current official guidance is to omit this entirely.** The old 2.x/3.x versioned formats have been merged into a single "Compose Specification" that the modern CLI always parses with the latest schema regardless of any `version:` value — the field is now purely informational and produces an obsolete-field warning if present. This is exactly the kind of thing an older blog post will get wrong, so if you see `version:` in a tutorial, drop it.

Source: [Version and name top-level elements (Docker Docs)](https://docs.docker.com/reference/compose-file/version-and-name/)

### 3.4 Day-to-Day Compose Commands

```bash
docker compose up -d          # build (if needed), create, and start everything, detached
docker compose logs -f api    # stream logs for one service
docker compose exec api sh    # shell into a running service's container
docker compose down           # stop and remove containers + the project's network
docker compose down -v        # ...and also remove named volumes (destructive!)
```

Note the command is `docker compose` (a space, built into the Docker CLI) rather than the older standalone `docker-compose` binary — the hyphenated form is the legacy v1 tool and shouldn't be your starting point for a new setup.

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Part 4: Production Concerns

### 4.1 Multi-Stage Builds

A single-stage Dockerfile that compiles code tends to ship the compiler, build caches, and dev dependencies in the final image — bloated and a larger attack surface. **Multi-stage builds** use more than one `FROM` in the same Dockerfile; each `FROM` starts a fresh stage, and you selectively copy only the finished artifacts forward, leaving the build tooling behind.

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: build
FROM node:22-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

# Stage 2: runtime — only the build output and prod deps come along
FROM node:22-bookworm-slim
ENV NODE_ENV=production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The final image never contains the TypeScript compiler, dev dependencies, or intermediate build files — only what's needed to actually run the app.

Source: [Multi-stage builds (Docker Docs)](https://docs.docker.com/build/building/multi-stage/)

### 4.2 Image Size and Attack Surface

Every package in your final image is something that can have a vulnerability. Official current guidance favors minimal base image variants:

- `-slim` variants (Debian-based, no build tooling) — a reasonable default for most apps.
- `alpine` variants — smaller still, but based on musl libc rather than glibc, which occasionally breaks native dependencies that assume glibc.
- **Distroless** images — no shell, no package manager at all; smallest attack surface, but harder to debug (you can't `docker exec ... sh` into them).

Avoid `latest` and avoid unpinned generic images (`ubuntu:latest`) in production — pin a specific tag (or a digest, for full reproducibility) so a base-image update doesn't silently change your build.

### 4.3 Running as Non-Root

By default, a process in a container that doesn't specify otherwise runs as **root** — same privilege level the daemon itself often has. If an attacker breaks out of the application (e.g., via a code injection vulnerability), running as root inside the container makes any container-escape vector far more dangerous.

**Wrong vs. Right:**

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

(Many official images, including `node`, already ship a non-root `node` user you can switch to — check whether your base image already provides one before creating a fresh one with `useradd`.)

### 4.4 Health Checks

By default, Docker (and Compose) only knows whether a container's main **process** is alive, not whether the service inside it is actually able to serve traffic. A `HEALTHCHECK` closes that gap by running a command on an interval and marking the container `healthy` or `unhealthy` based on its exit code.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD ["node", "healthcheck.js"]
```

Use the exec form (`CMD [...]`) rather than the shell form here too, especially for minimal/shell-less images, so the check doesn't depend on a shell being present.

**Real Scenario:** A service starts, opens its port immediately, but takes 15 seconds to finish loading a large in-memory cache before it can actually answer requests correctly. Without a healthcheck, `depends_on` (in Compose) or a load balancer sees "port is open" and routes traffic immediately, resulting in a burst of errors during every deploy. A healthcheck that pings a real readiness endpoint — not just "is the process alive" — fixes this by making dependents wait for `healthy`, not just `started`.

### 4.5 Configuration and Secrets

Bake as little environment-specific config into the image as possible; inject it at runtime instead (`environment:` / `env_file:` in Compose, or `-e` on `docker run`). This is what makes "build once, promote the same image through every environment" (see [5.3](#53-build-once-promote-everywhere)) possible.

For actual secrets (API keys, DB passwords), avoid `ENV` or `ARG` in a Dockerfile — both can end up visible in image history or layer metadata. Prefer runtime injection (env vars supplied only when the container starts, not baked into the image) or Docker's dedicated build-time secret mounts for anything needed only during the build.

### 4.6 Logging

The default logging model is simple: write to `stdout`/`stderr`, and let `docker logs` (or `docker compose logs`) capture it — don't have your application manage its own log files inside the container. This keeps the container's filesystem stateless and lets you swap the logging backend (e.g., forwarding to a centralized log system) without touching application code.

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Part 5: Real-World Patterns

### 5.1 Dev/Prod Parity

The goal: your Compose setup for local development should exercise the _same_ image and topology as production, differing only in configuration — not "a totally different set of Dockerfiles that happen to run the same app." A common way to do this without duplicating everything: a base Compose file plus an override file for local-only additions (extra debug ports, bind-mounted source for hot reload).

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

### 5.2 Reverse Proxy in Front of Containers

A very common single-host production pattern: one reverse proxy container (Nginx, Caddy, Traefik) is the only thing with a port published to the outside world; every application container sits behind it on the internal user-defined network and is reached by service name.

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

This means application containers themselves never need a port published to the host — only the proxy does — reducing your actual attack surface.

### 5.3 Build-Once, Promote-Everywhere

Rebuilding an image separately for each environment (dev image, staging image, prod image) invites exactly the "works in staging, breaks in prod" class of bug, because the artifact itself differs, not just its configuration. The pattern that holds up: build **one** image, tag it, and promote that same tag through staging into production; only the environment configuration (env vars, secrets, replica count) changes between environments, never the image contents.

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Cheat Sheets

### Core CLI

| Command                                      | What it does                                                          |
| -------------------------------------------- | --------------------------------------------------------------------- |
| `docker build -t name:tag .`                 | Build an image from the Dockerfile in the current directory           |
| `docker run -d -p 8080:80 name:tag`          | Run a container, detached, mapping host port 8080 → container port 80 |
| `docker ps` / `docker ps -a`                 | List running / all containers                                         |
| `docker logs -f <container>`                 | Stream a container's logs                                             |
| `docker exec -it <container> sh`             | Open a shell inside a running container                               |
| `docker stop` / `docker rm`                  | Stop / remove a container                                             |
| `docker image prune` / `docker system prune` | Reclaim disk space from unused images/objects                         |
| `docker network create <name>`               | Create a user-defined bridge network                                  |

### Compose CLI

| Command                            | What it does                                       |
| ---------------------------------- | -------------------------------------------------- |
| `docker compose up -d`             | Build (if needed) and start all services, detached |
| `docker compose down`              | Stop and remove containers + project network       |
| `docker compose down -v`           | ...and also remove named volumes                   |
| `docker compose logs -f <service>` | Stream one service's logs                          |
| `docker compose exec <service> sh` | Shell into a running service                       |
| `docker compose build --no-cache`  | Force a full rebuild, ignoring the layer cache     |

### Dockerfile Instructions

| Instruction   | Purpose                                                                                |
| ------------- | -------------------------------------------------------------------------------------- |
| `FROM`        | Base image                                                                             |
| `WORKDIR`     | Set working directory for subsequent instructions                                      |
| `COPY`        | Copy files from build context into the image                                           |
| `RUN`         | Execute a command at build time (new layer)                                            |
| `ENV`         | Set an environment variable (baked into the image — not for secrets)                   |
| `EXPOSE`      | Document which port the container listens on (informational only — doesn't publish it) |
| `USER`        | Drop to a non-root user for subsequent instructions and at runtime                     |
| `HEALTHCHECK` | Define how Docker checks whether the container is actually healthy                     |
| `CMD`         | Default command when the container starts (exec form preferred)                        |

[↑ back to top](#docker-a-working-reference-single-host--compose)

---

## Suggested Learning/Reference Order

1. **Part 0 + Part 1** — get the mental model (images vs. containers, client/daemon) before touching commands.
2. **Part 2** — write and run a single-service Dockerfile against your own app; get comfortable with caching order and volumes/networking with plain `docker run`.
3. **Part 3** — move to Compose once you have 2+ services that need to talk to each other (e.g., app + database).
4. **Part 4** — once something works, harden it: multi-stage build, non-root, healthcheck, proper config/secret handling.
5. **Part 5** — once you're deploying somewhere real, adopt the reverse-proxy and build-once-promote patterns.

Come back to Part 4 specifically whenever you're about to ship a new service to production — it's the part most tutorials skip.

## Quick Self-Check

- Why does editing your source code sometimes force a full dependency reinstall in a Docker build — and how do you prevent it?
- Two containers started with plain `docker run` can't reach each other by name. What's missing?
- What's actually wrong with using `latest` as a base image tag in production?
- Why doesn't `EXPOSE` alone make a port reachable from outside the container?
- Why is running a container process as root riskier than it sounds, even though the container is "isolated"?
- What's the practical difference between a container being "started" and being "healthy," and which one should a dependent service wait for?
- Why is removing the `version:` key from a Compose file now the recommended practice instead of a lint nitpick?
- What problem do multi-stage builds solve that a `.dockerignore` file doesn't?
- In the build-once-promote-everywhere pattern, what's allowed to differ between staging and production, and what isn't?

[↑ back to top](#docker-a-working-reference-single-host--compose)
