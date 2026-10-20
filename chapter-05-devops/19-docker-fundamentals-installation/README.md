# Cheatsheet 19 — Docker Fundamentals & Installation

**From Zero to Fullstack · Chapter 05: DevOps**  
**Audience:** Developers using Docker for the first time; JavaScript/React/Node.js developers preparing for deployment  
**Primary platform:** Windows 10/11 · PowerShell · Docker Desktop · WSL 2 · Visual Studio Code  
**Version:** 1.0 · 20 October 2026  
**Source repository:** [Chapter 05 — DevOps](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops)  
**Method:** **Learn → Practice → Document → Reuse**

> **Evidence and scope.** **SOURCE PROJECT** refers to the actual files inspected in the Chapter 05 GitHub repository. **INDEPENDENT LAB** refers to the separate, dependency-free Node.js API bundled in this cheatsheet's ZIP, not code already committed to that repository. **IMPROVEMENT** refers to a recommended change, not an existing implementation. The lab's **9 Node.js tests passed**, and the Compose YAML was parsed successfully. **Docker Engine and Docker Desktop were not available** in the authoring environment: image builds, Docker/WSL installation and container/Compose runtime behavior have **not** been executed or verified here. Follow the local commands to validate them on your machine.

## 1. Learning goals

By the end of the guide, you will be able to:

1. Explain **DevOps**, Docker and containerization in simple terms.
2. Distinguish **image, container, registry, Dockerfile, volume, network and Docker Compose**.
3. Install or verify **WSL 2** and **Docker Desktop** on a supported Windows computer.
4. Understand Docker's client, daemon and Linux-container execution model.
5. Pull, run, inspect, stop and remove simple containers.
6. Explain the difference between a **host port** and a **container port**.
7. Build your own Docker image for a minimal Node.js HTTP API.
8. Use `docker compose` to manage a service through a `compose.yaml` file.
9. Use named volumes and custom networks without confusing them with host folders.
10. Inspect the **real Docker assets** in Chapter 05 and identify what must be checked before running them.
11. Follow a safe workflow for secrets, user permissions, image tags and cleanup.
12. Reuse a working beginner laboratory in other projects.

### Your first mental model

```text
Your Windows PC
    |
    +-- PowerShell: docker CLI and docker compose
    |
    +-- Docker Desktop (Docker Engine on WSL 2/Linux)
           |
           +-- Image: reproducible read-only application template
           |       ^ built from Dockerfile or pulled from a registry
           |
           +-- Container A: running instance of an image
           +-- Container B: another isolated instance
           |
           +-- Network: communication between containers
           +-- Volume: persistent data outside a container lifecycle
```

**Key idea:** `Dockerfile` describes how to build an **image**. `docker run` creates a **container** from an image. Docker Compose coordinates **services**. This does not mean each container contains its own complete virtual machine.

## 2. What Docker solves — and what it does not

A modern web application may need Node.js, an HTTP server, a database, OS libraries and configuration. Without containers, different computers often have different versions and settings. Docker packages the application and its runtime dependencies into a **container image** so the execution environment is more consistent across machines.

**A container is not a virtual machine:** containers isolate processes while sharing the host kernel (for Linux containers on Windows, Docker Desktop runs a Linux environment through WSL 2). A VM normally runs a full guest operating system. Containers typically start faster and consume fewer resources.

Docker does **not** replace good code, tests, network security, backups, database administration or system monitoring. It reduces configuration drift; it does not guarantee that an application works.

| Traditional local development | Containerized development |
|---|---|
| Install language runtimes directly on the host | Build or pull images containing the required runtime |
| Configure dependencies on each machine | Record dependencies and build steps in a Dockerfile |
| Run multiple apps that may conflict | Isolate processes, filesystems and dependencies |
| Use localhost host-to-host | Publish ports; communicate service-to-service over networks |
| Store data in arbitrary local directories | Use explicit volumes or mounts for durable data |

## 3. Essential vocabulary

| Term | Beginner explanation | Example |
|---|---|---|
| **Docker Engine / daemon** | Service that builds images and manages containers | The engine Docker Desktop starts |
| **Docker CLI** | Command-line client that talks to the daemon | `docker ps` |
| **Image** | Immutable template assembled in layers | `nginx:alpine`, `node:22-alpine` |
| **Container** | Created/running instance of an image | `hello-api` |
| **Dockerfile** | Recipe for building an image | `FROM`, `COPY`, `RUN`, `CMD` |
| **Registry** | Place from which images are pulled/pushed | Docker Hub |
| **Volume** | Docker-managed persistent storage | `cs19-data` |
| **Bind mount** | A host folder mounted into a container | Windows project folder → `/app` |
| **Network** | Connectivity and name resolution among containers | Compose default network |
| **Compose** | Declarative tool for starting related services | `docker compose up` |
| **Build context** | Files Docker can access during build | `.` for current folder |

The **image/container** relationship resembles a reusable class and an object instance: many containers can be created from one image. The analogy is imperfect but useful.

## 4. Install Docker on Windows from zero

### 4.1. Check Windows, virtualization and WSL

Open **PowerShell**. Check your Windows version with `winver` (Run dialog) or through **Settings → System → About**. Check that your system meets Docker Desktop's current Windows requirements before installing: supported Windows release, 64-bit hardware, hardware virtualization and adequate RAM. Requirements change, so use the official installer documentation rather than assuming old minimums still apply.

In PowerShell, inspect WSL:

```powershell
wsl --status
wsl --version
wsl --list --verbose
```

If WSL is unavailable, open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

Reboot when requested. Then check or update WSL:

```powershell
wsl --update
wsl --status
```

**Note:** Docker's current WSL 2 documentation calls for **WSL 2.1.5 or newer**; prefer the latest stable release. Docker Desktop can run Linux containers from Windows PowerShell even if you have not manually installed a separate Ubuntu distribution.

If virtualization is disabled, enable it in firmware/BIOS/UEFI according to your PC manufacturer. Do not change firmware settings blindly. On managed work computers, consult your administrator.

### 4.2. Download and install Docker Desktop

1. Open the official [Docker Desktop installation guide for Windows](https://docs.docker.com/desktop/setup/install/windows-install/).
2. Download Docker Desktop from the official Docker website; avoid third-party installers.
3. Run the installer. Select the **WSL 2-based engine** option when offered.
4. Restart Windows if requested, then launch **Docker Desktop** from the Start menu.
5. Accept the license terms, review onboarding settings, and wait until the engine is ready.
6. In Docker Desktop, verify the use of **Linux containers** and, where applicable, **Settings → General → Use the WSL 2 based engine**.

**Licensing:** Docker Desktop is free for personal use, education, non-commercial open-source projects and qualifying small businesses. Other commercial or government uses may require a paid subscription. Check the [current licensing conditions](https://docs.docker.com/subscription-billing/desktop-license/) if using it professionally.

### 4.3. Validate your installation

Open **a fresh PowerShell terminal** after Docker Desktop has started:

```powershell
docker --version
docker version
docker info
docker compose version
docker run --rm hello-world
```

Expected: a Docker CLI version, **both Client and Server** sections in `docker version`, an active engine in `docker info`, a Compose v2 version, and a successful `hello-world` message. The first run may download an image, so an internet connection is normally needed.

**If `docker --version` works but `docker info` fails**, you have installed the CLI but may not have a working engine. Start Docker Desktop, check the WSL backend, and retry.

### 4.4. Quick installer checklist

- [ ] Windows version and virtualization support verified.
- [ ] `wsl --version` confirms a supported WSL 2 installation.
- [ ] Docker Desktop installed from its official source and running.
- [ ] Docker Desktop set to Linux containers for our Node.js exercises.
- [ ] `docker version` displays the client and the daemon/server.
- [ ] `docker compose version` responds.
- [ ] `docker run --rm hello-world` succeeds.

## 5. Docker commands: your first container

### 5.1. Pull, list and inspect an image

```powershell
docker pull nginx:alpine
docker image ls
docker image inspect nginx:alpine
```

An **image** is downloaded but not running until Docker creates a container from it. Tags (such as `alpine`) identify an image variant; tags can change over time. For repeatable builds, consider pinning an explicit version or digest when appropriate.

### 5.2. Run Nginx and publish a port

```powershell
docker run -d --name cs19-nginx -p 127.0.0.1:8088:80 nginx:alpine
docker ps
docker logs cs19-nginx
```

Visit **http://localhost:8088** in your browser. You should see the Nginx welcome page.

```text
Browser on Windows --> http://localhost:8088
                          host port 8088
                               |
                          Docker port mapping
                               |
                          container port 80
                               |
                          Nginx process
```

The syntax `-p 127.0.0.1:8088:80` means **bind only to the host loopback interface**, expose host port **8088**, and forward requests to container port **80**. Omitting `127.0.0.1` may expose the port more broadly depending on host firewall and Docker settings.

### 5.3. Inspect and clean up

```powershell
docker ps                # Running containers
docker ps -a             # All created containers
docker logs cs19-nginx
docker inspect cs19-nginx
docker stop cs19-nginx
docker start cs19-nginx
docker rm -f cs19-nginx  # Stop/remove this demo container
```

Stopping a container does **not** delete it; `docker rm` removes its writable layer. Removing a container does not delete the underlying image. Avoid broad `docker system prune` commands before you understand their effects.

## 6. Learn to read a Dockerfile

Our independent lab contains the following **Dockerfile**:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY --chown=node:node package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --chown=node:node src/ ./src/
ENV NODE_ENV=production
USER node
EXPOSE 8080
CMD ["node", "src/server.js"]
```

| Instruction | Meaning | Why it matters |
|---|---|---|
| `FROM` | Choose a base image | Reproducible runtime baseline |
| `WORKDIR` | Set working directory inside image | Predictable paths |
| `COPY` | Add files from the build context | Include app files, not host dependencies |
| `RUN` | Execute during image build | Install locked dependencies |
| `ENV` | Define a runtime environment variable | Configure the process |
| `USER` | Use a non-root account | Reduce damage from compromise |
| `EXPOSE` | Document intended listening port | Does **not** publish a host port |
| `CMD` | Default command on container start | Start Node.js process |

**`RUN` vs `CMD`:** `RUN npm ci` happens when building the image. `CMD ["node", ...]` happens when a container starts. `COPY package*.json` before application source allows Docker to reuse dependency-installation layers when only code changes.

The lab uses **Node.js 22** in the image for a stable, documented example, independent of the source Chapter 05 files, which use **Node.js 20**. This is a teaching choice, not a modification to the original repository.

### Build context and `.dockerignore`

Docker reads files from its **build context**. A `.dockerignore` removes unwanted files from that context before build. Our lab ignores:

```gitignore
node_modules
npm-debug.log*
.git
.env
.env.*
.DS_Store
.vscode
test
README.md
```

Never copy passwords or private keys into an image. `.dockerignore` is **not** the same as `.gitignore`: excluding a file from Docker builds does not stop Git from tracking it.

## 7. Independent lab: build your first Node.js image

The ZIP includes a complete **`lab/`** directory. Its HTTP application uses only Node.js built-ins; there are no runtime npm libraries to download. Its dependency lockfile is included to demonstrate repeatable installation through `npm ci`.

### 7.1. Lab file tree

```text
lab/
├── Dockerfile
├── .dockerignore
├── compose.yaml
├── package.json
├── package-lock.json
├── src/
│   ├── app.js       # Handler with /, /health and /api/info
│   └── server.js    # Listen on 0.0.0.0:8080
└── test/
    └── app.test.js  # Built-in Node.js test runner
```

Unzip the package. Open PowerShell in the `lab` directory, **not** the Chapter 05 repository:

```powershell
node --version
npm.cmd test
npm.cmd start
```

In another terminal:

```powershell
Invoke-RestMethod http://localhost:8080/health
Invoke-RestMethod http://localhost:8080/api/info
```

Expected JSON:

```json
{ "status": "ok" }
```

Stop the process with **Ctrl+C**. Tests run the API on a temporary local port and clean up after themselves.

### 7.2. Build and run with Docker

With Docker Desktop running, in the same `lab` folder:

```powershell
docker build -t cs19-hello-api:1.0 .
docker image ls
docker run -d --name cs19-api -p 127.0.0.1:8080:8080 cs19-hello-api:1.0
docker ps
docker logs cs19-api
Invoke-RestMethod http://localhost:8080/health
Invoke-RestMethod http://localhost:8080/api/info
```

The `.` after `docker build` is the **build context**. Docker looks for `Dockerfile` in that directory by default. The app listens on `0.0.0.0` **inside the container**; binding only to `127.0.0.1` there would normally prevent external access through the published container port.

Clean up just this example:

```powershell
docker rm -f cs19-api
docker image rm cs19-hello-api:1.0
```

**Runtime status:** these are reproducible instructions, **not** claims that the image was built or run during authoring.

## 8. Compose: describe services instead of long commands

Docker Compose reads `compose.yaml` and manages services together. For this guide, a single-service Compose project demonstrates the basic syntax:

```yaml
name: cheatsheet19
services:
  hello-api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      NODE_ENV: production
    restart: unless-stopped
```

In the lab folder:

```powershell
docker compose config --quiet
docker compose up --build -d
docker compose ps
docker compose logs -f hello-api
```

Press **Ctrl+C** to stop following logs (the detached container remains running). Test the health endpoint in another terminal; then stop the demo:

```powershell
Invoke-RestMethod http://localhost:8080/health
docker compose down
```

`docker compose up --build` builds any required images and starts services; `-d` detaches the CLI. `docker compose down` stops and removes Compose-managed containers/networks, **not** named volumes unless `-v` is specified. **Do not run the manual `docker run` API and Compose API at the same time**: both would attempt to bind host port 8080.

## 9. Images, networking and storage exercises

### 9.1. Images and layers

An image is built in layers that Docker may cache. Changing application source after dependency installation usually means fewer layers need rebuilding. For reliable builds, prefer `npm ci` over `npm install` when an accurate `package-lock.json` exists.

```powershell
docker image ls
docker history cs19-hello-api:1.0
```

Run `docker history` only while the lab image exists. The image includes dependencies and application files, **not** a live database.

### 9.2. Named volumes: persist data

```powershell
docker volume create cs19-data
docker run --rm -v cs19-data:/data alpine:3.20 sh -c "echo hello > /data/message.txt"
docker run --rm -v cs19-data:/data alpine:3.20 cat /data/message.txt
docker volume inspect cs19-data
docker volume rm cs19-data
```

The second container prints `hello`, even though the first container was removed. This demonstrates why **database data must live outside the ephemeral container filesystem**. A Docker volume is not a backup. Back up important persistent data separately.

### 9.3. Custom networks

```powershell
docker network ls
docker network create cs19-net
docker network inspect cs19-net
docker network rm cs19-net
```

In a multi-container Compose app, containers typically resolve each other through **service names** on a shared network: for example, `mongodb://db:27017/blog` from a backend container when a Compose service is actually named `db`. Your **Windows browser** cannot resolve that Docker-internal service name; browser JavaScript needs an address reachable from the browser, typically a published localhost port or a real domain.

### 9.4. Volumes, bind mounts and exposed ports

| Feature | Purpose | Common example |
|---|---|---|
| Named volume | Durable data managed by Docker | MongoDB data directory |
| Bind mount | Share an existing host path | Hot reload from source code |
| `EXPOSE 8080` | Document container port | Dockerfile instruction |
| `-p 127.0.0.1:8080:8080` | Publish to Windows localhost | Access API from browser |
| Compose network | Service name resolution | `api` → `db:27017` |

## 10. Inspect the real Chapter 05 project (SOURCE PROJECT)

The actual [Chapter 05 source](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops) already contains a React frontend, an Express/Mongoose backend, two Dockerfiles and a Compose file. This first guide **inspects** them; the multi-service configuration will be taught in full in **Cheatsheet 20**.

```text
from-zero-to-fullstack-ch5-devops/
├── dockerfile                # SOURCE: lowercase filename (frontend)
├── .dockerignore
├── compose.yaml              # SOURCE: frontend + backend only
├── package.json              # Vite / React
├── src/                      # React code
└── backend/
    ├── dockerfile            # SOURCE: lowercase filename (backend)
    ├── .dockerignore
    ├── package.json
    └── src/                  # Express + Mongoose code
```

### 10.1. Source frontend Dockerfile

The source `dockerfile` starts from **`node:20`**, copies `package.json` and `package-lock.json`, runs `npm install` and `npm run build`, then copies `/build/dist` to an **Nginx** image. This is a **multi-stage build**: a Node.js build stage produces static frontend files; Nginx serves the final static files on port 80 inside its container.

### 10.2. Source backend Dockerfile

The backend `backend/dockerfile` starts from **`node:20`**, copies its lockfiles, runs `npm install`, copies source code and starts the app with `npm start`. The Compose file maps **host port 3001 to container port 3001** and passes `PORT=3001`.

### 10.3. Source Compose configuration

The actual `compose.yaml` defines two services:

| Service | Source build context | Ports | Configuration |
|---|---|---|---|
| `blog-backend` | `./backend` | `3001:3001` | `PORT=3001`, `DATABASE_URL=${DATABASE_URL}` |
| `blog-frontend` | `.` | `3000:80` | Build argument `VITE_BACKEND_URL` |

**Important:** there is **no MongoDB service** in this source Compose file. A working backend needs a reachable MongoDB instance via `DATABASE_URL`. A database running on `localhost` on your Windows host is **not** accessible via `localhost` from inside the backend container. For Docker Desktop host connectivity, `host.docker.internal` can be useful in local development; a properly configured Compose database service is preferable for a fully containerized setup. Do not confuse these patterns with production secrets management.

### 10.4. Four issues to check before using the source Compose file

1. **Lowercase Dockerfile filenames.** The source stores both as `dockerfile`, but the Compose configuration uses `build: ./backend` and `build.context: .` without naming the files. Docker looks for **`Dockerfile`** by default. **IMPROVEMENT:** rename files to `Dockerfile`, or explicitly set `build.dockerfile: dockerfile` for each service. Check case sensitivity when building on Linux/WSL 2.
2. **Missing database service.** The source requires `DATABASE_URL` from the Compose environment but does not define `mongodb`. A database must be separately accessible, and its data must be persisted appropriately. **IMPROVEMENT:** add a database service and a named volume in Cheatsheet 20.
3. **Legacy Compose `version`.** Source `compose.yaml` has `version: "3.9"`. Current Compose treats this top-level field as obsolete and may print a warning. **IMPROVEMENT:** remove the `version` line after checking compatibility.
4. **Build-time frontend URL and safety.** The frontend uses the Vite variable `VITE_BACKEND_URL` during the image build. Browser code needs a public/reachable API URL (for local tests, e.g. `http://localhost:3001/api/v1`). Never put secrets in variables prefixed `VITE_`: their values may be embedded into public browser assets.

### 10.5. Check tracked environment files safely

The inspected source Git tree lists a root `.env` file. **Being listed does not prove the file contains secrets**, and this cheatsheet does not read or expose its contents. However, committing a real `.env` file can be a serious risk. On your computer, verify tracking **without printing values**:

```powershell
git ls-files .env backend/.env
```

If a real secret has ever been committed, **rotate/revoke** it; removing the file in a later commit does not erase older Git history. For future changes, put `.env` into `.gitignore`, create a **sanitized `.env.example`** with placeholders, and use secret management in CI/production. `.dockerignore` protects the build context; it does **not** untrack Git files.

### 10.6. Corrected Compose *example* (not yet committed)

For clarity, here is a **minimal local correction** that still requires an external reachable database. Run only after reviewing your environment and configuring variables:

```yaml
services:
  blog-backend:
    build:
      context: ./backend
      dockerfile: dockerfile  # Existing lowercase source file
    environment:
      PORT: "3001"
      DATABASE_URL: ${DATABASE_URL:?Set DATABASE_URL before starting}
    ports:
      - "127.0.0.1:3001:3001"

  blog-frontend:
    build:
      context: .
      dockerfile: dockerfile  # Existing lowercase source file
      args:
        VITE_BACKEND_URL: ${VITE_BACKEND_URL:-http://localhost:3001/api/v1}
    ports:
      - "127.0.0.1:3000:80"
    depends_on:
      - blog-backend
```

**This is a learning illustration, not a tested source patch.** It deliberately does not add MongoDB or solve every application-level issue. `depends_on` controls creation/start ordering; it is **not** by itself proof that the API or MongoDB is ready. `DATABASE_URL` should be supplied securely and must point to the reachable database.

## 11. Useful verification and troubleshooting

| Symptom | Likely cause | Next check / remedy |
|---|---|---|
| `docker` command not recognized | CLI not installed / PATH not refreshed | Install Docker Desktop, restart terminal |
| Cannot connect to Docker daemon | Docker Desktop stopped / WSL problem | Start Desktop; `docker version`, `wsl --status` |
| WSL installation fails | Virtualization or Windows feature disabled | Confirm firmware virtualization, Windows version, admin permissions |
| `docker run hello-world` cannot pull | No network / registry DNS / proxy issue | Test connectivity and Docker proxy settings |
| `bind: address already in use` | Host port is occupied | Run `docker ps` or choose an unused host port |
| Nginx site works but React API fails | Wrong Vite API URL / CORS / backend unavailable | Browser Network tab, backend logs, build arg |
| `failed to read dockerfile` | File is lowercase `dockerfile` | Rename or specify `build.dockerfile` explicitly |
| Backend cannot connect to MongoDB | No MongoDB service / wrong URL / DNS | Verify `DATABASE_URL`, connectivity and network |
| Data disappears after deletion | Saved only inside container filesystem | Use a named volume plus backups |
| `docker compose config` shows a warning | Obsolete `version` key or unresolved variables | Update schema and supply environment values |
| Node app works on Windows but not from published port | Server bound to `127.0.0.1` inside container | Bind to `0.0.0.0` inside app; publish host port |
| Build tries to use a missing file | `.dockerignore` removed required context item | Review paths and ignore rules |

### Diagnostic sequence

```powershell
docker version
docker info
docker ps -a
docker images
docker compose config
docker compose ps
docker compose logs --tail=100
```

Run Compose commands in the directory containing the relevant `compose.yaml`. `docker compose config` can interpolate and print environment variable values, including secrets; **never paste unredacted output into a public issue or chat**. Use `docker compose config --quiet` when only validation is needed.

## 12. Security and engineering best practices

- **Never put secrets inside source code or image layers.** Prefer runtime secrets and an appropriately configured secret manager.
- **Avoid `latest` for production release reproducibility.** Pin trusted images to reviewed tags/digests and update deliberately.
- **Run as a non-root user** inside containers whenever practical; our lab uses `USER node`.
- **Minimize build context** with `.dockerignore` and exclude `.git`, `.env*`, `node_modules` and build outputs.
- **Avoid open ports by default.** Bind local demo endpoints to `127.0.0.1` on the host.
- **Use the exec form of `CMD`** for predictable signal handling.
- **Do not assume images are secure because they are official.** Track base-image updates and vulnerabilities.
- **Use the correct configuration phase:** Docker build `ARG` vs runtime `ENV`; Vite `VITE_` variables become browser-visible.
- **Persist and back up database data.** Volumes survive container deletion, but volumes are not backups.
- **Use health checks** for readiness when multi-container dependencies require them; `depends_on` alone does not guarantee an app is ready.
- **Limit cleanup scope.** Avoid `docker system prune -a` and `docker volume prune` unless you understand all resources that would be removed.

## 13. Quick reference: practical Docker commands

| Task | Command |
|---|---|
| Check client | `docker --version` |
| Check client + server | `docker version` |
| Check Compose | `docker compose version` |
| Verify engine | `docker info` |
| Verify image download/run | `docker run --rm hello-world` |
| List images | `docker image ls` |
| List running containers | `docker ps` |
| List all containers | `docker ps -a` |
| Pull image | `docker pull nginx:alpine` |
| Build image from Dockerfile | `docker build -t cs19-hello-api:1.0 .` |
| Run image on localhost | `docker run -d --name cs19-api -p 127.0.0.1:8080:8080 cs19-hello-api:1.0` |
| Read logs | `docker logs cs19-api` |
| Remove example container | `docker rm -f cs19-api` |
| Validate Compose without printing config | `docker compose config --quiet` |
| Build and start Compose | `docker compose up --build -d` |
| List Compose services | `docker compose ps` |
| Stop Compose app | `docker compose down` |
| List volumes / networks | `docker volume ls` / `docker network ls` |

## 14. Practice: five short challenges

1. **Concepts:** explain the difference between `Dockerfile`, **image** and **container** in three sentences.
2. **Port mapping:** run Nginx on host port `8090` without exposing it to your network; verify in a browser.
3. **Build:** run the lab's nine tests, build the lab image and visit `/health` through port 8080.
4. **Persistence:** create a named Docker volume, write a file from one temporary container and read it from another.
5. **Code review:** point to the two lowercase source Dockerfiles and explain why the source Compose file needs adjustment. Identify where MongoDB would need to connect.

Expected evidence: `docker version`, your Nginx response, a JSON `{ "status": "ok" }` from the lab, the persistent `hello` output, and a short note explaining the Chapter 05 configuration gaps.

## 15. Reuse checklist

- [ ] I verified Windows, virtualization, WSL 2, Docker Desktop and the engine.
- [ ] I know the difference between an image, container, volume and network.
- [ ] I can run `hello-world` and manage a demo Nginx container.
- [ ] I can explain `-p HOST_PORT:CONTAINER_PORT`.
- [ ] I can read `FROM`, `WORKDIR`, `COPY`, `RUN`, `ENV`, `USER`, `EXPOSE` and `CMD`.
- [ ] I can build the lab image and reach the lab's health endpoint.
- [ ] I know why an app must listen on `0.0.0.0` inside a container.
- [ ] I can validate and run a small Compose file.
- [ ] I understand why a database needs persistent storage.
- [ ] I can distinguish Git-tracked `.env` risks from Docker build-context ignores.
- [ ] I documented which source-project issues remain uncorrected and untested.
- [ ] I can clean up only the resources created for this exercise.

## 16. Reference links and the next cheatsheet

**Source repository:** [From Zero to Fullstack — Chapter 05 DevOps](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops). Source files directly inspected for this guide: `README.md`, `compose.yaml`, `dockerfile`, `backend/dockerfile`, `.dockerignore`, `backend/.dockerignore`, `package.json`, `backend/package.json` and `.github/workflows/ci-cd.yaml`. The repository's `.env` content was **not** retrieved or reproduced.

**Official Docker references:**

- [Docker overview: images, containers and the engine](https://docs.docker.com/get-started/docker-overview/)
- [Docker Desktop installation on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker Desktop WSL 2 backend](https://docs.docker.com/desktop/features/wsl/)
- [Docker Desktop licensing](https://docs.docker.com/subscription-billing/desktop-license/)
- [Dockerfile fundamentals](https://docs.docker.com/build/concepts/dockerfile/)
- [Docker Compose build specification](https://docs.docker.com/reference/compose-file/build/)
- [Compose `version` element (obsolete)](https://docs.docker.com/reference/compose-file/version-and-name/)
- [Docker official getting-started lab](https://docs.docker.com/guides/lab-container-getting-started/)

**Evidence summary:** The independent lab's **nine Node.js tests** passed, including seven HTTP/protocol tests and two static file checks. Its Compose file was parsed and checked with a local YAML parser. **No Docker image/container was built or run** during preparation; Docker's Windows installation and the original project's multi-container stack require hands-on validation. The source repo was only read, not changed.

**Next:** **Cheatsheet 20 — Dockerizing a MERN Fullstack Application** will connect the React frontend, Express backend and MongoDB with a correct multi-container Compose configuration, networks and durable data.

---

*Learn → Practice → Document → Reuse.*