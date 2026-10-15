# Cheatsheet 10 — MongoDB with Mongoose

**From Zero to Fullstack · Chapter 3: Backend Development**  
**Audience:** Absolute beginners and developers building reusable Node.js applications  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Documentation version:** 1.0 · 15 October 2026  
**Original source:** [Chapter 3 — Backend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install Mongoose, understand schemas, models, documents and validation, connect Node.js to MongoDB, reproduce the Blog post model and safely execute database operations. This guide is based on the actual Chapter 3 code; beginner exercises and recommended hardening are clearly identified as additions.

## Quick reference — for future projects

| Task | Command or pattern |
| --- | --- |
| Verify prerequisites | `node --version`; `npm.cmd --version`; `git --version` |
| Install Mongoose in a new Node project | `npm.cmd install mongoose@^8` |
| Existing Chapter 3 dependencies | `npm.cmd ci` (repository root) |
| Run the actual backend | `npm.cmd run dev` |
| Run the existing tests | `npm.cmd test -- --runInBand` |
| Connect to local MongoDB | `await mongoose.connect('mongodb://127.0.0.1:27017/blog')` |
| Chapter 3 default URI | `mongodb://localhost:27018/blog` |
| Read the configured URI | `process.env.DATABASE_URL` |
| Define model | `mongoose.model('post', postSchema)` |
| Create | `await Post.create({ title: 'Hello' })` |
| Read | `await Post.find({ author: 'Amina' })` |
| Find by id | `await Post.findById(id)` |
| Update + validate | `await Post.findOneAndUpdate({ _id: id }, { $set: updates }, { new: true, runValidators: true })` |
| Delete | `await Post.deleteOne({ _id: id })` |
| Close connection | `await mongoose.disconnect()` |

**Source fidelity:** Chapter 3 stores `src/`, `package.json`, and `jest.config.json` **at the repository root**, not inside `backend/`. Its current `README.md` describes an older nested layout, but the published code is flat. **Run all source-project commands from the cloned root.**

## 1. What are MongoDB, the native driver and Mongoose?

**MongoDB** stores documents in collections inside databases. A document looks similar to a JavaScript object, but is persisted in **BSON**, with types such as `ObjectId` and `Date`. A **collection** is roughly analogous to a SQL table, but the document database model is different.

The **MongoDB Node.js driver** (Chapter 2) communicates directly with MongoDB through `MongoClient`. **Mongoose** is an **ODM (Object Document Mapper)** that runs on top of the MongoDB driver. It lets us organize JavaScript data using **schemas, models, document methods, validation, query helpers, and middleware**.

- **Schema:** Describes document fields, types, constraints and options.
- **Model:** A class compiled from a schema; provides collection queries and document construction.
- **Document:** A JavaScript instance of a model, corresponding to a stored MongoDB document when saved.
- **Connection:** The active link from Node.js to a MongoDB instance.
- **Query:** A description of which documents to retrieve, modify or remove.
- **Validation:** Checks a document's constraints; it is not the same as authorization or complete input sanitization.

```text
Client / Blog frontend
         |
      HTTP / JSON
         v
Express route: src/routes/posts.js
         |
Service layer: src/services/posts.js
         |
Mongoose model: src/db/models/post.js
         |
Mongoose connection: src/db/init.js
         |
MongoDB: database 'blog' -> collection 'posts'
```

**Why use Mongoose?** When building our Blog API, a Post has a required title, optional author and content, an array of tags and automatic timestamps. Mongoose expresses these rules centrally and lets the service layer perform CRUD operations with model methods.

## 2. Prerequisites and installation from scratch

You need basic JavaScript knowledge (objects, arrays, functions, promises, `async/await`) and the ability to open a PowerShell terminal. No prior Mongoose experience is assumed.

### 2.1 Install the developer tools

1. Download and install [Visual Studio Code](https://code.visualstudio.com/download).
2. Download [Git for Windows](https://git-scm.com/install/windows).
3. Install a supported [Node.js LTS](https://nodejs.org/en/download), such as Node.js 22 LTS. npm is included with the normal Node.js installer.
4. Restart PowerShell to refresh `PATH`.
5. Verify:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

**Expected:** Each command prints a version. We use `npm.cmd` because some Windows PowerShell systems block `npm.ps1` under their script policy. There is no need to weaken the machine-wide execution policy.

### 2.2 Install MongoDB Community Server and MongoDB Shell

For a local database on Windows:

1. Open the official [MongoDB Community Server download page](https://www.mongodb.com/try/download/community).
2. Select **Windows** and the available **MSI installer** appropriate to your CPU.
3. Run the installer. If offered, **install MongoDB as a Windows service** so it can run independently of your terminal. Keep the service's port at **27017** unless you have a reason to change it.
4. Install **MongoDB Shell (`mongosh`)** separately if it was not included: [official MongoDB Shell downloads](https://www.mongodb.com/try/download/shell).
5. Optionally install [MongoDB Compass](https://www.mongodb.com/try/download/compass) for a graphical database view.
6. Restart the terminal and check the database:

```powershell
mongosh --version
Get-Service *MongoDB*
Test-NetConnection 127.0.0.1 -Port 27017
mongosh "mongodb://127.0.0.1:27017"
```

**Expected:** The service is running, the TCP port test returns `TcpTestSucceeded : True`, and the MongoDB shell connects. If the Windows service was **not** installed, start the `mongod` server according to the official installation instructions, with a writable data directory. The server must be running before Mongoose connects.

**Important source-specific detail:** Our Chapter 3 code defaults to **`mongodb://localhost:27018/blog`**, while the standard local MongoDB port is **27017**. Either run MongoDB on port 27018 as in the original environment **or set `DATABASE_URL` to your actual port**. Do not change ports blindly. For localhost IPv6 resolution issues on modern Node.js, prefer `127.0.0.1` in new connection strings.

If you have already installed MongoDB while following [Cheatsheet 08](../../chapter-02-backend-fundamentals/08-mongodb-native-driver/README.md), you can reuse that installation. Do not expose an unauthenticated local server to the internet.

### 2.3 Optional cloud alternative

MongoDB Atlas is another supported choice: create a cluster in [MongoDB Atlas](https://www.mongodb.com/atlas), configure network access and a database user, then store the MongoDB URI only in an **untracked** `.env` file. Atlas is **not required** for this guide. Cloud credentials and network access differ from local setup; follow the Atlas onboarding instructions.

## 3. Path A — Learn schemas and validation without any database

This **independent exercise** is not copied from the Chapter 3 repository. It demonstrates that Mongoose's document validation runs in JavaScript without an active database connection; persistence still requires MongoDB.

### 3.1 Create the learning project

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
New-Item -ItemType Directory -Force -Path "mongoose-learning-lab"
Set-Location "mongoose-learning-lab"
npm.cmd init -y
npm.cmd pkg set type=module
npm.cmd install mongoose@^8
code .
```

**Expected:** A `package.json`, `package-lock.json` and `node_modules/` folder appear. The `mongoose` package belongs in `dependencies` (not only dev dependencies). The standalone validation lab does not need the MongoDB service running.

### 3.2 Create `validate-post.js`

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

const Post = mongoose.model('LabPost', postSchema)

const validPost = new Post({
  title: 'My first Mongoose document',
  tags: ['nodejs', 'mongodb'],
})
console.log('Valid:', validPost.validateSync() === undefined)
console.log('Generated ObjectId:', validPost._id.toString())
console.log('Unsaved timestamps:', validPost.createdAt) // undefined

const invalidPost = new Post({ author: 'Amina' })
const error = invalidPost.validateSync()
console.log('Missing title:', error?.errors?.title?.message)
```

Run it:

```powershell
node .\validate-post.js
```

**Expected:** The first document passes validation and receives an `ObjectId`; the second produces a validation error for `title`. No data is saved to MongoDB, and `createdAt` is **not yet set** on the unsaved document. Timestamps are managed during persistence operations, not by just constructing a new document.

### 3.3 Understand each piece

| Code | Meaning |
| --- | --- |
| `new Schema({...})` | Define allowed paths and their types. |
| `title: { type: String, required: true }` | Title is a required string. |
| `tags: [String]` | Array of strings; the default for this array path is normally `[]`. |
| `{ timestamps: true }` | Maintain `createdAt` and `updatedAt` on save/update. |
| `mongoose.model('LabPost', postSchema)` | Compile a model named `LabPost`. |
| `new Post({...})` | Create an unsaved model instance. |
| `validateSync()` | Run synchronous schema validation without saving. |
| `_id` | MongoDB document identifier (usually an ObjectId). |

The real project uses **`mongoose.model('post', postSchema)`**, which maps to the **`posts`** collection under standard pluralization. `LabPost` is used here only to keep the standalone exercise clearly separate.

## 4. Path B — Clone and understand our real Chapter 3 Blog model

### 4.1 Clone and install dependencies

```powershell
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development.git
Set-Location "from-zero-to-fullstack-ch3-backend-development"
npm.cmd ci
code .
```

**Expected:** The repository root contains `package.json`, `jest.config.json`, `src/app.js`, `src/index.js`, `src/db/init.js`, `src/db/models/post.js`, `src/routes/posts.js`, `src/services/posts.js`, and `src/__tests__/posts.test.js`.

The checked Chapter 3 `package.json` declares **`mongoose: ^8.0.2`**, **`mongodb: ^6.3.0`**, **`dotenv: ^17.2.3`**, **`express: ^5.1.0`**, and **`mongodb-memory-server: ^9.1.1`**. These are *declared version ranges*, not a claim about your currently installed exact versions. Run `npm.cmd ls mongoose` to see the resolved version.

### 4.2 Configuration: `.env` and `DATABASE_URL`

The original `src/db/init.js` reads `process.env.DATABASE_URL` and otherwise defaults to `mongodb://localhost:27018/blog`.

Create a local `.env` file at the **repository root**:

```dotenv
DATABASE_URL=mongodb://127.0.0.1:27017/blog
PORT=3000
```

This file contains **example local-development values only**. Replace the port with the MongoDB port running on your computer. The original repository's `.gitignore` includes `.env`, so your local settings should not be committed. To check:

```powershell
git status --short
npm.cmd ls mongoose
```

**Expected:** `.env` does not appear in the Git status output; Mongoose's installed version appears in `npm ls`.

> **Security warning:** A database URI may contain a password. Never put a real credential in a cheatsheet, a commit, a public screenshot, or an application log. Our original `initDatabase()` logs the complete URI on the connection `open` event; **do not use that log statement with credential-bearing cloud URIs**. Prefer a redacted message such as `MongoDB connection established` in production.

### 4.3 Reproduce the real connection file

File: `src/db/init.js` — **original Chapter 3 source**:

```javascript
import mongoose from 'mongoose'

export function initDatabase() {
  const DATABASE_URL =
    process.env.DATABASE_URL || 'mongodb://localhost:27018/blog'
  mongoose.connection.on('open', () => {
    console.info('successfully connected to database:', DATABASE_URL)
  })
  const connection = mongoose.connect(DATABASE_URL)
  return connection
}
```

**Explanation:** `mongoose.connect()` starts the database connection and returns a promise. The exported `initDatabase()` returns this promise so the application can wait for the connection. The `open` listener reports successful connection, although logging the full connection string is not a good practice if it contains secrets.

### 4.4 Understand the actual Post model

File: `src/db/models/post.js` — **original Chapter 3 source**:

```javascript
import mongoose, { Schema } from 'mongoose'

const postSchema = new Schema(
  {
    title: { type: String, required: true },
    author: String,
    contents: String,
    tags: [String],
  },
  {
    timestamps: true,
  },
)

export const Post = mongoose.model('post', postSchema)
```

| Field | Type | Validation / behavior |
| --- | --- | --- |
| `_id` | `ObjectId` | Generated identifier (MongoDB document). |
| `title` | `String` | **Required** by the schema. |
| `author` | `String` | Optional in Chapter 3. |
| `contents` | `String` | Optional in Chapter 3. |
| `tags` | `[String]` | String array; may be empty. |
| `createdAt` | `Date` | Added by `timestamps: true` when persisted. |
| `updatedAt` | `Date` | Managed automatically when created/updated. |

**Important:** The model does **not** currently enforce a minimum title length, unique post title, mandatory author, or a specific tag vocabulary. Those would be additional product requirements, **not existing Chapter 3 validators**.

### 4.5 Understand the real startup sequence

File: `src/index.js` — **original source**:

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

Order: **load `.env` → connect MongoDB → listen for HTTP requests**. The server does not start listening until the initial database connection resolves. The original catch logs errors; a deployment-ready improvement would also set a non-zero exit status so process managers know startup failed.

Run:

```powershell
npm.cmd run dev
```

**Expected:** If MongoDB is running and `.env` is correct, nodemon starts the app and the terminal reports the database connection plus the HTTP server. If a connection fails, inspect the URI, service, port and firewall instead of changing unrelated code.

### 4.6 How Express uses the model

The **real** app has these layers:

```text
src/index.js           start server / connect database
src/app.js             create Express app, parse JSON, register routes
src/routes/posts.js    handle HTTP parameters and responses
src/services/posts.js  execute Mongoose queries
src/db/models/post.js  schema and compiled Post model
src/db/init.js         establish MongoDB connection
```

Example from `src/services/posts.js`:

```javascript
import { Post } from '../db/models/post.js'

export async function createPost({ title, author, contents, tags }) {
  const post = new Post({ title, author, contents, tags })
  return await post.save()
}
```

The service accepts the needed fields, constructs a document and calls `save()`. Mongoose validates it before saving. A missing title causes a `mongoose.Error.ValidationError` rather than creating a valid Post.

## 5. Mongoose CRUD operations — the actual Blog service

Each example below corresponds to a real service operation. **Do not run destructive commands against a production database.** Use a local sandbox database for experiments.

### 5.1 Create and save

```javascript
const post = new Post({
  title: 'Learning Mongoose',
  author: 'Amina',
  contents: 'My first persisted document',
  tags: ['mongoose', 'nodejs'],
})
const createdPost = await post.save()
console.log(createdPost._id)
```

This mirrors the actual `createPost()` service. `Post.create()` is a useful shorthand for the create-and-save operation in simple cases.

### 5.2 Read and filter

```javascript
const allPosts = await Post.find({})
const byAuthor = await Post.find({ author: 'Amina' })
const byTag = await Post.find({ tags: 'nodejs' })
const onePost = await Post.findById('000000000000000000000000')
```

`Post.find()` returns a list. `findById()` returns one document or `null` when a **valid-looking** ObjectId does not exist. An **invalid ObjectId string** can raise a `CastError`, which is different from a not-found result. Arrays can match a scalar query (`{ tags: 'nodejs' }`) when the array contains that value.

### 5.3 Sort

The Chapter 3 `listPosts()` helper defaults to newest creation time first:

```javascript
const sortValue = sortOrder === 'ascending' ? 1 : -1
const sort = { [sortBy]: sortValue }
return await Post.find(query).sort(sort)
```

- `1` means ascending; `-1` means descending.
- The actual service defaults to `sortBy = 'createdAt'` and `sortOrder = 'descending'`.
- **Future improvement:** whitelist valid sort fields before building `{ [sortBy]: ... }` from user input. This is not implemented in the current Chapter 3 service.

### 5.4 Update safely

The real `updatePost()` service copies only these allowed properties: `title`, `author`, `contents` and `tags`, then runs:

```javascript
return await Post.findOneAndUpdate(
  { _id: postId },
  { $set: updatedFields },
  { new: true, runValidators: true },
)
```

- `$set` updates specified properties without replacing the whole document.
- `new: true` returns the updated document rather than the previous one.
- `runValidators: true` enables Mongoose **update validators**, which are disabled by default on many update methods.
- Update validation has limitations: it generally checks updated paths rather than revalidating the full stored document. Do not assume `runValidators` makes every update automatically safe.
- The service's field allowlist prevents clients from updating arbitrary properties through this service function.

### 5.5 Delete

```javascript
const result = await Post.deleteOne({ _id: postId })
console.log(result.deletedCount) // 1 if deleted, 0 if not found
```

The actual Express route returns **HTTP 204** when deletion succeeds and **404** if `deletedCount` is zero.

## 6. Hands-on exercise — safe CRUD with a separate lab database

**Additional exercise, not present in the original repository.** To experiment without modifying the Blog database, use a second connection string in **another copy of the project**, or create a new `.env` with a dedicated local database named `blog_cheatsheet10`. Make sure no important data exists there; test data should not be mixed with real application data.

A simple standalone script can reuse the Chapter 3 Post model. Create the optional file `scripts/mongoose-lab.js` in your **local clone**, not in the source repository on GitHub:

```javascript
import dotenv from 'dotenv'
import mongoose from 'mongoose'
import { Post } from '../src/db/models/post.js'

dotenv.config()
const uri = process.env.DATABASE_URL
if (!uri || !uri.includes('/blog_cheatsheet10')) {
  throw new Error('Set DATABASE_URL to a dedicated blog_cheatsheet10 database')
}

let createdId
try {
  await mongoose.connect(uri)
  const created = await Post.create({
    title: 'Lab post',
    author: 'Cheatsheet Student',
    tags: ['mongoose', 'lab'],
  })
  createdId = created._id
  console.log('Created ID:', createdId.toString())

  const posts = await Post.find({ author: 'Cheatsheet Student' })
  console.log('Matching posts:', posts.length)

  const updated = await Post.findOneAndUpdate(
    { _id: createdId },
    { $set: { title: 'Lab post updated' } },
    { new: true, runValidators: true },
  )
  console.log('Updated title:', updated.title)

  const deleted = await Post.deleteOne({ _id: createdId })
  createdId = undefined
  console.log('Deleted count:', deleted.deletedCount)
} finally {
  // Try to clean up only the document created by this exercise.
  if (createdId) await Post.deleteOne({ _id: createdId })
  await mongoose.disconnect()
}
```

**Safety note:** The URI check is only a beginner guardrail, not a security boundary; always inspect the actual host/database in your `.env` before running write or delete operations. Use a local database named `blog_cheatsheet10`:

```dotenv
DATABASE_URL=mongodb://127.0.0.1:27017/blog_cheatsheet10
```

Run from the cloned Chapter 3 root (with a local MongoDB service running):

```powershell
New-Item -ItemType Directory -Force -Path .\scripts
node .\scripts\mongoose-lab.js
```

**Expected output:** A created ObjectId, at least one matching post, an updated title and `Deleted count: 1`. The example removes its own created document. It does not remove the database or unrelated documents. If the MongoDB connection fails, check service status, port, URI, or network access first.

### 6.1 Two validation experiments

In a separate local script with Mongoose installed, test the **real schema** before persistence:

```javascript
import { Post } from './src/db/models/post.js'

const missingTitle = new Post({ author: 'Amina' })
console.log(missingTitle.validateSync()?.errors?.title?.message)

const good = new Post({ title: 'Valid title' })
console.log(good.validateSync() === undefined) // true
```

Run from the repository root with an appropriate file path. Validation can be demonstrated offline because no `.save()` is required. A database is required for saving or querying persisted documents.

## 7. Testing the real Chapter 3 code

The repository contains **Jest** tests in `src/__tests__/posts.test.js` and uses **MongoDB Memory Server** during its test setup:

```text
jest.config.json
  -> src/test/globalSetup.js       starts MongoDB Memory Server
  -> src/test/setupFileAfterEnv.js connects Mongoose
  -> src/__tests__/posts.test.js    checks Post service operations
  -> src/test/globalTeardown.js    stops the test database
```

Useful commands:

```powershell
npm.cmd run lint
npm.cmd test -- --runInBand
```

Tests cover creating valid posts, rejecting missing titles, reading/filtering/sorting, finding one, updating and deleting. Test setup downloads/uses a MongoDB binary and may require internet access and an OS that supports the requested MongoDB version. The source explicitly asks MongoDB Memory Server for binary version `6.0.4`; compatibility with your operating system may require adjustments **for new projects**. See **Cheatsheet 13** for a detailed Jest/testing walkthrough.

**Verification boundary:** The source code and test definitions have been inspected. This document does **not** claim that the full database integration suite was executed in the documentation environment.

## 8. Common errors and troubleshooting

| Symptom | Likely cause | Resolution |
| --- | --- | --- |
| `ECONNREFUSED 127.0.0.1:27017` | MongoDB not running or wrong port | Start the MongoDB service and check port `27017`/`27018`. |
| `MongooseServerSelectionError` | URI/network/service mismatch | Check `.env`, firewall and server availability. |
| `MongooseError: Operation ... buffering timed out` | Query ran while disconnected | Connect first with `await mongoose.connect(...)`. |
| `MissingSchemaError` | Model not registered | Import the model's defining module before using it. |
| `OverwriteModelError` | Same model name compiled multiple times | Reuse compiled model or avoid repeated compilation in reloadable code. |
| `ValidationError: title ... required` | Attempt to save a Post without title | Provide a valid string title. |
| `CastError: Cast to ObjectId failed` | Invalid ID format | Validate/handle user input before querying. |
| Update bypasses validation | Query validators not enabled | Use `runValidators: true` and know its limitations. |
| `npm.ps1 cannot be loaded` | PowerShell execution policy | Use `npm.cmd` without changing global policy. |
| `Cannot find module 'mongoose'` | Dependencies absent or wrong working directory | Run `npm.cmd ci` in project root. |
| Secrets visible in logs | Connection URI includes credentials | Remove logging of raw database URI. |
| Tests cannot download MongoDB | Network restrictions / binary compatibility | Check test setup and MongoDB Memory Server docs. |

**Diagnostic sequence (PowerShell):**

```powershell
Get-Location
npm.cmd ls mongoose
mongosh --version
Get-Service *MongoDB*
Test-NetConnection 127.0.0.1 -Port 27017
# Check the file privately in VS Code; do not paste or share its values.
```

**Never share real `.env` values.** Review the file privately in VS Code and redact any connection strings before sharing diagnostic output. Prefer example-only connection strings when discussing errors.

## 9. Security, reliability and best practices

These items include **recommendations for future projects** and must not be confused with the current Chapter 3 implementation:

1. **Protect credentials:** `.env` should be ignored by Git. Use an example-only `.env.example`, safe rotation practices and a secret manager for production.
2. **Never log full URIs:** The original `initDatabase()` logs the connection string; change this before using real database credentials.
3. **Validate inputs at boundaries:** Mongoose validation helps but does not replace checking HTTP request shape, size, authorization, and errors.
4. **Whitelist editable fields:** Our `updatePost()` service already does this for the four post properties.
5. **Whitelist sort keys:** Restrict user-provided sorting fields; the original service currently interpolates `sortBy`.
6. **Handle errors by category:** Invalid ObjectIds and validation errors should normally become clear client-side HTTP errors, not be treated as generic server failures. The current Chapter 3 routes do not distinguish every error category.
7. **Use indexes intentionally:** Create indexes based on real query patterns and performance evidence; Chapter 3 has no explicit custom indexes on the Post schema.
8. **Manage startup/shutdown:** Connect before accepting traffic and disconnect on graceful shutdown when appropriate.
9. **Avoid network exposure:** A local MongoDB service for practice should listen only on trusted interfaces unless authentication/network protections are configured.
10. **Separate environments:** Use dedicated test/dev databases; never practice destructive operations against production.

### Optional improved connection pattern (not Chapter 3 source)

```javascript
import mongoose from 'mongoose'

export async function connectDatabase() {
  if (!process.env.DATABASE_URL) {
    throw new Error('DATABASE_URL is required')
  }
  await mongoose.connect(process.env.DATABASE_URL)
  console.info('MongoDB connection established') // no secrets in logs
}
```

This is a recommended new-project pattern: require the variable instead of silently using a default, await initial connection and log a safe status message. A production application should also implement a suitable shutdown strategy.

## 10. Reusable starter template

```text
my-node-api/
├── .env                 # Local only, ignored by Git
├── .env.example         # Safe placeholder values, versioned
├── .gitignore
├── package.json
└── src/
    ├── index.js         # Startup: dotenv, database, listen
    ├── app.js           # Express app
    ├── db/
    │   ├── init.js      # mongoose.connect()
    │   └── models/
    │       └── post.js  # Schema + model
    ├── routes/
    │   └── posts.js     # HTTP layer
    └── services/
        └── posts.js     # Query/service layer
```

**Checklist before reusing Mongoose:**

- [ ] Node.js and npm installed; `npm.cmd` works.
- [ ] MongoDB Community Server (or a trusted Atlas database) is accessible.
- [ ] Required packages are installed (`mongoose`, `dotenv`, Express if using HTTP).
- [ ] `.env` is ignored by Git and contains the correct host/port/database name.
- [ ] Application waits for the initial database connection.
- [ ] Schema field types and required constraints match business requirements.
- [ ] CRUD service functions use intended models and safe query filters.
- [ ] Updates run appropriate validation and whitelist editable fields.
- [ ] Invalid identifiers and user validation failures receive appropriate handling.
- [ ] Secrets are absent from logs, source code and published documentation.
- [ ] Local/test data is separate from real business data.
- [ ] Tests are run and their actual results are recorded.

**What we actually achieved in Chapter 3:** a Mongoose `post` model, a MongoDB connection helper, CRUD services, an Express API, and source-defined Jest tests. This guide's standalone labs and hardening suggestions are **supplementary training material**.

## 11. Sources and version notes

**Original Chapter 3 project (examined 15 October 2026):**

- [Project repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)
- [`src/db/init.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/db/init.js)
- [`src/db/models/post.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/db/models/post.js)
- [`src/services/posts.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/services/posts.js)
- [`src/routes/posts.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/routes/posts.js)
- [`src/__tests__/posts.test.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/__tests__/posts.test.js)

**Official documentation:**

- [Mongoose 8 documentation](https://mongoosejs.com/docs/8.x/docs/)
- [Mongoose connections](https://mongoosejs.com/docs/connections.html)
- [Mongoose schemas](https://mongoosejs.com/docs/guide.html)
- [Mongoose validation and update validators](https://mongoosejs.com/docs/validation.html)
- [Mongoose timestamps](https://mongoosejs.com/docs/timestamps.html)
- [MongoDB Community Server](https://www.mongodb.com/try/download/community)
- [MongoDB Shell](https://www.mongodb.com/try/download/shell)

**Compatibility:** This guide documents **Mongoose 8** because that is the project's declared major version. Official Mongoose documentation may now default to a newer major version; behavior or available options can differ. Always check your installed `mongoose` version. Source-derived observations and recommendations are deliberately separated throughout.

---

**Next planned:** Cheatsheet 11 — Building a Complete CRUD REST API.  
[Back to Chapter 3](../README.md) · [Full library roadmap](../../README.md)