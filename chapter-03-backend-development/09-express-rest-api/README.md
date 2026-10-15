# Cheatsheet 09 — Express.js REST API from Scratch

**From Zero to Fullstack · Chapter 3: Backend Development**  
**Audience:** Absolute beginners and developers building reusable Node.js APIs  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Documentation version:** 1.0 · 15 October 2026  
**Real source project:** [Chapter 3 — Backend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install Express, understand HTTP requests and responses, build and test an independent JSON REST API, then run and understand the actual Chapter 3 Blog backend. This guide focuses on **Express and API fundamentals**; Mongoose, complete CRUD architecture, and Jest are covered in Cheatsheets 10–13.

## Quick reference — save for your next project

| Task | Command or concept |
| --- | --- |
| Check prerequisites | `node --version`, `npm.cmd --version`, `git --version` |
| Create Node.js project | `npm init -y` |
| Install Express 5 | `npm install express@^5` |
| Add development auto-restart | `npm install --save-dev nodemon` |
| Start an existing Chapter 3 project | `npm.cmd ci`; then `npm.cmd run dev` |
| Run the lab server | `npm.cmd run dev` (after adding the script) |
| Parse JSON request bodies | `app.use(express.json())` |
| Read route parameters | `req.params.id` |
| Read query parameters | `req.query.author` |
| Read JSON body | `req.body` (validate it first) |
| Send JSON with status | `res.status(201).json({ ... })` |
| API base path from Chapter 3 | `/api/v1/posts` |
| Call an endpoint on Windows | `Invoke-RestMethod -Uri http://localhost:3000/api/v1/posts` |
| Stop the server | `Ctrl + C` |

**Source accuracy:** The Chapter 3 **source files are at the repository root** (`src/`, `package.json`, etc.). The older Chapter 3 README describes a `backend/` folder, but that folder is **not present** in the current GitHub tree. Run commands from the repository **root**, not from `backend/`.

## 1. Why Express? From Chapter 2 to Chapter 3

In Chapter 2, we built a server with Node.js's built-in `createServer()` from `http`. We were responsible for checking paths/methods, selecting status codes, and creating responses. **Express** is a Node.js web framework that makes those tasks easier with **routing, middleware, request helpers, and response helpers**.

A **REST-style API** organizes resources under meaningful URLs and uses HTTP methods: `GET` reads, `POST` creates, `PATCH` partially updates, and `DELETE` removes. **JSON** is the data format our Blog API exchanges with its clients. Express does not install MongoDB or store data for you; it handles HTTP, while services and a database handle business logic and persistence.

```text
Client (browser / PowerShell / Postman)
                 |
       HTTP method + URL + JSON
                 |
                 v
          Express app (app.js)
        JSON parsing + routing
                 |
                 v
        routes/posts.js (HTTP)
                 |
                 v
       services/posts.js (logic)
                 |
          [actual project]
                 v
       Mongoose model -> MongoDB
                 |
       status code + JSON response
                 v
               Client
```

**Terminology:**

- **Endpoint:** A method + path combination, such as `GET /api/v1/posts`.
- **Route handler:** The JavaScript function called for a matching endpoint.
- **Middleware:** A function in the request pipeline (for example `express.json()`).
- **Request (`req`):** Path, headers, parameters, query string, and body from the client.
- **Response (`res`):** What the server sends back (status, headers, and JSON).
- **Status code:** An HTTP number describing the outcome.
- **REST:** A design style; simply using Express does not guarantee good REST design.

## 2. Prerequisites and installation from scratch

**Required knowledge:** Basic JavaScript variables, functions, objects, arrays, `async/await`, and JSON. Read [Cheatsheet 01](../../chapter-01-foundations/01-developer-environment/README.md) if this is your first development environment and [Cheatsheet 06](../../chapter-02-backend-fundamentals/06-node-http/README.md) for HTTP fundamentals.

1. Install [VS Code](https://code.visualstudio.com/download), [Git for Windows](https://git-scm.com/install/windows), and a supported [Node.js LTS](https://nodejs.org/en/download) through their **official installers**.
2. Restart PowerShell after installation to refresh your `PATH`.
3. Verify the commands below; each should return a version number:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

The original source declares **Express `^5.1.0`** and uses ES modules (`"type": "module"`). Express 5 requires Node.js 18 or later; a supported LTS release such as Node.js 22 or 24 is a good choice. Use `npm.cmd` on Windows when PowerShell cannot execute `npm.ps1` under its script policy. **Do not change machine-wide execution policy just for this guide.**

This cheatsheet offers two paths:

- **Path A — Beginner lab:** Create a new Express API that stores sample posts **in memory**, so **MongoDB is not required**. The data resets when the process restarts. This is an original learning exercise, **not copied from the Chapter 3 GitHub repository**.
- **Path B — Real Chapter 3:** Clone and run our original Express + Mongoose + MongoDB backend. MongoDB must already be running, as explained in [Cheatsheet 08](../../chapter-02-backend-fundamentals/08-mongodb-native-driver/README.md).

## 3. Path A — Create an independent Express project

### 3.1 Create the project on Windows

From a normal PowerShell terminal:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
New-Item -ItemType Directory -Force -Path "express-rest-lab"
Set-Location "express-rest-lab"
npm.cmd init -y
npm.cmd pkg set type=module
npm.cmd install express@^5
npm.cmd install --save-dev nodemon
npm.cmd pkg set "scripts.start=node src/index.js"
npm.cmd pkg set "scripts.dev=nodemon src/index.js"
New-Item -ItemType Directory -Force -Path "src\routes", "src\services"
code .
```

**Expected:** `package.json` appears with `"type": "module"`, an `express` dependency, a development dependency `nodemon`, and `start` / `dev` scripts. npm also creates `package-lock.json` and `node_modules/`.

**Recommended `.gitignore`** (create in the project root):

```gitignore
node_modules/
.env
.env.*
!.env.example
```

Never commit `node_modules/`, real `.env` values, passwords, or database credentials. Do commit `package.json` and `package-lock.json`.

### 3.2 Create `src/app.js`

```javascript
import express from 'express'
import { postsRoutes } from './routes/posts.js'

export const app = express()

// Parse requests with Content-Type: application/json.
app.use(express.json({ limit: '100kb' }))

// A simple endpoint used only by this learning lab.
app.get('/health', (req, res) => {
  return res.status(200).json({ status: 'ok' })
})

postsRoutes(app)

// Handle URLs not matched by any route above.
app.use((req, res) => {
  return res.status(404).json({ error: 'Route not found' })
})

// Central handler for unexpected errors (keep implementation details private).
app.use((err, req, res, next) => {
  if (res.headersSent) return next(err)
  console.error('Unexpected server error:', err)
  return res.status(500).json({ error: 'Internal server error' })
})
```

**Explanation:** `express()` makes the app; `express.json()` fills `req.body` for JSON requests; routes are registered **before** the 404 fallback; the error handler takes **four arguments** (`err, req, res, next`). In Express 5, rejected promises from async route handlers are automatically forwarded to error handling, but explicit `try/catch` is also valid when you need a specific response.

### 3.3 Create `src/index.js`

```javascript
import { app } from './app.js'

const PORT = Number(process.env.PORT) || 3000

app.listen(PORT, '127.0.0.1', () => {
  console.info(`Lab server: http://127.0.0.1:${PORT}`)
})
```

**Why two files?** `app.js` configures middleware and routes, whereas `index.js` starts the network listener. This separation makes API testing and later database initialization easier. Binding to `127.0.0.1` limits this beginner lab to the local computer.

### 3.4 Create `src/services/posts.js` (temporary in-memory service)

```javascript
let posts = [
  { id: 1, title: 'Welcome to Express', author: 'Awa', tags: ['node'] },
  { id: 2, title: 'HTTP and JSON', author: 'Moussa', tags: ['http'] },
]
let nextId = 3

export function listPosts({ author, tag } = {}) {
  return posts.filter((post) => {
    if (author && post.author !== author) return false
    if (tag && !post.tags.includes(tag)) return false
    return true
  })
}

export function getPostById(id) {
  return posts.find((post) => post.id === Number(id)) ?? null
}

export function createPost({ title, author = '', tags = [] }) {
  const post = { id: nextId++, title: title.trim(), author, tags }
  posts.push(post)
  return post
}

export function updatePost(id, updates) {
  const post = getPostById(id)
  if (!post) return null
  if (updates.title !== undefined) post.title = updates.title.trim()
  if (updates.author !== undefined) post.author = updates.author
  if (updates.tags !== undefined) post.tags = updates.tags
  return post
}

export function deletePost(id) {
  const before = posts.length
  posts = posts.filter((post) => post.id !== Number(id))
  return posts.length !== before
}
```

This is **only a teaching exercise**: data is lost after restart, there are no database-generated `_id` values or timestamps, and it is **not a replacement for MongoDB**. The production-oriented Mongoose implementation is in the original Chapter 3 repository and in Cheatsheets 10–11.

### 3.5 Create `src/routes/posts.js` (HTTP rules and validation)

```javascript
import {
  listPosts, getPostById, createPost, updatePost, deletePost,
} from '../services/posts.js'

const isNonEmptyString = (value) =>
  typeof value === 'string' && value.trim().length > 0

const validTags = (value) =>
  value === undefined ||
  (Array.isArray(value) && value.every((tag) => typeof tag === 'string'))

export function postsRoutes(app) {
  const base = '/api/v1/posts'

  app.get(base, (req, res) => {
    const { author, tag } = req.query
    if (author && tag) {
      return res.status(400).json({ error: 'Use author OR tag, not both' })
    }
    return res.status(200).json(listPosts({ author, tag }))
  })

  app.get(`${base}/:id`, (req, res) => {
    const post = getPostById(req.params.id)
    if (!post) return res.status(404).json({ error: 'Post not found' })
    return res.status(200).json(post)
  })

  app.post(base, (req, res) => {
    const { title, author, tags } = req.body ?? {}
    if (!isNonEmptyString(title) || !validTags(tags)) {
      return res.status(400).json({ error: 'Invalid title or tags' })
    }
    const post = createPost({ title, author, tags })
    return res.status(201).json(post)
  })

  app.patch(`${base}/:id`, (req, res) => {
    const updates = req.body ?? {}
    if (
      (updates.title !== undefined && !isNonEmptyString(updates.title)) ||
      !validTags(updates.tags)
    ) {
      return res.status(400).json({ error: 'Invalid title or tags' })
    }
    const post = updatePost(req.params.id, updates)
    if (!post) return res.status(404).json({ error: 'Post not found' })
    return res.status(200).json(post)
  })

  app.delete(`${base}/:id`, (req, res) => {
    const deleted = deletePost(req.params.id)
    if (!deleted) return res.status(404).json({ error: 'Post not found' })
    return res.status(204).end()
  })
}
```

**Important security boundary:** This demonstration validates `title` and `tags`, but it is **not a complete request schema**: e.g., the lab does not constrain `author`, limit tags, or validate numeric IDs. Never copy minimal lab validation unmodified into a production API. Add a robust validation schema, authentication/authorization as needed, and rate limiting in real projects.

### 3.6 Start and verify the lab

```powershell
npm.cmd run dev
```

**Expected:** `Lab server: http://127.0.0.1:3000`. Keep this terminal open and open a **second PowerShell terminal**:

```powershell
Invoke-RestMethod -Uri 'http://127.0.0.1:3000/health'
Invoke-RestMethod -Uri 'http://127.0.0.1:3000/api/v1/posts'
Invoke-RestMethod -Uri 'http://127.0.0.1:3000/api/v1/posts?author=Awa'
```

**Expected:** `/health` returns `{ "status": "ok" }`. The list starts with two fictional posts; filtering by `author=Awa` returns one. Restarting the server resets the data.

## 4. HTTP requests and API tests in PowerShell

### 4.1 Create a post — `POST` and JSON body

```powershell
$api = 'http://127.0.0.1:3000/api/v1/posts'
$body = @{
  title = 'Learning Express'
  author = 'Fatou'
  tags = @('express', 'api')
} | ConvertTo-Json

$newPost = Invoke-RestMethod -Uri $api -Method Post `
  -ContentType 'application/json' -Body $body
$newPost
```

**Expected:** Response status **201 Created**; output contains a numeric `id`, a title, author, and tags. Capture its ID for the next commands:

```powershell
$id = $newPost.id
Invoke-RestMethod -Uri "$api/$id" -Method Get
```

### 4.2 Partially update — `PATCH`

```powershell
$patchBody = @{ title = 'Updated Express Guide' } | ConvertTo-Json
Invoke-RestMethod -Uri "$api/$id" -Method Patch `
  -ContentType 'application/json' -Body $patchBody
```

**Expected:** The title changes; the author and tags remain unchanged.

### 4.3 Delete — `DELETE` and `204`

```powershell
Invoke-WebRequest -Uri "$api/$id" -Method Delete
```

**Expected:** HTTP **204 No Content**; no JSON response body. Requesting the same post again should return HTTP **404**.

### 4.4 Detect bad requests

```powershell
# Contradictory filters -> 400
Invoke-WebRequest -Uri "${api}?author=Awa&tag=node" -Method Get

# Missing title -> 400
Invoke-WebRequest -Uri $api -Method Post `
  -ContentType 'application/json' -Body '{"author":"Fatou"}'
```

**Expected:** These are error responses. PowerShell may **throw an exception on HTTP 4xx** rather than display the error as a normal result. Inspect the exception's status or use the `-SkipHttpErrorCheck` switch in **PowerShell 7+**. In **Windows PowerShell 5.1**, inspect the exception's response instead; `-SkipHttpErrorCheck` is not available there.

**PowerShell note:** The `${api}` syntax explicitly delimits the variable name before the `?` query string. It avoids ambiguous variable expansion in longer commands.

## 5. HTTP methods, paths, parameters, and status codes

| HTTP method | Path | Purpose | Success |
| --- | --- | --- | --- |
| GET | `/api/v1/posts` | List posts / filter | 200 |
| GET | `/api/v1/posts/:id` | Read one post | 200 |
| POST | `/api/v1/posts` | Create one post | 201 |
| PATCH | `/api/v1/posts/:id` | Modify some fields | 200 |
| DELETE | `/api/v1/posts/:id` | Delete a post | 204 |

**Three places inputs come from:**

```javascript
// GET /api/v1/posts?author=Fatou
req.query.author             // 'Fatou'
// GET /api/v1/posts/123
req.params.id                // '123' (string, not a number)
// POST /api/v1/posts, JSON body: {"title":"Hello"}
req.body.title               // 'Hello', after express.json()
```

**Status code vocabulary:** `200 OK` (successful read/update), `201 Created` (created resource), `204 No Content` (deleted without body), `400 Bad Request` (invalid input), `404 Not Found` (missing resource), `409 Conflict` (a conflicting resource state, in appropriate designs), `500 Internal Server Error` (unexpected failure). `401 Unauthorized` and `403 Forbidden` become especially relevant in authentication/authorization chapters.

**Never send JSON after `res.status(204).end()`.** 204 deliberately has no response body. Remember that a route's method matters: `GET /api/v1/posts` is distinct from `POST /api/v1/posts`.

## 6. Path B — Run the actual Chapter 3 Blog API

This section is **source-derived** from the [Chapter 3 GitHub repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development). The project uses **Express `^5.1.0`**, **Mongoose `^8.0.2`**, **MongoDB**, **dotenv `^17.2.3`**, **nodemon**, and **Jest**. The actual project structure is:

```text
from-zero-to-fullstack-ch3-backend-development/
├── .gitignore           # Ignores .env and node_modules
├── package.json         # Scripts: dev, start, lint, test
├── package-lock.json
├── jest.config.json
├── nodemon.json
└── src/
    ├── app.js           # Express setup + express.json()
    ├── index.js         # dotenv, database init, app.listen()
    ├── routes/
    │   └── posts.js     # HTTP endpoints and status codes
    ├── services/
    │   └── posts.js     # Database-backed CRUD functions
    ├── db/
    │   ├── init.js      # Mongoose connection
    │   └── models/
    │       └── post.js  # Post schema
    ├── __tests__/
    │   └── posts.test.js
    └── test/             # MongoDB Memory Server test lifecycle
```

### 6.1 Clone from GitHub and install dependencies

```powershell
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development.git
Set-Location 'from-zero-to-fullstack-ch3-backend-development'
npm.cmd ci
```

**Expected:** Dependencies install from the **existing** lockfile. You are now in the root directory containing `package.json`. If the repository is already cloned, enter that folder rather than cloning again.

### 6.2 Install/start MongoDB and configure a local connection

Review [Cheatsheet 08 — MongoDB Installation & Native Driver](../../chapter-02-backend-fundamentals/08-mongodb-native-driver/README.md) for MongoDB server installation and `mongosh` verification. A default local MongoDB server usually uses **port 27017**; the original Chapter 3 fallback URI is **`mongodb://localhost:27018/blog`**.

Do **not** silently assume the original port is 27017. If your MongoDB server runs on the standard 27017 port, create a new local **`.env`** file at the **Chapter 3 repository root**:

```dotenv
PORT=3000
DATABASE_URL=mongodb://127.0.0.1:27017/blog
```

If you intentionally run MongoDB on port 27018, configure `DATABASE_URL=mongodb://127.0.0.1:27018/blog` instead. Avoid printing live production connection strings in logs or screenshots. The source `.gitignore` already ignores `.env`; the root `.env` file is **created locally**, not provided by the repository.

**Confirm MongoDB first:**

```powershell
Test-NetConnection 127.0.0.1 -Port 27017
mongosh 'mongodb://127.0.0.1:27017/blog'
```

Expected: `TcpTestSucceeded : True` and a working shell connection for port 27017 (adjust if you use 27018). Exit `mongosh` with `.exit`.

### 6.3 Start the real application

```powershell
npm.cmd run dev
```

This runs **`nodemon src/index.js`**. The code in `src/index.js` loads `.env`, waits for the database connection, and only then calls `app.listen(PORT)`.

**Expected:** The terminal logs a successful DB connection followed by an Express server listening on **port 3000**, unless you configured another port. To query the empty collection, open a second terminal:

```powershell
Invoke-RestMethod -Uri 'http://localhost:3000/api/v1/posts'
```

An empty database should return an empty JSON array. To create a record:

```powershell
$post = @{ title = 'My first Blog API post'; author = 'Awa'; contents = 'Hello'; tags = @('express') } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://localhost:3000/api/v1/posts' `
  -Method Post -ContentType 'application/json' -Body $post
```

**Expected:** `201 Created` and a MongoDB document with `_id`, `title`, other submitted fields, and timestamps. Here data **persists** between server restarts as long as MongoDB storage is intact. Do not use the lab's numeric `id` for this project; MongoDB uses `_id` identifiers.

### 6.4 Actual Express initialization — exact source code

`src/app.js`:

```javascript
import express from 'express'
import { postsRoutes } from './routes/posts.js'

const app = express()
app.use(express.json())

postsRoutes(app)

export { app }
```

`src/index.js`:

```javascript
import { app } from './app.js'
import dotenv from 'dotenv'
import { initDatabase } from './db/init.js'

try {
  dotenv.config()
  await initDatabase()
  const PORT = process.env.PORT || 3000
  app.listen(PORT)
  console.info(`express server running on http://localhost:${PORT}`)
} catch (error) {
  console.error('error connecting to database:', error)
}
```

**Why this matters:** Express does not connect to MongoDB automatically. Initialization order ensures that the app does not start listening until the database connection succeeds. The actual `src/db/init.js` uses `mongoose.connect(DATABASE_URL)`, and the `Post` schema defines `title` as required plus optional `author`, `contents`, `tags`, and timestamps.

### 6.5 Actual routes — endpoint contract from `src/routes/posts.js`

| Endpoint | Input | Outcome in current source |
| --- | --- | --- |
| `GET /api/v1/posts` | Optional `author`, `tag`, `sortBy`, `sortOrder` | JSON array; `400` if **both** author and tag |
| `GET /api/v1/posts/:id` | MongoDB document ID | `200` with post; `404` when no matching record |
| `POST /api/v1/posts` | JSON with `title`, `author`, `contents`, `tags` | `201` on success |
| `PATCH /api/v1/posts/:id` | JSON subset of allowed fields | `200` updated post; `404` if missing |
| `DELETE /api/v1/posts/:id` | MongoDB document ID | `204` if deleted; `404` if not found |

The route handlers call the corresponding **service functions** in `src/services/posts.js`. The services use a Mongoose `Post` model for storage. The actual code uses `req.query.tag` (singular), while the schema stores `tags` (plural array); the service searches `{ tags }` for a matching tag value. Avoid mistakenly sending `?tags=react` when you mean `?tag=react`.

**Source fidelity warning:** Existing routes respond with a generic `500` for many unexpected/validation errors; for instance, an invalid MongoDB ID may be treated as `500` rather than `400`. The simple Chapter 3 implementation does **not** yet provide full schema-based HTTP validation, standardized errors, authentication, or hardened sort-field allowlists. The lab illustrates some improvements, but they are **not claimed as source features**.

## 7. Middleware, order, and error handling

**Middleware order is significant:** register JSON parsing **before** handlers that read `req.body`; register routes **before** the 404 fallback; register error-handling middleware last. Typical flow:

```text
Incoming request
    -> express.json()                 parses JSON
    -> route matcher                 e.g. POST /api/v1/posts
    -> route handler                 validates/dispatches
    -> service                       performs work
    -> res.status(...).json(...)     sends response
    -> (if no match) 404 fallback
    -> (if exception) error handler
```

In **Express 5**, rejected promises returned from route handlers are passed to error-handling middleware automatically. The original Chapter 3 routes often use `try/catch` directly and return status `500`. Both approaches can coexist; **do not accidentally send a second response** after calling `res.json()` or `res.end()`.

Useful helpers:

```javascript
app.use(express.json())             // Parse JSON request bodies
res.status(200).json({ ok: true })   // JSON response
req.params.id                        // /posts/:id
req.query.tag                        // /posts?tag=react
req.body                             // parsed JSON body, possibly missing
```

For real projects, validate all untrusted input and add an application-wide error policy. Use an explicit **allowlist** for sorting fields and limit page sizes; do not interpolate arbitrary `req.query` fields into database operations without checking them.

## 8. Troubleshooting (Windows + Node.js + Express)

| Symptom | Likely reason | What to do |
| --- | --- | --- |
| `node` / `npm` not recognized | Node.js not installed or terminal PATH stale | Install official Node.js LTS; reopen the terminal; run `node --version` |
| `npm.ps1 cannot be loaded` | PowerShell script execution restriction | Use `npm.cmd` for npm commands |
| `Cannot find package 'express'` | Dependencies not installed / wrong folder | Run `pwd`, then `npm.cmd install` in the project root |
| `Cannot use import outside a module` | Missing `"type":"module"` | Set `npm.cmd pkg set type=module` in your new lab |
| `Missing script: dev` | Missing script or wrong folder | Check `package.json`; use `npm.cmd run` to list scripts |
| `EADDRINUSE: port 3000` | Another process is using the port | Stop the other server or use a free port |
| `ERR_CONNECTION_REFUSED` | Server stopped / port mismatch | Check console output, address, and port |
| `req.body` undefined | `express.json()` missing / wrong `Content-Type` | Register JSON parsing and send `application/json` |
| `404` on `/posts` | Actual base path has `/api/v1` | Use `/api/v1/posts` |
| MongoDB refused on `27018` | Source fallback differs from your local port | Set `DATABASE_URL` in root `.env` for your active MongoDB port |
| GET returns `500` for a malformed ID | Original route catches Mongoose cast errors as internal errors | Use well-formed IDs when testing; add validation in later improvements |
| `npm test` needs MongoDB binary download | `mongodb-memory-server` downloads required assets | Check internet/cache and revisit testing guide 13 |

**Check the location first:** In Chapter 3, `package.json` is at **repository root**. The README's old `backend/` command is outdated. From Windows PowerShell: `pwd`, `Get-ChildItem`, `Get-Content package.json`.

## 9. Testing and source verification boundaries

**Local lab:** You can exercise the new in-memory Express app using its `/health` and `/api/v1/posts` routes. Successful cases: 200 on listing, 201 on creation, 200 on update, and 204 on deletion. Error cases: 400 for missing title / contradictory filters, and 404 for absent posts. Since it is not database-backed, no MongoDB installation or test fixture is required.

**Real Chapter 3:** The repository provides `npm test`, which runs **Jest** against **MongoDB Memory Server**; its checked-in tests primarily exercise **services** rather than every HTTP route. Run `npm.cmd test` from the root of the cloned project. A database server at port 27017/27018 is not needed for those **in-memory tests**, but the test library may need to download MongoDB binaries initially.

**Verification limitations for this guide:** GitHub source structure and code were inspected directly. The four lab JavaScript files passed `node --check`, and two Node.js tests of the in-memory CRUD service passed. Full HTTP integration tests could not be executed in this authoring environment because the npm registry was unavailable for installing Express; the HTTP commands are therefore clearly marked as **expected results**, not tested network responses. Executing the complete database-backed source also requires a reachable MongoDB instance; this guide does not claim that every Chapter 3 API request has been retested here.

## 10. Security and production recommendations

These improvements are **recommended additions**, not claims about the Chapter 3 implementation:

1. **Validate request bodies and IDs** with a schema validator such as Zod, Joi, or a comparable library.
2. **Whitelist sort fields** (`createdAt`, `updatedAt`, etc.), constrain order, and avoid arbitrary query operators.
3. **Limit request sizes** and consider rate limits, HTTPS, security headers, and CORS rules as appropriate.
4. **Centralize error handling**; return correct 4xx errors for invalid input and a safe generic 500 for unexpected failures.
5. **Do not leak credentials** by logging connection URIs; use redaction if logs need connection metadata.
6. **Use environment variables**; ignore `.env` and publish only sanitized `.env.example` files.
7. **Do not expose unauthenticated MongoDB** to the public network. Use a local-only server for learning.
8. **Test endpoints**, not only service functions. Create automated HTTP integration tests in a future testing module.
9. **Add authentication/authorization** before allowing unrelated users to modify shared data.
10. **Document the API contract** including errors, request validation, filtering, and versioning.

**Portability reminder:** The in-memory lab uses numeric `id` values; the real MongoDB application uses document `_id`. Never confuse the two when copying request examples.

## 11. Reusable project starter — quick workflow

```powershell
# 1) New independent API
New-Item -ItemType Directory -Force -Path "$HOME\Projects\my-api"
Set-Location "$HOME\Projects\my-api"
npm.cmd init -y
npm.cmd pkg set type=module
npm.cmd install express@^5
npm.cmd install --save-dev nodemon
npm.cmd pkg set "scripts.start=node src/index.js"
npm.cmd pkg set "scripts.dev=nodemon src/index.js"

# 2) Create files shown in Sections 3.2–3.5
# 3) Run and test
npm.cmd run dev
# From a second terminal:
Invoke-RestMethod -Uri 'http://127.0.0.1:3000/health'
```

For a **database-backed** project, copy the pattern of `app.js` + `index.js` + `routes/` + `services/`, install/configure Mongoose, and initialize MongoDB **before listening**. See the source project and follow-on cheatsheets; **do not** attempt to replace a database by keeping all data in memory for real customers.

## 12. Final reuse checklist

- [ ] Node.js, npm, Git, and VS Code are installed and verified.
- [ ] I understand the difference between Node.js native HTTP and Express.
- [ ] I created an independent Express 5 project with ES modules.
- [ ] I installed Express/nodemon and configured `start` / `dev` scripts.
- [ ] I separated Express app configuration from `app.listen()`.
- [ ] I can explain `req.params`, `req.query`, `req.body`, and `res.json()`.
- [ ] I registered middleware in the correct order.
- [ ] I built and called `GET`, `POST`, `PATCH`, and `DELETE` endpoints.
- [ ] I recognize statuses `200`, `201`, `204`, `400`, `404`, and `500`.
- [ ] I know in-memory storage resets and MongoDB persists data.
- [ ] I cloned the **root-level** Chapter 3 project and used `npm ci`.
- [ ] I configured `DATABASE_URL` to match my active MongoDB port.
- [ ] I know the current source limitations and recommended future fixes.
- [ ] I can use the quick reference for a new API without replaying the course.

## 13. Sources, versions, and what's next

**Verified in source (15 October 2026):**

- [Chapter 3 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)
- [package.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/package.json) — Express `^5.1.0`, npm scripts, MongoDB/Mongoose/Jest versions.
- [src/app.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/app.js) — `express()` / `express.json()`.
- [src/index.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/index.js) — environment/database initialization and listener.
- [src/routes/posts.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/routes/posts.js) — current HTTP endpoints, filtering, and codes.
- [src/services/posts.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/services/posts.js) — Mongoose-backed operations.
- [src/db/init.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/db/init.js) — default MongoDB URI uses port 27018.
- [src/db/models/post.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/db/models/post.js) — schema and timestamp fields.

**Official documentation:**

- [Node.js downloads](https://nodejs.org/en/download)
- [Express — Installing](https://expressjs.com/en/starter/installing.html)
- [Express — Hello world](https://expressjs.com/en/starter/hello-world.html)
- [Express — Basic routing](https://expressjs.com/en/starter/basic-routing.html)
- [Express — Middleware](https://expressjs.com/en/guide/using-middleware.html)
- [Express 5 — Error handling](https://expressjs.com/en/5x/guide/error-handling/)
- [MongoDB server setup](https://www.mongodb.com/docs/manual/administration/install-community/)

**Scope boundary:** *Source-verified* steps reproduce the actual Chapter 3 repo. The *new teaching lab*, its `/health` endpoint, in-memory posts, explicit 404 middleware, and improved request validation are **additional educational code**, not features present in the checked-in Chapter 3 backend.

---

**Next:** [Cheatsheet 10 — MongoDB with Mongoose](../) · [Chapter 3 index](../) · [Library roadmap](../../README.md)