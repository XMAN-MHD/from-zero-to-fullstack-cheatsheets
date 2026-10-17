# Cheatsheet 12 — Backend Architecture: Routes, Services & Models

**From Zero to Fullstack · Chapter 3: Backend Development**  
**Audience:** Beginners and developers building maintainable Node.js backends  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Version:** 1.0 · 17 October 2026  
**Project studied:** [From Zero to Fullstack — Chapter 3](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)  
**Companion:** [Cheatsheet 11 — Building a Complete CRUD REST API](../11-crud-rest-api/README.md)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Understand and reproduce the real Blog API architecture, then build a separate, testable mini-project that introduces controllers and repository injection. **The real Chapter 3 code and the improved learning lab are explicitly different applications.**

## Quick reference — come back here later

| Concern | Actual Chapter 3 location | Responsibility |
| :--- | :--- | :--- |
| HTTP bootstrap | `src/index.js` | Load environment, connect DB, listen |
| Express configuration | `src/app.js` | Parse JSON, register Blog routes |
| Routes and HTTP handlers | `src/routes/posts.js` | URL/method, params, status, error response |
| Business/data operations | `src/services/posts.js` | CRUD, filtering, sorting, update allowlist |
| Data model | `src/db/models/post.js` | Mongoose schema, document shape, validation |
| Database connection | `src/db/init.js` | `mongoose.connect(DATABASE_URL)` |
| Tests | `src/__tests__/posts.test.js` | Existing service tests |
| Optional lab controller | `examples/architecture-lab/src/controllers/posts.js` | HTTP adapters (not in original) |
| Optional lab repository | `examples/architecture-lab/src/repositories/` | Storage abstraction (not in original) |

**Original source flow:** `HTTP request → Express route handler → service function → Mongoose model → MongoDB`. The Chapter 3 repository **does not** have a separate controller or repository implementation in this source revision. The extra controller/repository layers in this cheatsheet are **optional teaching extensions**.

## 1. Learning outcomes and prerequisites

When finished, you should be able to explain separation of concerns, name each existing source file's responsibility, trace a request through the architecture, avoid coupling business logic to Express, distinguish a Mongoose model from an architectural service or repository, refactor cautiously, test business logic without HTTP, and reuse the structure for a new resource.

You need only basic JavaScript functions, ES module imports/exports, Promise/`async`/`await`, HTTP and JSON knowledge. Cheatsheets 06–11 provide the background. Read the actual source **before** editing it.

### Install essential tools on Windows

Install [Node.js LTS](https://nodejs.org/en/download), [Git for Windows](https://git-scm.com/install/windows), and [VS Code](https://code.visualstudio.com/download). MongoDB is needed to execute the **original** persistent Blog API, but is **not** needed for the separate in-memory lab.

Open PowerShell and verify:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

If a PowerShell execution policy blocks `npm.ps1`, run **`npm.cmd`**, as shown above. This avoids changing system-wide execution policy. If `code` is not on `PATH`, open the folder directly in VS Code using *File → Open Folder*.

## 2. Clone and run the **actual** Chapter 3 backend

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development.git
Set-Location .\from-zero-to-fullstack-ch3-backend-development
npm.cmd ci
code .
```

**Important repo-specific fact:** `package.json` and `src/` are at the **repository root**. An older source README mentions a `backend/` directory, but this GitHub revision does not contain that layout: do **not** run `cd backend` here. The repository uses ES modules (`"type": "module"`), **Express 5**, **Mongoose 8**, `dotenv`, `nodemon`, ESLint and Jest.

### Configure the correct MongoDB port

`src/db/init.js` defaults to `mongodb://localhost:27018/blog`. MongoDB Community Server typically uses port **27017** unless reconfigured. For a standard local installation, create `.env` beside `package.json`:

```dotenv
DATABASE_URL=mongodb://127.0.0.1:27017/blog
PORT=3000
```

The original `.gitignore` excludes `.env`. To check the configured port:

```powershell
Test-NetConnection 127.0.0.1 -Port 27017
# If your MongoDB really listens on 27018, check that instead:
Test-NetConnection 127.0.0.1 -Port 27018
```

Use whichever port matches your actual MongoDB service. Do not commit credentials; **the original `initDatabase()` logs the connection URI**, which may leak secrets if used with credentials. Later in this guide, you will see the recommended safer alternative.

Start the original app from the repository root:

```powershell
npm.cmd run dev
```

In another PowerShell window:

```powershell
Invoke-RestMethod -Uri 'http://localhost:3000/api/v1/posts' -Method Get
```

**Expected:** a JSON list (possibly empty), once the database connects. If startup fails, check MongoDB, `.env`, package installation and the port; do not mistake the separate lab's in-memory behavior for real database persistence.

## 3. The architecture already used in our Blog API

```text
                         Chapter 3 — ACTUAL repository

 React / browser / PowerShell
              |  HTTP + JSON
              v
     src/index.js       --> startup, dotenv, DB connection
              |
     src/app.js         --> express(), express.json()
              |
     src/routes/posts.js
       URL + HTTP method + req/res + status/error handling
              |
     src/services/posts.js
       CRUD + author/tag filtering + sorting
              |
     src/db/models/post.js
       Mongoose schema + validation + timestamps
              |
     src/db/init.js --> mongoose.connect(...) --> MongoDB

   No separate controllers/ or repositories/ files in this version.
```

**Separation of concerns** means each layer has a focused job and communicates through an understandable interface. It does **not** mean every project must have the maximum number of folders. Our current API is a sensible starting structure: its route handlers still contain HTTP decisions, while its service methods perform database operations via a Mongoose model.

| Question | Good home | Avoid putting it in... |
| :--- | :--- | :--- |
| Which URL and method? | Route | Model |
| Which HTTP status to return? | Route/controller | Mongoose model |
| Which fields can be updated? | Service | Generic route wiring |
| How is a post stored or validated? | Model and database | Express request object |
| Where are credentials loaded? | Bootstrap/config | Git-tracked source constants |
| How do we run/stop server? | Entry point | Model |

## 4. Follow one complete request through the real files

### 4.1 Entry point: `src/index.js`

The actual entry point imports `app`, calls `dotenv.config()`, awaits `initDatabase()`, then starts listening on `process.env.PORT || 3000`. This ensures the initial MongoDB connection has been requested and awaited before HTTP requests are accepted.

### 4.2 Express composition: `src/app.js`

The actual source is small and important:

```javascript
import express from 'express'
import { postsRoutes } from './routes/posts.js'

const app = express()
app.use(express.json())
postsRoutes(app)
export { app }
```

`express.json()` parses valid JSON request bodies. `postsRoutes(app)` registers routes against the app. Because the `app` is exported independently from the listener, it is easier to reuse in integration tests and other setups.

### 4.3 HTTP route: `src/routes/posts.js`

The route module imports seven service functions. For a **create** request, it calls:

```javascript
app.post('/api/v1/posts', async (req, res) => {
  try {
    const post = await createPost(req.body)
    return res.status(201).json(post)
  } catch (err) {
    console.error('error creating post', err)
    return res.status(500).end()
  }
})
```

**Responsibility:** receive HTTP data, invoke the service, send `201 Created` for success. This is an exact excerpt of the source behavior, not an idealized example.

**Known limitation:** the current route maps **all create failures**, including likely Mongoose validation failures, to HTTP `500`. A better API would distinguish malformed input (`400`/`422`) from unexpected server problems (`500`). We will not claim this improvement already exists.

### 4.4 Service: `src/services/posts.js`

```javascript
export async function createPost({ title, author, contents, tags }) {
  const post = new Post({ title, author, contents, tags })
  return await post.save()
}
```

The service knows the `Post` model, but does not import Express or call `res.status()`. The module also exports `listAllPosts`, `listPostsByAuthor`, `listPostsByTag`, `getPostById`, `updatePost` and `deletePost`. The `updatePost()` function only copies `title`, `author`, `contents` and `tags` into `$set`; it calls `findOneAndUpdate()` with `{ new: true, runValidators: true }`. This allowlist helps prevent clients from changing arbitrary fields.

**Further improvement:** the current list service constructs a sort key from `sortBy` without explicitly restricting it to supported fields. Validate `sortBy` against an allowlist before using it in production; later add pagination and query limits.

### 4.5 Model: `src/db/models/post.js`

```javascript
import mongoose, { Schema } from 'mongoose'

const postSchema = new Schema(
  {
    title: { type: String, required: true },
    author: String,
    contents: String,
    tags: [String],
  },
  { timestamps: true },
)

export const Post = mongoose.model('post', postSchema)
```

The **schema** defines document structure and validation rules. The **model** (`Post`) exposes Mongoose operations such as `find`, `save`, `findById` and `findOneAndUpdate`. `timestamps: true` maintains `createdAt` and `updatedAt`. Mongoose normally stores model `post` in the collection `posts` by its pluralization convention.

**Important:** a *Mongoose model* is different from the architectural *service*. The model handles data representation and persistence APIs; the service coordinates application operations.

### 4.6 Real request trace

```text
POST /api/v1/posts
Content-Type: application/json
{ "title": "Architecture matters", "tags": ["nodejs"] }
           |
           v
express.json() --> req.body
           |
           v
app.post(...) in src/routes/posts.js
           |
           v
createPost(req.body) in src/services/posts.js
           |
           v
new Post(...).save() using src/db/models/post.js
           |
           v
MongoDB inserts a document
           |
           v
route returns 201 + JSON post
```

If `title` is missing, Mongoose validation fails. The existing route currently replies `500`; this is a known limitation, **not** recommended REST semantics.

## 5. Why and when to introduce a controller layer

A controller is an **HTTP-facing adapter**: it reads request inputs and determines response status and JSON shape. In the **current Chapter 3**, those controller responsibilities live directly inside route callbacks. You do not need to invent a controller file to describe the original source accurately.

For growing APIs, extracting a controller may improve organization:

```text
          OPTIONAL target architecture (not yet in Chapter 3)

HTTP --> Express Router --> Controller --> Service --> Repository --> Model/DB
          URLs/methods     req/res      domain     storage      Mongoose
```

This is a **recommended variation**, not an assertion that every project needs a repository abstraction. Prefer the smallest design that satisfies your actual tests and maintenance needs. A small REST API can safely use `route → service → Mongoose model`, just like our real project.

### Dependency direction and testability

- A **route** should know how to wire endpoints to handlers.
- A **controller** should know about Express request/response, not `mongoose.connect()`.
- A **service** should know business and validation rules, not HTTP status codes.
- A **repository** (optional) should know how to read/write data, not `req` or `res`.
- A **model** should express the storage schema, validation and data access operations.

A powerful technique is **dependency injection**: pass a repository into a service, instead of hard-coding database access. We can then test the service with an in-memory repository and without starting Express or MongoDB.

## 6. Practical lab — create a layered Express API from zero

**This entire lab is new pedagogical code. It is not in the original Chapter 3 repository.** It demonstrates `Router → Controller → Service → Repository`, with **temporary in-memory storage**. A server restart resets its data. Lab IDs are UUID strings; original MongoDB post IDs are ObjectIds.

### 6.1 Create the project on Windows

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects\architecture-lab"
Set-Location "$HOME\Projects\architecture-lab"
npm.cmd init -y
npm.cmd pkg set type=module
npm.cmd install express@^5
New-Item -ItemType Directory -Force src, src\routes, src\controllers, src\services, src\repositories, test
```

Create the files listed below in VS Code; use the content shown in Sections 6.2–6.7. You may also download this cheatsheet's ZIP archive, which already includes a ready-to-run `examples/architecture-lab/` folder. In that case, open that folder instead of typing the files by hand.

**Target tree:**

```text
architecture-lab/
├── package.json
├── src/
│   ├── server.js
│   ├── app.js
│   ├── routes/posts.js
│   ├── controllers/posts.js
│   ├── services/posts.js
│   └── repositories/memory-post-repository.js
└── test/service.test.js
```

### 6.2 The repository: a replaceable storage interface

This implementation uses `Map` and is intentionally **not MongoDB**. It supports `list`, `get`, `create`, `update`, `remove`. The rest of the application can use these operations without knowing how items are stored.

**File: `src/repositories/memory-post-repository.js`**

```javascript
import { randomUUID } from 'node:crypto'

// Exercise-only storage. All data is lost when the server restarts.
export function createMemoryPostRepository() {
  const records = new Map()

  return {
    async list() {
      return [...records.values()]
    },
    async get(id) {
      return records.get(id) ?? null
    },
    async create(data) {
      const now = new Date().toISOString()
      const post = { id: randomUUID(), ...data, createdAt: now, updatedAt: now }
      records.set(post.id, post)
      return post
    },
    async update(id, changes) {
      const post = records.get(id)
      if (!post) return null
      const updated = { ...post, ...changes, updatedAt: new Date().toISOString() }
      records.set(id, updated)
      return updated
    },
    async remove(id) {
      return records.delete(id)
    },
  }
}
```

### 6.3 The service: reusable application rules

The service accepts an injected repository. It validates the post title, validates authors and tags, limits updateable fields, and throws a dedicated `ValidationError`. It **does not import Express** and cannot decide the HTTP status code. The controller handles that translation.

**File: `src/services/posts.js`**

```javascript
// Domain validation is deliberately independent from Express.
export class ValidationError extends Error {
  constructor(message) {
    super(message)
    this.name = 'ValidationError'
  }
}

// The service knows the data interface, but not Express or HTTP.
export function createPostService(repository) {
  return {
    async list() {
      return repository.list()
    },
    async get(id) {
      return repository.get(id)
    },
    async create(input) {
      if (!input || typeof input.title !== 'string' || !input.title.trim()) {
        throw new ValidationError('A non-empty title is required')
      }
      if ('author' in input && typeof input.author !== 'string') {
        throw new ValidationError('author must be text')
      }
      if ('tags' in input && (!Array.isArray(input.tags) || !input.tags.every(t => typeof t === 'string'))) {
        throw new ValidationError('tags must be a list of strings')
      }
      return repository.create({
        title: input.title.trim(),
        author: typeof input.author === 'string' ? input.author : '',
        tags: Array.isArray(input.tags) ? input.tags : [],
      })
    },
    async update(id, changes) {
      if (!changes || typeof changes !== 'object' || Array.isArray(changes)) {
        throw new ValidationError('The update must be a JSON object')
      }
      const allowed = {}
      if ('title' in changes) {
        if (typeof changes.title !== 'string' || !changes.title.trim()) {
          throw new ValidationError('A non-empty title is required')
        }
        allowed.title = changes.title.trim()
      }
      if ('author' in changes) {
        if (typeof changes.author !== 'string') throw new ValidationError('author must be text')
        allowed.author = changes.author
      }
      if ('tags' in changes) {
        if (!Array.isArray(changes.tags) || !changes.tags.every(t => typeof t === 'string')) {
          throw new ValidationError('tags must be a list of strings')
        }
        allowed.tags = changes.tags
      }
      return repository.update(id, allowed)
    },
    async remove(id) {
      return repository.remove(id)
    },
  }
}
```

### 6.4 The controller: translate results into HTTP

The controller has access to `req`/`res`. Its methods return success statuses and handle `404` when a record cannot be found. Unexpected errors travel to Express 5's error middleware (Express 5 automatically forwards rejected promises from async handlers).

**File: `src/controllers/posts.js`**

```javascript
// HTTP concerns belong here: request parsing, status codes, JSON responses.
export function createPostController(service) {
  return {
    list: async (_req, res) => res.status(200).json(await service.list()),
    get: async (req, res) => {
      const post = await service.get(req.params.id)
      return post ? res.status(200).json(post) : res.status(404).json({ error: 'Post not found' })
    },
    create: async (req, res) => {
      const post = await service.create(req.body)
      return res.status(201).json(post)
    },
    update: async (req, res) => {
      const post = await service.update(req.params.id, req.body)
      return post ? res.status(200).json(post) : res.status(404).json({ error: 'Post not found' })
    },
    remove: async (req, res) => {
      const deleted = await service.remove(req.params.id)
      return deleted ? res.status(204).end() : res.status(404).json({ error: 'Post not found' })
    },
  }
}
```

### 6.5 Router: map HTTP verbs to controllers

The router defines endpoints but does not contain business logic or MongoDB calls. `Router()` keeps the module small and reusable.

**File: `src/routes/posts.js`**

```javascript
import { Router } from 'express'

// Declare URLs and verbs here; delegate logic to a controller.
export function createPostRouter(controller) {
  const router = Router()
  router.get('/', controller.list)
  router.get('/:id', controller.get)
  router.post('/', controller.create)
  router.patch('/:id', controller.update)
  router.delete('/:id', controller.remove)
  return router
}
```

### 6.6 Composition root: wire the dependencies

`createApp()` creates a repository, service and controller, registers the router, adds JSON parsing and an error handler, then **returns** the Express app. We deliberately do not call `listen()` in this file. Validation errors return `400`; unexpected errors return `500` with a generic message.

**File: `src/app.js`**

```javascript
import express from 'express'
import { createPostRouter } from './routes/posts.js'
import { createPostController } from './controllers/posts.js'
import { createPostService, ValidationError } from './services/posts.js'
import { createMemoryPostRepository } from './repositories/memory-post-repository.js'

export function createApp() {
  const app = express()
  app.use(express.json({ limit: '32kb' }))

  const repository = createMemoryPostRepository()
  const service = createPostService(repository)
  const controller = createPostController(service)
  app.use('/api/v1/posts', createPostRouter(controller))

  app.use((_req, res) => res.status(404).json({ error: 'Route not found' }))

  app.use((err, _req, res, _next) => {
    if (err instanceof SyntaxError && err.status === 400 && 'body' in err) {
      return res.status(400).json({ error: 'Invalid JSON' })
    }
    if (err instanceof ValidationError) {
      return res.status(400).json({ error: err.message })
    }
    console.error('Unexpected API error:', err)
    return res.status(500).json({ error: 'Internal server error' })
  })
  return app
}
```

### 6.7 Server entry point and package scripts

Only the entry point starts listening. This separation makes the Express app easier to test independently.

**File: `src/server.js`**

```javascript
import { createApp } from './app.js'

const port = Number(process.env.PORT ?? 3000)
createApp().listen(port, '127.0.0.1', () => {
  console.log(`Architecture lab running at http://127.0.0.1:${port}`)
})
```

**File: `package.json`**

```json
{
  "name": "architecture-lab",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js",
    "test": "node --test"
  },
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

The provided lab `package.json` includes `dev`, `start` and `test`. If you typed only `npm init -y` and `npm install` above, replace the generated `package.json` with this version, then run:

```powershell
npm.cmd install
npm.cmd run dev
```

Expected server message: `Architecture lab running at http://127.0.0.1:3000`. **`npm ci` requires a lockfile**; in a brand-new lab without a lockfile, `npm install` is appropriate. The original Chapter 3 clone *does* have a lockfile, so use `npm ci` there.

## 7. Make HTTP requests from PowerShell

Keep the lab server running in its first terminal. Open a **second** PowerShell tab. The commands below work with the lab's temporary in-memory data, not with Chapter 3 MongoDB records.

### 7.1 Create then read

```powershell
$base = 'http://127.0.0.1:3000/api/v1/posts'
$payload = @{ title = 'My layered API'; author = 'Amina'; tags = @('node', 'architecture') } | ConvertTo-Json
$newPost = Invoke-RestMethod -Method Post -Uri $base -Body $payload -ContentType 'application/json'
$newPost
$newPost.id
Invoke-RestMethod -Method Get -Uri $base
Invoke-RestMethod -Method Get -Uri "$base/$($newPost.id)"
```

Expected status for create: `201`; the response includes an `id` string, timestamps, `title`, `author` and `tags`. Get returns `200` and the created post. `$newPost` is used in the next commands: **run them in the same PowerShell session.**

### 7.2 Update then delete

```powershell
$update = @{ title = 'Architecture improved' } | ConvertTo-Json
Invoke-RestMethod -Method Patch -Uri "$base/$($newPost.id)" -Body $update -ContentType 'application/json'
Invoke-WebRequest -Method Delete -Uri "$base/$($newPost.id)" | Select-Object StatusCode
```

Expected PATCH status `200` with the updated object; DELETE status `204` with no response body. Repeat GET after deletion to observe `404`. Note: `Invoke-RestMethod` throws on `4xx/5xx` by default; inspect `$_` in a `catch` block if testing errors.

### 7.3 Invalid requests and missing resources

```powershell
try {
  Invoke-RestMethod -Method Post -Uri $base -Body '{"title":""}' -ContentType 'application/json'
} catch {
  Write-Host "Expected rejection: $($_.Exception.Message)"
}

try {
  Invoke-RestMethod -Method Get -Uri "$base/unknown-id"
} catch {
  Write-Host "Expected not-found: $($_.Exception.Message)"
}
```

Expected: invalid title produces `400`, unknown ID produces `404`, unexpected server errors produce `500`. This behavior is **implemented in the lab**. The original Chapter 3 routes may instead return `500` for some invalid inputs.

## 8. Verify the service with Node's built-in test runner

Because the service and repository do not import Express, these tests run **without an HTTP server or MongoDB and before `npm install`**. The included `test/service.test.js` uses `node:test` and `node:assert/strict`.

**File: `test/service.test.js`**

```javascript
import test from 'node:test'
import assert from 'node:assert/strict'
import { createPostService, ValidationError } from '../src/services/posts.js'
import { createMemoryPostRepository } from '../src/repositories/memory-post-repository.js'

function makeService() {
  return createPostService(createMemoryPostRepository())
}

test('create, list and get are independent of Express', async () => {
  const service = makeService()
  const created = await service.create({ title: ' Clean architecture ', tags: ['node'] })
  assert.equal(created.title, 'Clean architecture')
  assert.equal((await service.list()).length, 1)
  assert.equal((await service.get(created.id)).id, created.id)
})

test('create rejects blank titles', async () => {
  await assert.rejects(makeService().create({ title: '   ' }), ValidationError)
})

test('update allows only specified fields', async () => {
  const service = makeService()
  const created = await service.create({ title: 'Initial' })
  const changed = await service.update(created.id, { title: 'Revised', admin: true })
  assert.equal(changed.title, 'Revised')
  assert.equal(Object.hasOwn(changed, 'admin'), false)
})

test('remove returns false for missing ID', async () => {
  assert.equal(await makeService().remove('not-found'), false)
})

test('records are intentionally ephemeral', async () => {
  const first = makeService()
  await first.create({ title: 'Temporary' })
  const second = makeService()
  assert.deepEqual(await second.list(), [])
})
```

From the lab root:

```powershell
node --test
# Or, using the supplied package script:
npm.cmd test
```

**Expected:** five passing tests: (1) create/list/get, (2) blank-title validation, (3) update allowlist, (4) deleting unknown ID, and (5) ephemeral storage. These five service tests have been executed successfully while preparing this guide. **This does not verify the Express HTTP layer or a MongoDB connection.** Run the HTTP requests from Section 7 after installing Express to verify that layer on your machine.

For the real source repository, use its own **Jest + MongoDB Memory Server** tests:

```powershell
Set-Location "$HOME\Projects\from-zero-to-fullstack-ch3-backend-development"
npm.cmd test -- --runInBand
```

That is a **different suite** from the lab's `node --test`. The original test harness may download a MongoDB binary, so it needs sufficient network access and platform support. Do not run real-source service tests without checking which database the test harness selects.

## 9. Architectural improvements to consider (NOT yet in the original source)

### 9.1 Central error handling and domain validation

- Validate request bodies and route parameters at the API boundary, then keep business validation in the service.
- Use deliberate error classes (such as `ValidationError`) so unexpected JavaScript `TypeError`s do **not** accidentally become client `400` errors.
- Return `400` for malformed JSON or input, `404` for nonexistent posts, and `500` only for unexpected failures. A valid alternative for semantic validation is `422`, if documented consistently.
- Return stable JSON error objects, without exception stacks or credentials.
- Add one centralized Express error middleware after all routes. In **Express 5**, rejected promises from async route handlers automatically reach error middleware if not caught.

### 9.2 Repository injection is optional, not mandatory

A large application may define a `PostRepository` interface implemented by `MongoPostRepository`, and pass it to `createPostService()`. This makes tests and database migrations easier. For a small CRUD app, directly importing a Mongoose model inside a service is also reasonable. Avoid generating folders that do not contain meaningful responsibilities.

### 9.3 Input and query safeguards

- Restrict updateable fields; our actual service already uses an allowlist for `title`, `author`, `contents`, `tags`.
- Validate ObjectIds before calling Mongoose methods; malformed values may raise `CastError` rather than yield a genuine `404`.
- Whitelist sort keys (e.g. `createdAt`, `updatedAt`, `title`) and directions; do not directly trust arbitrary `sortBy` values.
- Apply pagination and sensible size limits to list endpoints.
- Limit JSON body size, validate string lengths and arrays, and never put untrusted data directly in MongoDB operators.
- Add authentication, authorization, request logging/redaction, rate limiting and CORS policies when a real product needs them. These are **not** implemented by the lab or automatically provided by Express.

### 9.4 Safer database startup and shutdown

The original `src/db/init.js` prints the full `DATABASE_URL` after connection. For projects with credentials, log a constant success message instead:

```javascript
import mongoose from 'mongoose'

export async function initDatabase() {
  const uri = process.env.DATABASE_URL
  if (!uri) throw new Error('DATABASE_URL is required')
  await mongoose.connect(uri)
  console.info('MongoDB connection established') // never log credentials
}
```

This is a **proposed replacement**, not code already committed to Chapter 3. Add graceful SIGINT/SIGTERM handling and health checks for production deployments. Use environment variables and a secrets manager; never commit `.env`.

## 10. Common architecture mistakes and troubleshooting

| Symptom | Likely cause | What to do |
| :--- | :--- | :--- |
| `cd backend` fails | Old README refers to outdated layout | Use repository root; inspect `src/` and `package.json` |
| `Cannot find module 'express'` | Dependencies missing in the lab | Run `npm.cmd install` inside `architecture-lab` |
| `ECONNREFUSED` on 3000 | Server not started or wrong port | Keep server running; check `PORT` and terminal |
| Original MongoDB connection fails | 27018/27017 mismatch or server offline | Check `.env`, MongoDB service, `Test-NetConnection` |
| Original invalid payload returns `500` | Route catches validation as generic server error | Known source limitation; improve error mapping separately |
| Lab `GET` returns `[]` after restart | The lab uses an in-memory `Map` | Expected; use MongoDB for persistence |
| Lab `404` after `DELETE` | Record no longer exists | Expected; test create again |
| `node --test` cannot import ES modules | Missing `"type": "module"` or wrong folder | Use provided `package.json`, run from lab root |
| PowerShell blocks `npm.ps1` | Execution policy | Use `npm.cmd` without changing system policy |
| Test suite downloads MongoDB | `mongodb-memory-server` setup | Allow binary download or configure a compatible test environment |
| Circular imports or difficult tests | Too much layer coupling | Keep dependency flow one-way; inject repository into service |

### Architectural code review checklist

Before approving a new endpoint, ask: Where is the URL declared? Who translates HTTP request to an application command? Does the service know `req` or `res`? What validates the data? What persists it? How are expected errors mapped? Can service tests run without opening a TCP port? Which code is reusable from another frontend?

## 11. Quick reference commands

| Action | Windows PowerShell |
| :--- | :--- |
| Check Node | `node --version` |
| Check npm | `npm.cmd --version` |
| Clone real Chapter 3 | `git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development.git` |
| Install existing source | `npm.cmd ci` |
| Start original API | `npm.cmd run dev` |
| Run real Jest suite | `npm.cmd test -- --runInBand` |
| Install the lab | `npm.cmd install` |
| Start lab | `npm.cmd run dev` |
| Test lab service only | `node --test` |
| Test list endpoint | `Invoke-RestMethod -Uri 'http://127.0.0.1:3000/api/v1/posts'` |
| Check MongoDB port | `Test-NetConnection 127.0.0.1 -Port 27017` |

## 12. Reuse checklist for the next backend resource

- [ ] Define one resource and its HTTP URLs, verbs and response codes.
- [ ] Decide whether `route → service → model` is enough; add controller/repository layers **only if useful**.
- [ ] Keep HTTP details (`req`, `res`, status codes) out of services and models.
- [ ] Define a schema and validate required fields and input types.
- [ ] Restrict editable properties and supported sorting/filtering keys.
- [ ] Keep secrets outside source control; never log connection credentials.
- [ ] Separate Express app composition from server listening.
- [ ] Add service tests without networking, plus HTTP integration tests when dependencies are available.
- [ ] Check success, not-found, validation and unexpected-error paths.
- [ ] Document how data is stored and whether it survives server restart.
- [ ] Keep a README with verified setup commands for Windows.
- [ ] Execute the tests and record exactly what passed, failed or was not run.

## 13. Source notes and official documentation

**Verified repository files (read on 17 October 2026):**

- Bootstrap: `src/index.js`, `src/app.js`, `src/db/init.js`.
- API layers: `src/routes/posts.js`, `src/services/posts.js`.
- Persistence/config: `src/db/models/post.js`, `package.json`, and the existing Jest test folder.

The repo has direct route handlers + services + Mongoose, not standalone controller/repository modules. The original source README describes some obsolete directory paths; this guide follows the source tree.

**Exercise provenance:** `examples/architecture-lab/` in the downloadable ZIP is an original companion exercise for this guide; it is **not** from the Chapter 3 repository. Five Node built-in service tests passed while authoring; no MongoDB-connected integration test or Express HTTP integration test is claimed as performed.

Recommended references:

- [Node.js documentation — test runner](https://nodejs.org/api/test.html)
- [Node.js downloads](https://nodejs.org/en/download)
- [Express — Routing](https://expressjs.com/en/guide/routing.html)
- [Express — Writing middleware](https://expressjs.com/en/guide/writing-middleware.html)
- [Express — Error handling (including Express 5)](https://expressjs.com/en/guide/error-handling.html)
- [Mongoose — Models](https://mongoosejs.com/docs/models.html)
- [Mongoose — Validation](https://mongoosejs.com/docs/validation.html)
- [MongoDB Community Server](https://www.mongodb.com/try/download/community)
- [GitHub — Chapter 3 source](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)

---

**Next:** Cheatsheet 13 — *Automated Backend Testing with Jest.*

**Learn → Practice → Document → Reuse.**
