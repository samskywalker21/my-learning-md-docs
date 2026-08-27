# Mastering Docker — A Reference Guide

## About This Document

- **Framing:** Goal-driven — optimized for actually developing and deploying production apps in Docker, not general interest.
- **Scope:** Single-app containers → Docker Compose multi-service setups → production hardening. Kubernetes/orchestration deliberately out of scope (a separate topic).
- **Assumed background:** None — written from zero Docker experience.
- **Examples:** Generic/stack-agnostic by request (not tied to Sam's AdonisJS/Nuxt stack).
- **Sourcing standard:** Version-specific details checked against official docs at `docs.docker.com` as of August 2026, linked inline. Where a common blog convention conflicts with official docs, the official docs win and the conflict is flagged.
- **Update instructions for future-me:** Match this structure and citation style if asked to update. Keep the wrong-vs-right snippet pattern for gotchas, and keep Kubernetes out unless explicitly asked to add it as a new section.

---

## Table of Contents

1. [What Docker Actually Is](#1-what-docker-actually-is)
2. [Getting Started](#2-getting-started)
3. [Images & Dockerfiles](#3-images--dockerfiles)
4. [Containers & Runtime](#4-containers--runtime)
5. [Docker Compose](#5-docker-compose)
6. [Networking & Volumes in Depth](#6-networking--volumes-in-depth)
7. [Production Hardening](#7-production-hardening)
8. [Registries & CI/CD](#8-registries--cicd)
9. [Deploying to Production](#9-deploying-to-production)
10. [Real-World Patterns & Common Mistakes](#10-real-world-patterns--common-mistakes)
11. [Cheat Sheets](#11-cheat-sheets)
12. [Suggested Learning Order](#12-suggested-learning-order)
13. [Quick Self-Check](#13-quick-self-check)

---

## 1. What Docker Actually Is

**What it is.** Docker packages an application together with everything it needs to run — code, runtime, libraries, system tools — into a single unit called an **image**. A running instance of that image is a **container**. Containers share the host machine's kernel but are isolated from each other and from the host, unlike a virtual machine which virtualizes an entire OS.

```
 Virtual Machine                          Container
 ┌─────────────────────┐                  ┌─────────────────────┐
 │  App A  │  App B     │                  │  App A  │  App B     │
 │  Bins/  │  Bins/     │                  │  Bins/  │  Bins/     │
 │  Libs   │  Libs      │                  │  Libs   │  Libs      │
 │  Guest  │  Guest     │                  ├─────────┴────────────┤
 │  OS     │  OS        │                  │   Docker Engine       │
 ├─────────┴────────────┤                  ├────────────────────────┤
 │      Hypervisor       │                  │      Host OS Kernel     │
 ├─────────────────────┤                  └────────────────────────┘
 │      Host OS Kernel     │
 └─────────────────────┘
```
Each VM ships a full guest OS; containers share one kernel, which is why they start in milliseconds instead of minutes and use a fraction of the resources.

**Why it matters.** The classic "it works on my machine" problem exists because your machine, your teammate's machine, and the production server all have subtly different library versions, OS patches, and configuration. An image is a single artifact that runs identically everywhere Docker runs — dev laptop, CI runner, production server.

**Real scenario.** A team ships an app that depends on a specific native library version. It works for every developer on the team, then breaks in production because the server's OS shipped a different version of that library. Docker doesn't just make this less likely — it makes the *exact* runtime environment reproducible and versionable, so "what changed?" has a git-diffable answer instead of a scavenger hunt through server configuration.

[⬆ back to top](#table-of-contents)

---

## 2. Getting Started

**Install.** Docker Desktop bundles the Engine, CLI, Compose, Buildx, and Docker Scout in one installer — the standard way to get everything working on Windows and macOS. As of mid-2026, Docker Desktop is at version 4.66, running Docker Engine v28.5. ([get-started](https://docs.docker.com/get-started/))

- **Windows:** download Docker Desktop, and during setup keep the **"Use WSL 2 instead of Hyper-V"** option checked — this is the current recommended backend on Windows.
- After install, restart, and Docker Desktop starts automatically in the system tray.

**First container:**
```bash
docker run hello-world
```
This pulls a tiny test image (if not already cached locally) and runs it — confirming your install works end to end.

**Core commands you'll use constantly:**
```bash
docker images              # list images cached locally
docker ps                  # list running containers
docker ps -a                # list all containers, including stopped ones
docker build -t my-app:1.0 .   # build an image from a Dockerfile in the current directory
docker run -d -p 8080:80 my-app:1.0   # run it, detached, mapping host port 8080 to container port 80
docker logs -f <container>   # follow a container's stdout/stderr
docker exec -it <container> sh   # get a shell inside a running container
```

```bash
# Wrong — forgetting -d leaves the container attached to your terminal,
# so closing the terminal kills it
docker run -p 8080:80 my-app:1.0

# Right — detach it, and manage it with docker ps / docker logs / docker stop
docker run -d -p 8080:80 my-app:1.0
```

[⬆ back to top](#table-of-contents)

---

## 3. Images & Dockerfiles

**What it is.** A Dockerfile is a text recipe: a sequence of instructions Docker executes to assemble an image, one layer per instruction. `docker image history <image>` shows you exactly which instruction produced each layer. ([image-building best practices](https://docs.docker.com/get-started/09_image_best/))

**Why layer order matters — caching.** Docker caches each layer. If a layer's inputs haven't changed since the last build, Docker reuses the cached result instead of re-running it. This means instruction *order* is a performance decision, not just style.

```dockerfile
# Wrong — copying all source before installing dependencies busts the cache
# on every single code change, forcing a full dependency reinstall each time
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]

# Right — copy only the dependency manifest first; npm install is only
# re-run when package.json/package-lock.json actually change
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

**Pin your base image.** `FROM node:latest` silently builds something different every week and can break without warning. Pin to a specific version, ideally a digest, for reproducible builds:
```dockerfile
# Avoid — a moving target
FROM node:latest
# Better — a specific version + slim variant
FROM node:20.11-slim
# Best — pinned to an immutable digest
FROM node:20.11-slim@sha256:0e6f...
```

**Multi-stage builds.** A traditional single-stage build leaves every build tool and intermediate artifact in your final image — bulky and a larger attack surface. Multi-stage builds let you use one stage to compile/build, and copy only the finished artifact into a lean final stage. Recommended for essentially all application types. ([multi-stage builds](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/))

```dockerfile
# Stage 1: build
FROM node:20-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: runtime — only the built output and prod deps make it here
FROM node:20-slim AS final
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install --omit=dev
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```
The `builder` stage's compilers, dev dependencies, and source maps never reach the final image — smaller size, fewer CVEs to scan for.

**`.dockerignore`.** Works like `.gitignore` for the build context — files matching these patterns are never sent to the Docker daemon and can't accidentally end up `COPY`'d into an image. At minimum: `.env`, `.env.*`, `*.pem`, `*.key`, `node_modules`, `.git`. ([Dockerfile security hardening](https://www.decryptiondigest.com/blog/dockerfile-security-hardening-secure-base-image-cicd-guide))

**`EXPOSE` vs. actually publishing a port** — a common point of confusion:
```dockerfile
EXPOSE 8080
```
This is documentation only — it records which port the app listens on but does **not** publish it to the host. You still need `-p` at `docker run` time (or a Compose `ports:` mapping) to actually reach it. ([Dockerfile best practices](https://generalistprogrammer.com/tutorials/dockerfile-best-practices))

**Real scenario.** A 1.2 GB image that takes four minutes to rebuild after a one-line code change, versus a 150 MB image from the same app that rebuilds in seconds — the difference is almost entirely instruction order, multi-stage builds, and base image choice, not anything about the application code itself. ([Dockerfile best practices](https://generalistprogrammer.com/tutorials/dockerfile-best-practices))

[⬆ back to top](#table-of-contents)

---

## 4. Containers & Runtime

**Lifecycle:**
```
 docker create  ──▶  docker start  ──▶  (running)  ──▶  docker stop  ──▶  docker rm
       │                                    │
       └──────────── docker run = create + start ───────┘
```

**Common lifecycle commands:**
```bash
docker stop <container>     # graceful shutdown (SIGTERM, then SIGKILL after a timeout)
docker restart <container>
docker rm <container>       # remove a stopped container
docker rm -f <container>    # force-stop and remove
```

**Environment variables** — pass config at runtime rather than baking it into the image:
```bash
docker run -e NODE_ENV=production -e PORT=3000 my-app:1.0
```

**Basic networking** — by default, containers on the same user-defined bridge network can reach each other by container name:
```bash
docker network create my-net
docker run --network my-net --name db postgres:18
docker run --network my-net --name app my-app:1.0   # app can reach "db" by that hostname
```
(Full networking model in §6.)

**Basic persistence** — anything written inside a container's writable layer disappears when the container is removed. A named volume survives that:
```bash
docker volume create pgdata
docker run --network my-net --name db -v pgdata:/var/lib/postgresql/data postgres:18
```
(Full storage model in §6.)

[⬆ back to top](#table-of-contents)

---

## 5. Docker Compose

**What it is.** Compose defines and runs multi-container applications from a single YAML file — instead of remembering a long chain of `docker run` flags for every service, you declare the whole stack once.

**A note on the `version:` key — a real, recent change:** older Compose files started with a `version: "3.8"` line. In current Compose (the Compose Specification, Compose v2), that key is obsolete — Compose ignores it and prints a warning. Modern files start directly with `services:`. If you copy an example from an older tutorial with a `version:` key, it'll still work but will nag you; just delete the line. ([Compose healthchecks guide](https://toolsops.dev/en/guides/docker-compose-healthchecks))

**A realistic multi-service `compose.yaml`** (app + database + reverse proxy):
```yaml
services:
  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/appdb
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 30s

  db:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    depends_on:
      app:
        condition: service_healthy
    ports:
      - "80:80"

volumes:
  pgdata:
```
```bash
docker compose up -d       # start the whole stack, detached
docker compose logs -f app # follow one service's logs
docker compose down        # stop and remove containers (add -v to also remove volumes)
```

**Why `depends_on` alone isn't enough — a very common gotcha:**
```yaml
# Wrong — depends_on without a healthcheck only waits for the container to *start*,
# not for the database to actually be ready to accept connections. The app can
# crash-loop against a database that's technically "up" but not yet accepting queries.
services:
  app:
    depends_on:
      - db
  db:
    image: postgres:18

# Right — pair depends_on's long form with a healthcheck, so Compose actually
# waits for readiness, not just process start
services:
  app:
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
```
Compose creates services in dependency order and, when a dependency is marked `service_healthy`, waits for its healthcheck to pass before starting the dependent service. ([Compose services reference](https://docs.docker.com/reference/compose-file/services/))

**Referencing the container's own env vars inside a healthcheck command** needs a doubled `$$` — Compose interpolates `$VAR` itself, so `$$VAR` is how you pass a literal `$VAR` through to the container's shell:
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
```

[⬆ back to top](#table-of-contents)

---

## 6. Networking & Volumes in Depth

**Networking.** Docker networks support different drivers; the default is **bridge**. Containers on the same user-defined bridge network resolve each other by container/service name — this is what makes `DATABASE_URL: postgres://db:5432/...` work in the Compose example above without hardcoding an IP. ([Docker networking](https://blog.vighnesh153.com/2020/12/docker-volumes-and-networking.html))

```
        user-defined bridge network "my-net"
   ┌─────────────────────────────────────────┐
   │   app  ──resolves "db" by name──▶  db     │
   │                                     │      │
   │                                nginx (published: 80 → host)
   └─────────────────────────────────────────┘
```
Only the ports you explicitly publish (`ports:` in Compose, or `-p` on `docker run`) are reachable from outside this network — everything else stays internal by default, which is a security property worth keeping, not just a default to override.

**Volumes vs. bind mounts vs. tmpfs** — the three storage options, and when each fits:

| Type | Managed by | Use case |
|---|---|---|
| **Volume** | Docker | Persistent app/database data that should survive container removal |
| **Bind mount** | You (host filesystem path) | Sharing source code into a container during development, or config files |
| **tmpfs** | Memory only | Data that should never touch disk, lost when the container stops |

```bash
# Named volume — Docker manages where this actually lives
docker run -v pgdata:/var/lib/postgresql/data postgres:18

# Bind mount — direct path from your host filesystem
docker run -v "$(pwd)/src:/app/src" my-app:1.0
```
([Sharing local files with containers](https://docs.docker.com/get-started/docker-concepts/running-containers/sharing-local-files/), [Volumes](https://docs.docker.com/engine/storage/volumes/))

```bash
# Wrong — bind-mounting your whole project root gives the container write
# access to everything on your host under that path, including files it
# has no business touching (and vice versa: a compromised container can
# modify host files)
docker run -v "$(pwd):/app" my-app:1.0

# Right — bind-mount only what genuinely needs live syncing (source code
# for dev hot-reload), and use a named volume for anything that's actually
# data the app owns
docker run -v "$(pwd)/src:/app/src" -v pgdata:/var/lib/postgresql/data my-app:1.0
```

**Real scenario.** A container restarts (a routine deploy, a crash, a host reboot) and the database "loses" all its data — panic ensues. In nearly every case, this is because the data directory was never mounted to a volume in the first place: it lived only in the container's writable layer, which is wiped when the container is removed. This is the single most common Docker data-loss story, and the fix is always the same one-line volume mount.

[⬆ back to top](#table-of-contents)

---

## 7. Production Hardening

This section is the difference between "it runs in a container" and "it's safe to expose to the internet."

**Run as non-root.** By default, everything in a container — every `RUN`, `COPY`, `CMD` — executes as root. If an attacker exploits a vulnerability in your app, they inherit root inside the container, and depending on host configuration, a container escape can mean root on the *host*. ([Docker security hardening](https://jonesrussell.github.io/blog/docker-security-users/), [Dockerfile security hardening](https://www.decryptiondigest.com/blog/dockerfile-security-hardening-secure-base-image-cicd-guide))
```dockerfile
# Wrong — no USER instruction; the app runs as root, silently
FROM node:20-slim
WORKDIR /app
COPY . .
CMD ["node", "server.js"]

# Right — create and switch to an unprivileged user before CMD
FROM node:20-slim
WORKDIR /app
COPY --chown=node:node . .
USER node
CMD ["node", "server.js"]
```
(Many official images, like `node`, already ship a non-root `node` user you can switch to — check the image's docs before creating your own.)

**Keep secrets out of the image, at build time and at runtime.** Anything passed via `ARG` or baked into a layer via `COPY`/`ENV` can be extracted from the image by anyone who pulls it, even after later layers "remove" it. Use BuildKit secret mounts for build-time credentials — they're available only for the duration of a single `RUN` instruction and never land in a layer:
```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) && pip install --no-cache-dir -r requirements.txt
```
```bash
docker build --secret id=npm_token,src=./token.txt -t my-app .
```
For runtime secrets (database passwords, API keys the running app needs), inject them as environment variables from a vault or your platform's secret manager — never commit them into `compose.yaml` or the image. ([Build secrets](https://docs.docker.com/build/building/secrets.md), [Docker security guide](https://techoral.com/docker/phase-8-security.html))

**Resource limits.** Without limits, one runaway container can starve every other container on the host:
```bash
docker run --memory=512m --cpus=1 my-app:1.0
```
In Compose:
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
```

**Health checks are a production requirement, not a nicety.** Without one, a process can crash-loop, hang, or serve incomplete responses while Docker still reports the container as "running" — a false-positive state that costs real debugging time in an incident. (§5 covers the Compose syntax.)

**Scan images before shipping.** Docker Scout ships built into Docker Desktop; Trivy and Snyk are common alternatives. Fail the build on critical/high CVEs in CI rather than catching them after deploy:
```bash
docker scout cves my-app:latest
```
([Docker security best practices](https://www.salayan.com/blog/docker-security-best-practices-2026-a-practical-guide))

**A pre-production checklist worth keeping** (adapted from common industry guides — not exhaustive, but catches the expensive mistakes):
```
✅ Non-root USER in the Dockerfile
✅ Minimal, pinned base image
✅ No secrets in ENV, ARG, or image layers — BuildKit secret mounts for build-time creds
✅ .dockerignore excludes .env, .git, keys
✅ Image scanned — no critical CVEs
✅ Healthcheck defined for every long-running service
✅ Resource limits set (memory, CPU)
✅ Only necessary ports published
✅ Logs go to stdout/stderr, not files inside the container
```
([Docker tricks 2026](https://devkraken.com/blog/docker-tricks-2026/), [Docker security: rootless containers](https://techoral.com/docker/phase-8-security.html))

[⬆ back to top](#table-of-contents)

---

## 8. Registries & CI/CD

**What it is.** A registry stores and distributes images. Docker Hub is the default public registry; most teams also use a private registry (a cloud provider's, GitHub Container Registry, or a self-hosted one) for proprietary images.

**Tagging and pushing:**
```bash
docker build -t myregistry.example.com/my-app:1.2.0 .
docker push myregistry.example.com/my-app:1.2.0
```
```bash
# Wrong — tagging everything "latest" makes it impossible to know which
# actual build is running in production, or to roll back to a known-good one
docker build -t my-app:latest .

# Right — tag with something traceable: a semver, or the git commit SHA
docker build -t my-app:1.2.0 -t my-app:$(git rev-parse --short HEAD) .
```

**CI/CD pattern.** A typical pipeline: on merge to main, CI builds the image, scans it, pushes it to the registry tagged with the commit SHA, then triggers a deploy that pulls that exact tag. This is what makes "roll back to the previous version" a one-line operation instead of a rebuild-and-hope.

[⬆ back to top](#table-of-contents)

---

## 9. Deploying to Production

**Where Compose fits, and where it stops.** Docker Compose is well-suited to production for a single host — small-to-medium deployments, up to roughly 10–20 containers on one machine. Beyond that (multi-host, automatic scaling, self-healing, rolling updates across a fleet), you're in orchestrator territory — Kubernetes is the dominant option, deliberately out of scope for this document. Most teams start with Compose for development and a single-host deploy, and graduate to an orchestrator only once they actually need multi-host scaling. ([Docker tutorial 2026](https://tech-insider.org/docker-tutorial-beginners-containerization-2026/))

**Zero-downtime patterns worth knowing even on a single host:**
- Health-check-gated rollout: don't route traffic to a new container until its healthcheck passes (this is what the `service_healthy` pattern in §5 sets you up for).
- Keep the previous image tag around so a bad deploy is a `docker run <previous-tag>` away, not a rebuild.
- `docker compose up -d` with an unchanged `compose.yaml` for unaffected services only recreates the containers whose config actually changed — it won't blindly restart everything.

**Logging.** Send application logs to stdout/stderr rather than to files inside the container — this is what lets `docker logs` and any log-aggregation tooling (Loki, CloudWatch, etc.) pick them up without extra plumbing. A container writing logs only to its own internal filesystem loses them the moment it's removed.

[⬆ back to top](#table-of-contents)

---

## 10. Real-World Patterns & Common Mistakes

**"My database loses data on every restart."** Almost always a missing volume mount for the data directory (§6). Check `docker volume ls` — if you don't see a volume for it, that's the bug.

**"depends_on isn't working — my app crashes trying to reach the database."** `depends_on` without `condition: service_healthy` only waits for the container process to *start*, not for the service inside to be ready to accept connections (§5). Add a healthcheck.

**"My image is huge and takes forever to rebuild."** Almost always: no multi-stage build, dependencies re-copied/reinstalled before source code (busting the cache on every change), or a full OS base image instead of a slim/alpine variant (§3).

**"It works locally but breaks in the container."** Check for anything that assumes the host filesystem, host networking (`localhost` meaning something different inside vs. outside a container), or environment variables that exist on your machine but were never passed into the container.

**"A container I thought I removed is still using disk space / a port."** Stopped containers aren't automatically removed; `docker ps -a` shows them, `docker system prune` cleans up dangling images, stopped containers, and unused networks (review what it'll remove first — it's not selective by default).

**Common mistake — treating `docker run -v $(pwd):/app` as harmless.** It's fine for quick local experiments, but in anything approaching production it hands the container broad write access to your host filesystem under that path (§6) — scope bind mounts to exactly what needs to be shared.

[⬆ back to top](#table-of-contents)

---

## 11. Cheat Sheets

### Core CLI
| Command | Purpose |
|---|---|
| `docker build -t name:tag .` | Build an image from a Dockerfile |
| `docker run -d -p host:container image` | Run detached, with port mapping |
| `docker ps` / `docker ps -a` | List running / all containers |
| `docker logs -f <container>` | Follow logs |
| `docker exec -it <container> sh` | Shell into a running container |
| `docker stop` / `docker rm` | Stop / remove a container |
| `docker image history <image>` | See how an image's layers were built |
| `docker scout cves <image>` | Scan for known vulnerabilities |
| `docker system prune` | Clean up unused images/containers/networks |

### Compose
| Command | Purpose |
|---|---|
| `docker compose up -d` | Start the stack, detached |
| `docker compose down` | Stop and remove (`-v` also removes volumes) |
| `docker compose logs -f <service>` | Follow one service's logs |
| `docker compose build` | Rebuild images defined by `build:` |

### Storage decision
| Need | Use |
|---|---|
| Database/app data that must survive container removal | Named **volume** |
| Live-syncing source code during dev | **Bind mount** |
| Sensitive data that should never touch disk | **tmpfs** |

### Dockerfile security essentials
`USER` (non-root) · pinned base image · multi-stage build · `.dockerignore` · BuildKit `--mount=type=secret` for build-time creds · `HEALTHCHECK`

[⬆ back to top](#table-of-contents)

---

## 12. Suggested Learning Order

1. **Install Docker Desktop, run `hello-world`** (§2) — confirm the install works before touching a real app.
2. **Containerize one real app with a plain (single-stage) Dockerfile** (§3, §4) — get something running end to end before optimizing it.
3. **Convert it to a multi-stage build and add a non-root `USER`** (§3, §7) — see the image size drop, understand why.
4. **Add a second service (a database) with Docker Compose** (§5) — this is where `depends_on` + healthchecks starts to matter.
5. **Deliberately break the healthcheck/depends_on setup once**, on purpose, so you recognize the failure mode later (§5, §10).
6. **Add volumes for anything with state** (§6) — verify data survives a `docker compose down && docker compose up`.
7. **Run the pre-production checklist against your app** (§7) — non-root, secrets, resource limits, scanning.
8. **Push to a registry with a real tag, not `latest`** (§8).
9. **Only then** look at what changes for multi-host deployment — and treat that as the point where Kubernetes or another orchestrator becomes a separate, deliberate topic to learn.

## 13. Quick Self-Check

- What's the actual difference between an image and a container?
- Why does instruction *order* in a Dockerfile affect build speed, not just correctness?
- What does a multi-stage build remove from your final image that a single-stage build doesn't?
- Why isn't `depends_on` alone enough to guarantee a database is ready before your app starts?
- What's the difference between a named volume and a bind mount, and when would you pick each?
- Why is running a container as root a security risk even though the container is "isolated"?
- Where do build-time secrets end up if you pass them via `ARG` instead of a BuildKit secret mount — and why does that matter even after the image is built?
- At roughly what point does Docker Compose stop being the right tool for production, and what takes over?

[⬆ back to top](#table-of-contents)
