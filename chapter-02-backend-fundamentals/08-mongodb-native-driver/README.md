# Cheatsheet 08 — MongoDB Installation & Native Node.js Driver

**From Zero to Fullstack · Chapter 2: Backend Fundamentals**  
**Audience:** Absolute beginners and developers who need a reusable MongoDB setup  
**Primary platform:** Windows 10/11 · PowerShell · VS Code  
**Documentation version:** 1.0 · 15 October 2026  
**Source project:** [Chapter 2 — Backend Fundamentals](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install MongoDB Community Server, verify it with `mongosh`, reproduce the Chapter 2 native-driver HTTP example, and build a separate, safe CRUD exercise. No Express or Mongoose is required.

## Quick reference — save this for your next project

| Task | Tool / command |
| --- | --- |
| Check Node.js | `node --version` |
| Check installed MongoDB Shell | `mongosh --version` |
| Check Windows MongoDB service | `Get-Service MongoDB` (PowerShell) |
| Test the default MongoDB port | `Test-NetConnection 127.0.0.1 -Port 27017` |
| Connect to default local MongoDB | `mongosh "mongodb://127.0.0.1:27017"` |
| Connect to Chapter 2's port | `mongosh "mongodb://127.0.0.1:27018"` |
| Test MongoDB from mongosh | `db.adminCommand({ ping: 1 })` |
| Install the Node.js native driver | `npm install mongodb@^6` (new lab) |
| Inspect a collection with JavaScript | `await collection.find({}).toArray()` |
| Start the original Chapter 2 server | `node backend/mongodbweb.js` |

**Critical distinction:** Our published Chapter 2 source connects to **`mongodb://localhost:27018/`**. A typical fresh MongoDB Community installation listens on **27017**. The original repository does not provide a MongoDB service, Docker Compose configuration, or instructions creating port 27018. You must either make a **local practice adjustment** to 27017 or actually start a MongoDB instance on 27018. See Section 5.

## 1. What are MongoDB and the native driver?

A **database** stores information between program runs. **MongoDB** is a document database: information is stored in **BSON documents** resembling JavaScript objects. A **collection** groups documents; a **database** contains collections. The **MongoDB server** (`mongod`) stores the data and accepts connections. The **MongoDB Shell** (`mongosh`) is a separate terminal application for querying that server. **MongoDB Compass** is an optional graphical interface. The **MongoDB Node.js driver** (`mongodb`) lets a Node.js program talk to the server.

```text
Browser → HTTP request → Node.js server (createServer)
                              |
                              v
                  MongoClient (mongodb package)
                              |
                       TCP connection
                              |
                   MongoDB Community Server
                              |
                         ch2 database
                              |
                       users collection
```

**Key vocabulary:**

- **Document:** A record such as `{ name: 'Awa', email: 'awa@example.test' }`.
- **Collection:** A group of documents, e.g. `users`.
- **Database:** A container for related collections, e.g. `ch2`.
- **BSON / ObjectId:** MongoDB's binary document format and its common automatically generated `_id` type.
- **URI:** An address that tells a client which database server to contact.
- **CRUD:** Create, Read, Update, Delete.
- **Cursor:** A query result that is iterated or converted to an array with `toArray()`.
- **Driver:** A client library connecting your application to a database. This chapter uses the **native driver**, not Mongoose.

## 2. Prerequisites and Windows system requirements

You need Windows 10/11 on supported 64-bit hardware, an internet connection for downloads, permission to install programs, and enough free disk space for the server and its data. We recommend at least 8 GB RAM for comfortable work, although the specific supported OS/CPU requirements depend on your chosen MongoDB Server release.

Install **Git**, **Node.js LTS**, and **VS Code** using [Cheatsheet 01](../../chapter-01-foundations/01-developer-environment/README.md). Then check:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

Expected: each command prints a version, not a “command not found” error. `npm.cmd` is the Windows launcher to use when a PowerShell execution policy blocks `npm.ps1`. MongoDB installation is separate from `npm install`; installing the `mongodb` npm package does **not** install a database server.

## 3. Install MongoDB Community Server on Windows

1. Open the official [MongoDB Community Server Download Center](https://www.mongodb.com/try/download/community).
2. Choose a **supported release** for your Windows edition, **Windows**, and **MSI**. The exact version number and download buttons may change.
3. Download and run the installer, following its wizard.
4. For a beginner-friendly local setup, choose **Complete** and **Install MongoD as a Service**, if offered. Accept appropriate data/log directories and remember the service name (normally `MongoDB`).
5. **MongoDB Compass** is optional; select it if you want a graphical browser for databases.
6. Finish installation. A service installation normally starts the server automatically.

In a new **PowerShell** window, check:

```powershell
Get-Service MongoDB
Test-NetConnection 127.0.0.1 -Port 27017
```

Expected: `MongoDB` shows **Running** and `TcpTestSucceeded : True` on port **27017**, provided your installer used the normal default. If the service name or port was customized, substitute those values. If no service was installed, see the official instructions for starting `mongod` manually; installing only server binaries does not start a server.

**Service management** (requires an elevated terminal for changes):

```powershell
# Inspect as a normal user
Get-Service MongoDB
# Run PowerShell as Administrator for these operations:
Start-Service MongoDB
Stop-Service MongoDB
```

Do **not** stop a running MongoDB service that you or another application currently need. Do not delete its data directories.

## 4. Install MongoDB Shell and check the server

**Important:** MongoDB Server's Windows MSI does **not** bundle the `mongosh` shell; install it separately.

1. Open the official [MongoDB Shell installation page](https://www.mongodb.com/docs/mongodb-shell/install/).
2. Download the Windows 64-bit MSI and install it. Include `mongosh` in your PATH if the installer offers the option.
3. Open a **new** PowerShell window:

```powershell
mongosh --version
mongosh "mongodb://127.0.0.1:27017"
```

Inside the **mongosh prompt**, enter:

```javascript
db.adminCommand({ ping: 1 })
show dbs
use ch2
show collections
```

Expected: the ping returns an object containing `ok: 1`. Initially, `ch2` might not appear in `show dbs` because MongoDB generally materializes a database once it contains data. Exit the shell using `.exit`.

If `mongosh` is not recognized, restart the terminal and inspect its installation/PATH. If the shell starts but cannot connect, verify the **server service** and its **port**: installing `mongosh` alone does not start MongoDB.

**Optional GUI:** MongoDB Compass can connect to `mongodb://127.0.0.1:27017`, browse `ch2`, and inspect collections. Compass is not required for our code or exercises.

## 5. Port mismatch: 27017 vs 27018 (essential!)

The published source **literally uses**:

```javascript
const url = 'mongodb://localhost:27018/'
```

However, installing MongoDB as a default Windows service normally listens on `127.0.0.1:27017`. Do **not** confuse the Node.js HTTP port **3000** with the database port **27017/27018**.

**Option A — Recommended for a new learner:** Keep the default MongoDB Windows service on **27017**. Clone the Chapter 2 repository and **change only your own local copy** of `backend/mongodbweb.js` to:

```javascript
const url = 'mongodb://127.0.0.1:27017/'
```

This is a **local adaptation**, not the code committed to the original repository. Keep a note of your edit and run `git diff` before committing anything.

**Option B — Reproduce the original 27018 port:** Start a **separate local MongoDB instance** on port 27018 with its own distinct data directory. If you have the `mongod` executable on PATH:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\mongo-ch2-data"
mongod --dbpath "$HOME\mongo-ch2-data" --port 27018 --bind_ip 127.0.0.1
```

The `mongod` terminal must stay open while you test. If `mongod` is not on PATH, use the actual `mongod.exe` from your MongoDB Server installation directory instead of the bare `mongod` command. This option **does not modify** the default Windows service on port 27017. From a **second** terminal:

```powershell
Test-NetConnection 127.0.0.1 -Port 27018
mongosh "mongodb://127.0.0.1:27018"
```

Expected: `TcpTestSucceeded : True`, then a working `mongosh` connection. Stop the manually started lab process with **Ctrl + C** when done. Do not run two MongoDB processes against the **same** data directory.

**Security rule:** Use `127.0.0.1` for this no-authentication learning setup. **Never expose** an unauthenticated development database on a public network.

## 6. Clone and run our real Chapter 2 code

Clone the exact Chapter 2 repository, unless you already have it:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals.git
Set-Location from-zero-to-fullstack-ch2-backend-fundamentals
npm.cmd ci
```

This is a **mixed React/Vite + beginner backend** repository. The MongoDB driver appears under the root `package.json` dependency `"mongodb": "^6.3.0"`; its `"type": "module"` allows ES module `import` syntax. The root does **not** define an `npm run backend` script, so use `node backend/mongodbweb.js` directly.

**Published code in `backend/mongodbweb.js` (unchanged):**

```javascript
import { createServer } from 'http'
import { MongoClient } from 'mongodb'

const url = 'mongodb://localhost:27018/'
const dbName = 'ch2'
const client = new MongoClient(url)

try {
  await client.connect()
  console.log('Successfully connected to database!')
} catch (err) {
  console.error('Error connecting to database:', err)
}

const server = createServer(async (req, res) => {
  const db = client.db(dbName)
  const users = db.collection('users')
  const usersList = await users.find().toArray()
  res.statusCode = 200
  res.setHeader('Content-Type', 'application/json')
  res.end(JSON.stringify(usersList))
})

const host = 'localhost'
const port = 3000
server.listen(port, host, () => {
  console.log(`Server listening on http://${host}:${port}`)
})
```

**How it works:** `MongoClient` connects to the database server → `db('ch2')` selects a database → `.collection('users')` selects a collection → `.find().toArray()` reads the documents → `JSON.stringify` converts results into an HTTP response → `createServer()` serves JSON on **port 3000**.

**Original-code limitations (not already fixed in Chapter 2):**

- It catches initial connection errors but **still starts** the HTTP server even when MongoDB is unavailable.
- It does **not** catch errors from the asynchronous HTTP request handler.
- It does **not** restrict HTTP paths or methods; every request tries to list users.
- It does **not** close its MongoDB connection during a graceful shutdown.
- It has no `.env`-based URI configuration or production-ready authentication setup.

These omissions are educational opportunities; they do not imply that the original code contains features that it does not.

## 7. Seed sample data and verify the Chapter 2 HTTP endpoint

Open `mongosh` **using the same database port that your Node.js source uses**:

```powershell
# If using default service + local code adjustment:
mongosh "mongodb://127.0.0.1:27017"
# Or, if reproducing the original 27018 source:
mongosh "mongodb://127.0.0.1:27018"
```

At the `mongosh` prompt, enter:

```javascript
use ch2
db.users.insertOne({
  name: 'Awa Learning',
  email: 'awa@example.test'
})
db.users.find().toArray()
```

**Expected:** The collection contains the inserted document and an automatically generated `_id`. These are **practice-only fictional details**. If you have already inserted the sample, you can skip inserting it again to avoid duplicates.

Now, **back in PowerShell** at the Chapter 2 project root:

```powershell
node backend/mongodbweb.js
```

Expected console messages when the database is available:

```text
Successfully connected to database!
Server listening on http://localhost:3000
```

From a **second** PowerShell terminal:

```powershell
curl.exe -i http://localhost:3000/
```

**Expected:** HTTP **200**, `Content-Type: application/json`, and a JSON array containing the sample user. The generated MongoDB `_id` values will differ from machine to machine. Stop Node.js with **Ctrl + C**.

If you get an **empty array `[]`**, the database connection works but the selected collection may be empty, or you inserted data into the **other port/database**. Double-check the server URI, database `ch2`, and collection `users`.

## 8. Build an independent MongoDB CRUD lab from scratch

This is a **new reusable teaching example**, **not** a claim about the original Chapter 2 code. It uses a separate database named `ch2_cheatsheet_lab` so it does not modify the original `ch2.users` data.

### 8.1 Create the lab

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects\mongodb-native-lab"
Set-Location "$HOME\Projects\mongodb-native-lab"
npm.cmd init -y
npm.cmd pkg set type=module
npm.cmd install 'mongodb@^6'
```

Create a new file named **`crud.mjs`** in the lab folder:

```javascript
import { MongoClient } from 'mongodb'

const uri = process.env.MONGODB_URI ?? 'mongodb://127.0.0.1:27017'
const client = new MongoClient(uri, { serverSelectionTimeoutMS: 5000 })

try {
  await client.connect()
  await client.db('admin').command({ ping: 1 })

  const users = client.db('ch2_cheatsheet_lab').collection('users')

  const email = 'learner@example.test'
  const existing = await users.findOne({ email })
  if (existing) {
    console.log('Lab user already exists; reusing it')
  } else {
    const result = await users.insertOne({ name: 'Learner', email, active: true })
    console.log('CREATE:', result.insertedId.toString())
  }

  const found = await users.findOne({ email })
  console.log('READ:', found?.name, found?.email)

  const update = await users.updateOne({ email }, { $set: { active: false } })
  console.log('UPDATE:', update.matchedCount, update.modifiedCount)

  const all = await users.find({ email }).sort({ name: 1 }).toArray()
  console.log('LIST:', all.length, all[0]?.active)

  const removed = await users.deleteOne({ email })
  console.log('DELETE:', removed.deletedCount)
} catch (error) {
  console.error('MongoDB lab failed:', error.message)
  process.exitCode = 1
} finally {
  await client.close()
}
```

Run:

```powershell
node --check .\crud.mjs
node .\crud.mjs
```

**Expected (database running on 27017):** `CREATE` with an ObjectId, `READ: Learner learner@example.test`, `UPDATE: 1 1`, `LIST: 1 false`, then `DELETE: 1`. Because the example deletes its own lab record, it can generally be run again. If a previous run left it present, the code reuses that record and the exact output differs.

To use a running instance on **27018**, set the URI **for this terminal session only**:

```powershell
$env:MONGODB_URI = 'mongodb://127.0.0.1:27018'
node .\crud.mjs
Remove-Item Env:MONGODB_URI
```

**Why this design?** `try/catch/finally` handles connection failures and ensures a closed client. The URI is configurable without hard-coding private credentials. A single `MongoClient` instance performs the operations. Filters specify the exact lab user so the example never calls `deleteMany({})`.

### 8.2 CRUD method cheat table

| Intent | Native driver example | Returned result |
| --- | --- | --- |
| Create one | `insertOne({ name: 'Awa' })` | `insertedId` |
| Read one | `findOne({ name: 'Awa' })` | Document or `null` |
| Read many | `find({ active: true }).toArray()` | Array of documents |
| Update one | `updateOne({ name: 'Awa' }, { $set: { active: false } })` | Counts: `matchedCount`, `modifiedCount` |
| Delete one | `deleteOne({ name: 'Awa' })` | `deletedCount` |

**Safety:** `find()` returns a cursor; use `await` with `toArray()` only when the expected data volume fits memory. For large datasets, iterate with a cursor or use pagination. Always filter potentially destructive operations deliberately.

## 9. Next step: a safer HTTP integration (optional)

In the original code, request-handler errors are not caught. In a fresh project, use `try/catch` around database operations and validate the requested path/method; return a **500** JSON response on unexpected database errors and **404** on unsupported routes. Validate API inputs before calling MongoDB. Start accepting HTTP requests only **after** successful connection to the database.

Example request-handler pattern (illustrative, **not** the Chapter 2 source):

```javascript
const server = createServer(async (req, res) => {
  if (req.method !== 'GET' || req.url !== '/users') {
    res.writeHead(404, { 'Content-Type': 'application/json' })
    return res.end(JSON.stringify({ error: 'Not found' }))
  }
  try {
    const rows = await client.db('ch2').collection('users').find({}).toArray()
    res.writeHead(200, { 'Content-Type': 'application/json' })
    res.end(JSON.stringify(rows))
  } catch (error) {
    console.error('Database request failed', error)
    res.writeHead(500, { 'Content-Type': 'application/json' })
    res.end(JSON.stringify({ error: 'Internal server error' }))
  }
})
```

This is a **fragment** assuming `createServer` is imported and `client` has been connected; it is **not** a complete standalone program. We'll learn scalable route/service architecture with **Express** and **Mongoose** in Chapter 3.

## 10. Common errors and fixes

| Symptom | Likely cause | What to verify |
| --- | --- | --- |
| `mongosh` is not recognized | Shell not installed or not in PATH | Install shell separately; reopen PowerShell |
| `Get-Service MongoDB` not found | No service installed / custom name | Use `services.msc` and installer settings |
| `ECONNREFUSED 127.0.0.1:27018` | No database on Chapter 2's port | Change local URI to 27017 **or** start dedicated 27018 instance |
| `ECONNREFUSED 127.0.0.1:27017` | Default service stopped / different port | `Get-Service`, `Test-NetConnection`, `mongosh` |
| `Cannot find package 'mongodb'` | npm dependencies absent or wrong folder | Enter project root and run `npm.cmd ci`; new lab: `npm.cmd install mongodb@^6` |
| `Cannot use import statement outside a module` | ESM not configured | Use `.mjs` or set `"type": "module"` |
| `[]` from HTTP endpoint | Empty or wrong collection / port | Check `ch2`, `users`, and the selected MongoDB port |
| HTTP port `3000` is in use | Another server is listening | Stop it or use another HTTP port in your local copy |
| `MongoServerSelectionError` | Server unreachable, DNS/URI or auth issue | Check URI, port, server logs and permissions |
| Authentication failure | User credentials/authSource mismatch | Use valid least-privilege database credentials; never share them |
| `ETIMEDOUT` on Atlas | Network access or firewall issue | Validate Atlas network access and connection details |
| `mongod` cannot open data path | Missing folder or Windows filesystem rights | Use a writable, dedicated lab data directory |

### Diagnostic sequence (PowerShell)

```powershell
node --version
npm.cmd --version
Get-Service MongoDB
Test-NetConnection 127.0.0.1 -Port 27017
Test-NetConnection 127.0.0.1 -Port 27018
mongosh "mongodb://127.0.0.1:27017"
# In mongosh: db.adminCommand({ ping: 1 })
```

Check the **first error**, not only the last stack-trace line. Never paste credentials, real connection URIs, or private data into public issues or screenshots.

## 11. Security and production best practices

1. **Local-only by default.** This guide's no-authentication MongoDB examples are for `127.0.0.1` development, **not production**.
2. **Authentication and authorization.** Enable MongoDB authentication and use an application account with only necessary permissions in real deployments.
3. **Do not expose port 27017/27018 publicly.** Use secure networking and firewall rules.
4. **Secrets.** Store cloud URIs and passwords outside source control, in environment variables or a secrets manager. Do not commit `.env` with credentials; publish sanitized `.env.example` instead.
5. **Query safety.** Validate incoming input and avoid exposing raw arbitrary MongoDB operators directly to clients.
6. **Correct lifecycle.** Reuse a single `MongoClient` connection pool per application rather than opening a fresh client for every request. Close clients during shutdown or at the end of short scripts.
7. **Bounded results.** Use `limit()` or pagination instead of unbounded `toArray()` for large data.
8. **Data separation.** Use a dedicated test database and never use production records for a destructive lab exercise.
9. **Backups.** A lab is disposable; real data requires backups, recovery testing, and appropriate operational monitoring.
10. **Compatibility.** Validate Node.js, MongoDB Server, and driver versions against the current official compatibility matrix before deploying.

### Local installation vs MongoDB Atlas

This guide uses **MongoDB Community Server locally**. Atlas is MongoDB's managed cloud offering and requires a cluster, appropriate authentication, network access rules, and a cloud connection string. **Atlas setup is an optional expansion**, not part of the original Chapter 2 repository's configuration. Reuse the same `MongoClient(uri)` pattern with a securely supplied Atlas URI when you reach a cloud deployment chapter.

## 12. Reuse checklist

- [ ] Windows environment has Node.js, npm and Git.
- [ ] MongoDB Community Server installed and running locally.
- [ ] MongoDB Shell installed separately and opens normally.
- [ ] `db.adminCommand({ ping: 1 })` confirms server health.
- [ ] I can explain server vs shell vs native driver vs Compass.
- [ ] I verified **27017 vs 27018** before running Chapter 2.
- [ ] I cloned and installed the original Chapter 2 dependencies.
- [ ] `node backend/mongodbweb.js` connects using the correct local URI.
- [ ] `curl.exe` returns the test user array from the real source server.
- [ ] My independent CRUD lab safely creates, reads, updates and deletes its own test record.
- [ ] I used `try/catch/finally`, a reusable client and non-destructive filters.
- [ ] No credentials or real customer records are included in the documentation or repository.

## 13. Official references and verification boundaries

**Verified Chapter 2 source files (read from GitHub, 15 October 2026):**

- [Chapter 2 source repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals)
- [`backend/mongodbweb.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/mongodbweb.js) — original native-driver HTTP server and port 27018
- [`backend/files.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/files.js) — earlier JSON-file HTTP server
- [`package.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/package.json) — module type and MongoDB driver dependency

**Authoritative external documentation:**

- [MongoDB Community Server installation for Windows](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-windows/)
- [Run MongoDB Server as a Windows service](https://www.mongodb.com/docs/manual/administration/run-mongodb-windows/)
- [Install MongoDB Shell (`mongosh`)](https://www.mongodb.com/docs/mongodb-shell/install/)
- [MongoDB Node.js Driver v6.x](https://www.mongodb.com/docs/drivers/node/v6.x/)
- [Create a MongoClient](https://www.mongodb.com/docs/drivers/node/v6.x/connect/mongoclient/)
- [CRUD with the Node.js driver](https://www.mongodb.com/docs/drivers/node/v6.x/crud/)

**Verification statement:** The Chapter 2 implementation, port, package dependency and file paths were checked against the published GitHub source. The lab code is original instructional material. Code syntax can be checked using Node.js, but a **live MongoDB database execution is required** to validate the connection and CRUD outputs on the learner's machine; this document does not claim to have run the entire Windows installation or live database lab.

---

**Next:** Cheatsheet 09 — Express.js REST API from Scratch (Chapter 3).  
[Chapter 2 index](../README.md) · [Full cheatsheet roadmap](../../README.md)