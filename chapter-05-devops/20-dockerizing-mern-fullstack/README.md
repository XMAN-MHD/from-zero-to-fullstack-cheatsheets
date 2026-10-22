# Cheatsheet 20 — Dockerizing a MERN Fullstack Application

**From Zero to Fullstack · Chapter 05: DevOps**  
**Audience:** Beginners moving from locally running React + Express to multi-container applications  
**Primary platform:** Windows 10/11 · PowerShell · VS Code · Docker Desktop with WSL 2  
**Version:** 1.0 · 22 October 2026  
**Source project:** [Chapter 5 — DevOps](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops)  
**Method:** **Learn → Practice → Document → Reuse**

> **What was verified?** The source repository's actual `compose.yaml`, frontend `dockerfile`, backend `backend/dockerfile`, JavaScript API entry points and dependency manifests were inspected. An **independent MERN laboratory** accompanies this document in the ZIP; **8 Node.js unit tests passed** and the lab's Compose YAML passed static checks. This authoring environment has no Docker engine; `docker compose build`, container startup, actual MongoDB persistence and browser integration **were not executed**. Run the indicated Docker checks on your Windows workstation. A *SOURCE PROJECT* statement describes existing code; an *IMPROVEMENT* or *LAB* is proposed educational code, not a change committed to Chapter 5.

## 1. Learning outcomes and the final result

By the end of this cheatsheet, you should be able to:

1. Describe **MERN**: MongoDB, Express, React and Node.js.
2. Explain an **image**, a **container**, a **Dockerfile**, **Docker Compose**, a **volume** and a **network**.
3. Start Docker Desktop and inspect the engine from PowerShell.
4. Build frontend and backend images separately using sensible Dockerfiles.
5. Run React, Express and MongoDB together using one Compose file.
6. Tell the difference between `localhost` inside a container, Docker service names and `localhost` in a web browser.
7. Configure both server-side and **Vite build-time** variables correctly.
8. Make data survive a container restart by using a named volume.
9. Use health checks, `depends_on`, logs, exec and Compose validation.
10. Diagnose documented problems in the real Chapter 5 source project.
11. Run a complete independent three-service MERN learning lab.
12. Reuse the setup responsibly, knowing which security and production concerns remain unresolved.

### Architecture: three services, one application

```text
Windows browser   http://localhost:3000
       |
       | HTTP / and /api/v1/...
       v
  [web] Nginx :80  ---- serves compiled React/Vite files
       |
       | /api/ requests are reverse-proxied to http://api:3001
       v
  [api] Node.js + Express + Mongoose :3001
       |
       | mongodb://db:27017/blog  (Docker internal DNS)
       v
  [db] MongoDB :27017
       |
       v
  [mongo_data] named volume — persists when containers are recreated
```

In the lab, **only the web service exposes a host port**. Nginx and the API share one network; the API and MongoDB share a second, **internal** network. The database is not exposed to the host. The *source project* has only the frontend and backend defined in Compose; the database service is an improvement explained below.

## 2. Prerequisites: start from zero on Windows

You need a recent Windows 10/11 PC with virtualization enabled, a 64-bit processor, enough disk space for images and a suitable Docker Desktop license for your use. The exact supported Windows edition and WSL requirements may evolve, so check the official Docker installation documentation before installing.

1. Install **Git** and **Visual Studio Code** if not already available.
2. Install a supported current **Node.js** version if you also want to run React or Express outside Docker.
3. Install/enable **WSL 2** following Microsoft's instructions. In an elevated PowerShell terminal, on supported systems:

```powershell
wsl --install
wsl --update
wsl --status
```

4. Restart Windows if requested, install Docker Desktop from the official Docker website and select the **WSL 2 backend** where available.
5. Start Docker Desktop; wait for the engine to be ready. Reopen PowerShell, then run:

```powershell
docker version
docker info
docker compose version
docker run --rm hello-world
```

**Expected:** both Docker client and server information, a Compose version, and a successful greeting from `hello-world`. If you only see *Cannot connect to the Docker daemon*, start Docker Desktop, check WSL integration and repeat. Do not proceed with image builds until the engine is running.

**PowerShell note:** Commands here use `docker compose` (Compose v2), not the older `docker-compose` binary. Use separate terminal tabs when the instructions say to keep a process running.

## 3. Understand the two applications in the repository

The *SOURCE PROJECT* repository is:

`https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops`

Inspect its files before editing:

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops.git
cd from-zero-to-fullstack-ch5-devops
code .
Get-Content .\compose.yaml
Get-Content .\dockerfile
Get-Content .\backend\dockerfile
```

The actual relevant layout is:

```text
from-zero-to-fullstack-ch5-devops/
├── compose.yaml
├── dockerfile              # SOURCE: lowercase spelling
├── .dockerignore
├── package.json            # React / Vite
├── package-lock.json
├── src/
│   ├── Blog.jsx
│   └── api/posts.js
├── backend/
│   ├── dockerfile          # SOURCE: lowercase spelling
│   ├── .dockerignore
│   ├── package.json
│   ├── package-lock.json
│   └── src/
│       ├── index.js
│       ├── app.js
│       └── db/init.js
└── .github/workflows/ci-cd.yaml
```

**Source behavior:** The frontend Dockerfile builds Vite files using Node 20, then serves `dist/` with Nginx. The backend Dockerfile installs its Node dependencies and runs `npm start`. The existing Compose file defines services `blog-backend` and `blog-frontend`; it exposes host ports `3001` and `3000` respectively, but defines **no MongoDB service**. The backend obtains its `DATABASE_URL` from Compose variable substitution.

### Existing configuration: what is present and what is missing

| Part | Observed in Chapter 5 | What to verify or improve |
| :-- | :-- | :-- |
| Frontend image | Multi-stage `dockerfile`, Node 20 → Nginx | Declare the nondefault filename or rename it; build reliably |
| Backend image | `backend/dockerfile` uses Node 20 | Declare its filename; prefer reproducible installs |
| Compose | `blog-backend`, `blog-frontend` | Add MongoDB or supply a reachable external database |
| Build argument | `VITE_BACKEND_URL` supplied to frontend build | Set to an address the **browser** can reach |
| Database URL | `${DATABASE_URL}` injected into backend | Use Docker DNS (`blog-db`) for an internal MongoDB service |
| Persistence | No MongoDB volume because no MongoDB service | Add a named volume |
| Networking | Implicit default Compose network | Understand service DNS; optionally separate data network |
| Browser API | `src/api/posts.js` uses `fetch()` | Check `res.ok`; handle errors and 204 appropriately |

This is an inventory, **not** evidence that the two-container source deployment currently runs successfully.

## 4. Dockerfile basics: build and runtime are different stages

A Dockerfile is a sequence of instructions to **build an image**. A container is an **instance of an image**. A Node server needs a running Node process; a compiled React/Vite frontend normally needs only a static web server such as Nginx.

### Frontend: development compilation → Nginx

The source `dockerfile` has two `FROM` stages. The first runs `npm run build` and creates `dist/`; the second copies those files into Nginx's web root. This keeps the final image free of the Vite development server and most Node build tools.

A cleaned-up example for a repository that has already generated and committed `package-lock.json`:

```dockerfile
# Improvement for root Dockerfile; not committed to the source project
FROM node:22-alpine AS build
WORKDIR /app
ENV HUSKY=0
ARG VITE_BACKEND_URL=http://localhost:3001/api/v1
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

**Why `HUSKY=0`?** The source frontend's `package.json` has a Husky `prepare` script, but `.git` is excluded from its Docker build context. Disabling Git-hook installation during the image build avoids unnecessary hook setup. `npm ci` requires a matching committed lockfile; it is appropriate for a reproducible real project.

### Backend: keep Node.js at runtime

```dockerfile
# Improvement for backend/Dockerfile; requires committed backend/package-lock.json
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY src ./src
ENV NODE_ENV=production
USER node
EXPOSE 3001
CMD ["node", "src/index.js"]
```

The lab ships analogous Dockerfiles but intentionally uses `npm install`: its teaching skeleton does not include generated lockfiles. **After local initialization**, generate and commit lockfiles and replace `npm install` with `npm ci` before adopting the pattern in a real codebase.

### `dockerfile` versus `Dockerfile`: a real source issue

Docker Compose's normal `build: ./backend` convention looks for **`Dockerfile`** (capital D) in that context. The Chapter 5 source instead has `backend/dockerfile`, and the root is also `dockerfile`. On case-sensitive builds this causes a build failure unless explicitly configured. Two safe options:

```yaml
# Option A: preserve the source filenames (least invasive)
services:
  blog-backend:
    build:
      context: ./backend
      dockerfile: dockerfile
  blog-frontend:
    build:
      context: .
      dockerfile: dockerfile
```

Or rename both source files to `Dockerfile` and use the default lookup. **Do not apply both approaches at once.**

## 5. Docker Compose: services, dependencies and YAML

Compose coordinates multiple containers from `compose.yaml`. In a service entry:

- `build.context` is the folder sent to the Docker build.
- `build.dockerfile` selects a nondefault Dockerfile.
- `environment` passes runtime values to the container.
- `build.args` passes build-time values to the Dockerfile.
- `ports: "3000:80"` means **host port 3000 → container port 80**.
- `depends_on` arranges startup ordering; with `condition: service_healthy`, it waits for the dependency's health check.
- `networks` determine which containers can talk to each other.
- `volumes` attach storage that can survive container recreation.

**Compose version note:** New Compose v2 files generally do not need a top-level `version:` key. The source includes `version: "3.9"`, which may trigger an informational warning in newer Compose versions. The lab omits it.

## 6. Minimal correction path for the existing Chapter 5 project

**IMPROVEMENT — proposed local replacement for the source `compose.yaml`:** This keeps the frontend's browser-based HTTP URL and the source service names while adding a local MongoDB container. It also explicitly selects the existing lowercase Dockerfile names.

```yaml
services:
  blog-db:
    image: mongo:7
    restart: unless-stopped
    volumes:
      - blog_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s

  blog-backend:
    build:
      context: ./backend
      dockerfile: dockerfile
    environment:
      PORT: "3001"
      DATABASE_URL: mongodb://blog-db:27017/blog
    depends_on:
      blog-db:
        condition: service_healthy
    ports:
      - "127.0.0.1:3001:3001"

  blog-frontend:
    build:
      context: .
      dockerfile: dockerfile
      args:
        VITE_BACKEND_URL: http://localhost:3001/api/v1
    ports:
      - "127.0.0.1:3000:80"
    depends_on:
      - blog-backend

volumes:
  blog_data: {}
```

**Local browser reasoning:** Frontend JavaScript runs in the visitor's browser, so `http://localhost:3001/api/v1` works here only because port 3001 is published on **that same PC**. `http://blog-backend:3001` would work from **another container** on the Compose network, but **not from the browser**. The independent lab instead uses an Nginx reverse proxy and a relative browser URL (`/api/v1`), which is a better deployment pattern.

### Verify the URL through three perspectives

| Caller | Correct destination in this local setup | Explanation |
| :-- | :-- | :-- |
| MongoDB client inside backend | `mongodb://blog-db:27017/blog` | Docker service DNS and container port |
| Web browser on the Windows PC | `http://localhost:3001/api/v1/posts` | Published host port 3001 |
| Web browser for React | `http://localhost:3000` | Nginx host port 3000 |

If the backend database lives elsewhere (for example, a managed cloud MongoDB instance), the `blog-db` service is optional; replace the URL carefully and apply proper secrets management and network controls.

## 7. Database connection review: a second source issue

**SOURCE PROJECT:** `backend/src/db/init.js` calls Mongoose with these options:

```js
await mongoose.connect(DATABASE_URL, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
})
```

These options come from older driver configurations and should **not** be carried into a Mongoose 8 / modern MongoDB driver setup. A simple **IMPROVEMENT** is:

```js
import mongoose from 'mongoose'

export async function initDatabase() {
  const databaseUrl = process.env.DATABASE_URL
  if (!databaseUrl) throw new Error('DATABASE_URL is missing')
  await mongoose.connect(databaseUrl)
  console.info('Connected to MongoDB') // Do not log a credential-bearing URL.
}
```

The existing application calls `await initDatabase()` before `app.listen`. That is desirable because the server should not accept normal API traffic until its database connection has succeeded. Check your exact installed Mongoose version and run tests after making this local change.

**Security observation:** The repository tree contains a tracked root `.env` file. We have **not** copied or displayed it. Before sharing or deploying the repository, audit it yourself. If live secrets were committed, remove them appropriately from active use and **rotate** them; adding `.env` to `.gitignore` alone does not erase historical commits.

## 8. Build-time versus runtime environment variables

Vite replaces `import.meta.env.VITE_*` values **when compiling the frontend**. Changing a runtime variable in a finished Nginx container does **not** normally change an already-generated JavaScript bundle.

```text
Frontend build:
  docker compose build web
    -> ARG VITE_BACKEND_URL
    -> npm run build
    -> compiled JavaScript saved in dist/
    -> Nginx serves that fixed JavaScript

Backend runtime:
  docker compose up api
    -> DATABASE_URL and PORT passed to Node process
    -> mongoose.connect(process.env.DATABASE_URL)
```

For the source project, `ARG VITE_BACKEND_URL=...` appears **inside its build stage**, making it available to commands such as `RUN npm run build`. With a browser-reachable public URL, a rebuild is normally required when the value changes.

**Never put secrets into `VITE_*` variables or Docker build arguments.** Public frontend bundles are readable by visitors, and build arguments can also be exposed through metadata or history. Use only nonsecret public configuration for frontend URLs.

## 9. Docker networking, ports and reverse proxies

Docker Compose supplies service-name DNS on shared networks. A backend connecting to MongoDB should use `db` or `blog-db`, **not** `localhost`, because `localhost` within a container refers to that same container. Container IP addresses are temporary; use service names instead.

In a basic one-network Compose stack, all three services can see each other. A slightly safer topology uses two networks:

```yaml
networks:
  public_net: {}
  database_net:
    internal: true
```

Assign `web` and `api` to `public_net`; assign `api` and `db` to `database_net`. Only the API can cross between the two. The lab includes this design and keeps MongoDB's `27017` port unpublished.

**Reverse proxy pattern:** The lab's React code calls relative `/api/v1/posts`. Nginx receives that request on port 80 and forwards it to `http://api:3001` using Docker DNS. The user's browser never needs to resolve a Docker service name or connect to a second origin, making local CORS setup unnecessary.

## 10. MongoDB persistence: containers are disposable, data is not

MongoDB stores its database data inside `/data/db`. If that path is backed only by a container writable layer, the data can disappear when the container is recreated. Map it to a **named volume**:

```yaml
services:
  db:
    image: mongo:7
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data: {}
```

A named volume lives independently from the container and is normally preserved by `docker compose down` and subsequent `docker compose up` commands. **`docker compose down -v` deletes the Compose-managed named volumes and can destroy your local database contents.** Always make a backup first if the data matter.

Typical checks:

```powershell
docker volume ls
docker compose ps
docker compose down
docker compose up -d
```

Create a post before stopping the stack, then verify that it still exists after restarting. That is a **local exercise**, not a persistence claim already verified in this authoring environment.

## 11. An independent MERN lab: files and responsibilities

**LAB — bundled in the downloadable ZIP.** This is a **new teaching example**, not the exact contents of the original Chapter 5 repository. It contains three separately buildable services, a React CRUD page, a Mongoose-powered Express API, MongoDB with persistence, and a small static verification suite.

```text
lab/
├── README.md
├── compose.yaml
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   ├── src/
│   │   ├── index.js         # Express, Mongoose, CRUD endpoints
│   │   └── utils.js         # Pure reusable helpers
│   └── tests/utils.test.js  # No Docker or MongoDB required
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── nginx.conf           # Proxies /api/ to the backend
│   ├── package.json
│   ├── index.html
│   └── src/main.jsx         # React UI + TanStack Query
└── checks/
    └── check_compose.py    # Static YAML invariants
```

The independent lab API supports:

| HTTP method | Endpoint | Example result |
| :-- | :-- | :-- |
| GET | `/api/v1/health` | `200` with `{"status":"ok"}` |
| GET | `/api/v1/posts` | `200` with an array, possibly empty |
| POST | `/api/v1/posts` | `201` and created document |
| PATCH | `/api/v1/posts/:id` | `200` updated; `400` invalid ID; `404` missing |
| DELETE | `/api/v1/posts/:id` | `204` empty body; `400` invalid ID; `404` missing |

The frontend renders posts, allows creating and renaming them, and prompts for confirmation before deletion. The lab supports combined author/tag filtering in the API helper, although its minimal UI intentionally focuses on CRUD operations. More advanced filtering is covered in Cheatsheet 18.

## 12. Run the lab on Windows (step by step)

1. Extract the ZIP file for this cheatsheet. Open a PowerShell terminal inside its **`lab`** folder (the folder containing `compose.yaml`).
2. Verify Docker Desktop is running.
3. Validate the Compose configuration before building images:

```powershell
docker compose version
docker compose config
docker compose build
docker compose up -d
docker compose ps
```

4. Inspect startup messages:

```powershell
docker compose logs --tail=80 db
docker compose logs --tail=80 api
docker compose logs --tail=80 web
```

5. Open **http://localhost:3000**. You should see *MERN Docker Learning Lab*. Create a post in the form, edit its title, delete it, and inspect the UI after each operation.
6. Visit **http://localhost:3000/api/v1/health**. Expect an HTTP 200 response with `{"status":"ok"}`.
7. Query the API directly from PowerShell:

```powershell
Invoke-RestMethod 'http://localhost:3000/api/v1/health'
Invoke-RestMethod 'http://localhost:3000/api/v1/posts'

$body = @{ title = 'My first Docker post'; author = 'Student'; tags = @('docker','mern') } | ConvertTo-Json
$created = Invoke-RestMethod -Method Post `
  -Uri 'http://localhost:3000/api/v1/posts' `
  -ContentType 'application/json' -Body $body
$created._id

$patch = @{ title = 'Updated from PowerShell' } | ConvertTo-Json
Invoke-RestMethod -Method Patch `
  -Uri "http://localhost:3000/api/v1/posts/$($created._id)" `
  -ContentType 'application/json' -Body $patch
```

8. Try removing the created post (it should return **204 No Content**):

```powershell
Invoke-WebRequest -Method Delete -Uri "http://localhost:3000/api/v1/posts/$($created._id)"
```

9. Repeat with a post you **do not delete**, then run `docker compose down` and `docker compose up -d` and check that the post still exists. This verifies volume persistence on *your* machine.
10. Stop without deleting data:

```powershell
docker compose down
```

**First build time:** Docker may need to download Node, Nginx and MongoDB base images and npm dependencies. Internet access is needed for that first run. A valid configuration does not guarantee the download or deployment will succeed; inspect the actual command output.

## 13. Inspect the lab's important files

### 13.1 `compose.yaml`: three services

```yaml
name: mern-learning-lab
services:
  web:
    build: ./frontend
    ports: ["127.0.0.1:3000:80"]
    depends_on: [api]
    networks: [public_net]
  api:
    build: ./backend
    environment:
      PORT: "3001"
      DATABASE_URL: mongodb://db:27017/blog
    depends_on:
      db:
        condition: service_healthy
    networks: [public_net, database_net]
  db:
    image: mongo:7
    volumes: ["mongo_data:/data/db"]
    networks: [database_net]
networks:
  public_net: {}
  database_net:
    internal: true
volumes:
  mongo_data: {}
```

This is a **condensed explanatory excerpt**, not a byte-for-byte copy of the lab file. The ZIP's actual `compose.yaml` also contains the database health check, explicit build locations, runtime mode and API exposure.

### 13.2 Nginx reverse proxy

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  index index.html;

  location /api/ {
    proxy_pass http://api:3001;
  }
  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

The important detail is **no extra URI segment in `proxy_pass`** here, so `/api/v1/posts` reaches Express with that path preserved. The lab's `nginx.conf` is the working file to inspect.

### 13.3 Node.js + Mongoose connection

```js
const databaseUrl = process.env.DATABASE_URL
if (!databaseUrl) throw new Error('DATABASE_URL is required')
await mongoose.connect(databaseUrl)
// Only after connecting, start the Express HTTP server.
```

MongoDB is reached by the service name `db` via the data network. The lab does not print the connection URL in its logs, a sensible default because real connection strings may contain credentials.

### 13.4 React sends same-origin requests

```js
const API = '/api/v1/posts'
const response = await fetch(API)
if (!response.ok) throw new Error(`HTTP ${response.status}`)
const posts = await response.json()
```

This avoids embedding a development-only address into the compiled frontend bundle. Mutations update the server and invalidate `['posts']` in TanStack Query so the list can refresh.

## 14. Smoke tests, static checks and evidence

Three different verification levels have different meanings:

**Level A — executable without Docker (completed during authoring):**

```powershell
cd lab\backend
node --test tests\utils.test.js
```

Eight Node.js tests cover CSV-to-array tags, allowed updates and safe sorting. **Result during authoring: 8 passed, 0 failed.** These are tests of pure JavaScript functions, **not** tests of a live MongoDB database.

**Level B — static Compose configuration checks (completed during authoring):**

```powershell
cd ..
python checks\check_compose.py
```

This checks the parsed YAML structure, exactly three service definitions, volume mapping, restricted database host exposure, service DNS URL, build paths and Nginx proxy target. It requires **Python 3 and PyYAML**. **Result during authoring: passed.** This check does not start containers or prove MongoDB connectivity.

**Level C — real Docker and browser validation (must be run locally):**

```powershell
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs --tail=100 api
Invoke-RestMethod 'http://localhost:3000/api/v1/health'
```

Then use the actual React page for CRUD and restart the stack to check persistence. **These steps were not executed during authoring** because Docker Engine is unavailable in the preparation environment.

## 15. Troubleshooting table

| Symptom | Common cause | Concrete diagnostic or fix |
| :-- | :-- | :-- |
| `Cannot connect to Docker daemon` | Docker Desktop not started | Open Docker Desktop; check WSL integration; retry `docker version` |
| `failed to read dockerfile` | Source filenames are lowercase | Use `build.dockerfile: dockerfile` or rename to `Dockerfile` |
| API connects to wrong Mongo host | `localhost` used inside the API container | Use `mongodb://db:27017/blog` (or `blog-db` for source) |
| MongoDB refuses initial connection | Container not ready | Check `docker compose logs db`; use a health check and `depends_on` |
| React page works, API fails | Wrong browser URL or Nginx route | Inspect Network tab and `docker compose logs web api` |
| `ERR_NAME_NOT_RESOLVED` for `api` in browser | Browser attempted internal Docker hostname | Use an exposed host port or same-origin Nginx proxy |
| CORS policy blocks request | Browser and API use different origins | For local source verify Express `cors()` and API origin; proxy preferred |
| Vite API URL stays stale | It was compiled into the JS bundle | Set `ARG` correctly and rebuild the frontend image |
| `Module not found: prop-types` in source | Root frontend `package.json` does not declare dependency | Run `npm install prop-types` in source and commit lockfile changes |
| `DeletePostButton is not defined` | Existing `DeletePost.jsx` references a nonexistent variable | Change the `propTypes` assignment to `DeletePost.propTypes` |
| Mongo connect options warning/error | Old driver options in `backend/src/db/init.js` | Use `await mongoose.connect(DATABASE_URL)` with current Mongoose |
| Data vanish after shutdown | No volume or Compose run with `down -v` | Add named volume; use `down` without `-v`; keep backups |
| Port 3000 already in use | Another service listening | Find/stop conflicting process or change host-side port |
| Image build fails fetching npm packages | Network, registry or missing lock dependencies | Inspect logs, ensure Internet, use `npm ci` with a valid lockfile |
| Data database secrets appear in logs | Printing full connection string | Remove credentials from logs, rotate exposed secrets |

### A helpful diagnostic sequence

```powershell
docker compose config
docker compose ps
docker compose logs --tail=100
docker compose logs --tail=100 api
docker compose exec api node --version
docker image ls
docker volume ls
```

Remember that `docker compose config` validates and expands the Compose model, **not** Dockerfile syntax or runtime functionality.

## 16. Security and operational best practices

A successful local Compose demo is not automatically production-ready. Before deploying, account for:

1. **Credentials:** Never hard-code MongoDB passwords in frontend bundles or publish `.env` files. Audit version control and rotate exposed credentials. Use suitable production secrets management.
2. **Network boundaries:** Do not publish MongoDB's port to the public Internet. Keep private data services on a private network.
3. **Authentication and authorization:** The learning API intentionally has **no login or roles**. It must not be exposed to the Internet as-is.
4. **Input validation:** Use strict validation, allowlisted update fields, pagination and limits; avoid logging private request payloads.
5. **Base images:** Use supported, maintained image tags and pin image digests for production reproducibility. Rebuild when security updates arrive.
6. **Least privilege:** Prefer nonroot application processes and minimized runtime images. Nginx's default container image and port may need additional hardening for strict security environments.
7. **Lockfiles:** Commit frontend/backend lockfiles and use `npm ci` when transitioning from the lab to a maintained project.
8. **Backups:** A named volume is *persistence*, **not** a backup. Test restoring MongoDB data from independent backups.
9. **HTTPS and domains:** Use real hostnames, TLS, an ingress/reverse proxy and correct public API URLs for deployed applications. A browser's `localhost` refers to the browser user's machine.
10. **Monitoring:** Add structured logging, health/readiness checks, resource limits and alerts before operating at scale.
11. **Deployment hygiene:** Do not use production credentials in developer PCs or in disposable demonstration examples.
12. **Do not overclaim:** A static Compose check does not demonstrate that three services are healthy or that persistence survived restart.

## 17. Common misconceptions: quick corrections

| Misconception | Correct concept |
| :-- | :-- |
| “A Docker image is a running process.” | An image is a template; a container is an instance. |
| “All `localhost` URLs point to my PC.” | Each container has its own localhost; the browser has its own environment. |
| “`depends_on` alone guarantees database readiness.” | Plain `depends_on` orders startup; a service-health condition can wait for a health check. |
| “Docker Compose automatically adds MongoDB to MERN.” | Only services declared in the Compose file are created. |
| “Volumes are backups.” | Volumes persist local data; backups require separate copies and restore tests. |
| “Changing Vite env values restarts the frontend config.” | Vite public variables are usually baked into the build; rebuild to change them. |
| “Exposing MongoDB is required for the API.” | Backend containers communicate with MongoDB via the internal network and service name. |
| “The source Chapter 5 Compose file is already production-ready.” | It has no MongoDB service and requires verification and security review. |

## 18. Quick reference: Docker and Compose commands

| Purpose | Command |
| :-- | :-- |
| Check engine | `docker version` |
| Check Compose version | `docker compose version` |
| Validate Compose file | `docker compose config` |
| Build images | `docker compose build` |
| Build and start | `docker compose up --build -d` |
| Show service state | `docker compose ps` |
| Follow service logs | `docker compose logs -f api` |
| Run a command in container | `docker compose exec api node --version` |
| Restart service | `docker compose restart api` |
| Recreate stack with changes | `docker compose up -d --build` |
| Stop and remove containers | `docker compose down` |
| Remove containers **and volumes** | `docker compose down -v` **DANGER: data loss** |
| List named volumes | `docker volume ls` |
| List images | `docker image ls` |
| Remove unused dangling objects | `docker system prune` (review prompt before confirming) |

**Important:** Build commands require a live Docker engine. The Docker CLI and Python static checks are not substitutes for running the stack.

## 19. Reuse checklist for the next MERN project

- [ ] I can start Docker Desktop and run `docker version` and `docker compose version`.
- [ ] I understand the difference between the image build and the running container.
- [ ] I know where each Dockerfile lives and how Compose selects it.
- [ ] I have checked the frontend *build-time* API URL or implemented a same-origin proxy.
- [ ] I can explain which `localhost` is used by the browser, frontend container and API container.
- [ ] My database hostname refers to the proper Compose service.
- [ ] My MongoDB data directory is backed by a named volume.
- [ ] My database is not unnecessarily published on a host port.
- [ ] I can run `docker compose config`, `build`, `up`, `ps`, `logs` and `down`.
- [ ] I have tested creating, editing and deleting a record in a running stack.
- [ ] I have verified a record survives `down` then `up` without `-v`.
- [ ] I have not committed real environment secrets or exposed keys through Vite variables.
- [ ] I use lockfiles, maintained base images and documented security controls for maintained systems.
- [ ] I distinguish **source project**, **lab exercise**, **recommendation**, and **tested result**.

## 20. References and what comes next

**Primary code reference:** [XMAN-MHD/from-zero-to-fullstack-ch5-devops](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops), particularly `compose.yaml`, `dockerfile`, `backend/dockerfile`, `backend/src/db/init.js`, `src/api/posts.js` and the two `package.json` files. Source inspection performed on **22 October 2026**.

**Official technical references:**

- Docker Desktop installation for Windows: https://docs.docker.com/desktop/setup/install/windows-install/
- Compose networking and service DNS: https://docs.docker.com/compose/how-tos/networking/
- Compose build specification: https://docs.docker.com/reference/compose-file/build/
- Compose services and health checks: https://docs.docker.com/reference/compose-file/services/
- Compose volumes: https://docs.docker.com/reference/compose-file/volumes/
- Dockerfile overview: https://docs.docker.com/build/concepts/dockerfile/
- Docker build variables (`ARG` vs `ENV`): https://docs.docker.com/build/building/variables/
- Docker security practices: https://docs.docker.com/build/building/best-practices/
- Vite environment variables: https://vite.dev/guide/env-and-mode
- Mongoose connections: https://mongoosejs.com/docs/connections.html

**Next:** **Cheatsheet 21 — GitHub Actions CI from Scratch.** We will test, lint and build our JavaScript projects automatically after pushes and pull requests. This is separate from automatically deploying to Render, which is covered in Cheatsheet 22.

---

*Learn → Practice → Document → Reuse.*