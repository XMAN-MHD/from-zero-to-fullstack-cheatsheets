# Cheatsheet 06 — Node.js Fundamentals & Native HTTP Server

**From Zero to Fullstack · Chapter 2: Backend Fundamentals**  
**Audience:** Absolute beginners, React developers starting backend work, and developers needing a reusable native Node.js reference  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Version:** 1.0 · 13 October 2026  
**Project source:** [Chapter 2 — Backend Fundamentals](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install and verify Node.js; understand how HTTP requests and responses work; run the real Chapter 2 HTTP server; build an independent native Node.js HTTP server; and diagnose the most common errors.

## Quick reference — the smallest HTTP server

```js
// server.mjs — independent beginner example; NOT copied from Chapter 2
import { createServer } from 'node:http'

const server = createServer((req, res) => {
  res.statusCode = 200
  res.setHeader('Content-Type', 'application/json; charset=utf-8')
  res.end(JSON.stringify({ message: 'Hello from Node.js!' }))
})

server.listen(3000, '127.0.0.1', () => {
  console.log('Open http://127.0.0.1:3000')
})
```

Run with `node server.mjs` and visit `http://127.0.0.1:3000` in a browser. No npm packages are needed: **`node:http` is built into Node.js**.

| What | Command or concept |
| --- | --- |
| Check installation | `node --version` / `npm.cmd --version` |
| Start a Node program | `node server.mjs` or `node backend/files.js` |
| Run Chapter 2 example | `node backend/files.js` **from repository root** |
| Verify HTTP status and headers | `curl.exe -i http://localhost:3000/` |
| Read JSON in PowerShell | `Invoke-RestMethod http://localhost:3000/` |
| Stop the server | `Ctrl + C` in its terminal |
| Main Node API | `createServer((req, res) => { ... })` |
| Send a response | `res.statusCode`, `res.setHeader()`, `res.end()` |

## 1. What are Node.js, HTTP, and a server?

**Node.js** is a JavaScript runtime used outside the browser. Instead of running a React component, it can execute programs that accept HTTP requests, read local files, connect to databases, and perform backend tasks.

**HTTP** is a request-response protocol. A **client** (browser, frontend application, `curl.exe`) sends a request to a **server**. The server replies with a status code, optional headers, and an optional body.

```text
 Browser / React / curl.exe
           |
           |  HTTP request: GET /users
           v
    Node.js HTTP server
       createServer()
           |
           |  status: 200 OK
           |  Content-Type: application/json
           |  body: [{"name":"Adam Ondra", ...}]
           v
 Browser / React / curl.exe
```

**Key terms:**

- **Runtime**: software that executes your JavaScript program.
- **Server**: a process listening for connections on an address and port.
- **Host**: network address, such as `localhost` / `127.0.0.1`.
- **Port**: numbered entry point, such as `3000`.
- **Request** (`req`): the incoming HTTP message, including `method`, `url`, and headers.
- **Response** (`res`): the outgoing status, headers, and body.
- **JSON**: text format used to exchange structured data.
- **ES module**: JavaScript file using `import` / `export`.

**Important distinction:** Node.js is **not** Express. Our Chapter 2 examples use the built-in HTTP module directly; Express will be introduced in Chapter 3.

## 2. Prerequisites and installation from scratch

**Required:** Windows 10/11, internet connection, ability to open PowerShell, and a text editor. **Recommended:** 8 GB RAM and enough storage for Node.js, VS Code, and the project. A GitHub account is optional when only downloading or running code.

### 2.1 Install VS Code (recommended)

1. Download the Windows installer from <https://code.visualstudio.com/download>.
2. Follow its installer prompts. Open VS Code.
3. Restart the terminal and verify:

```powershell
code --version
```

If `code` is not recognized, reopen PowerShell or check VS Code's Windows PATH setup. You can always open VS Code from the Start menu and choose **File → Open Folder**.

### 2.2 Install Node.js LTS (required)

1. Download a **supported LTS** release from <https://nodejs.org/en/download>.
2. Choose the Windows installer suitable for your computer, usually x64.
3. Accept standard installation options. npm is normally included.
4. Close and reopen PowerShell; check:

```powershell
node --version
npm.cmd --version
node -e "console.log('JavaScript works:', 2 + 2)"
```

**Expected:** version strings followed by `JavaScript works: 4`. The exercises use modern ES modules, supported by current LTS versions. **Node.js 22 LTS or 24 LTS** is an appropriate baseline at the time this document was prepared; prefer a currently supported LTS if those lifecycle details change.

**Windows PowerShell note:** If `npm` tries to run `npm.ps1` and reports that script execution is disabled, use **`npm.cmd`**, for example `npm.cmd --version`. Do **not** change system-wide execution policy for this guide. Running the built-in HTTP example requires `node`, not npm.

### 2.3 Install Git (for cloning the project)

1. Install Git for Windows from <https://git-scm.com/install/windows>.
2. Restart PowerShell and verify:

```powershell
git --version
```

Git is needed to **clone** the original Chapter 2 repository. You can also use GitHub's **Code → Download ZIP** option instead; unzip it and open that folder.

## 3. Understand the original Chapter 2 project

We reviewed the following files on the default branch of the actual source repository:

```text
from-zero-to-fullstack-ch2-backend-fundamentals/
├── backend/
│   ├── files.js       # Native HTTP server reading a local JSON file
│   ├── mongodbweb.js  # Separate native HTTP server reading MongoDB
│   └── users.json     # Example data
├── package.json       # Existing Vite/React starter and npm dependencies
├── package-lock.json
├── src/                # Existing React/Vite starter
└── README.md           # Starter-derived README
```

### What the project actually does

- `backend/files.js` calls **`createServer()`**, reads **`backend/users.json`** using **`readFileSync()`**, and returns its content as JSON.
- `backend/mongodbweb.js` uses the **native MongoDB driver** and a server that reads a `users` collection. It expects a MongoDB instance at **`mongodb://localhost:27018/`**; this is covered in **Cheatsheet 08**, not required here.
- The repository contains a **React/Vite starter**, but **it does not define dedicated npm scripts for either backend server**. Use `node backend/files.js`, not `npm run backend`.
- The current top-level `package.json` declares **`"type": "module"`**, enabling ESM `import` syntax in `.js` files inside this package.
- `backend/files.js` responds with **200 and JSON for every incoming request**, without inspecting route or method. That is the exercise's actual limitation, not complete REST routing.

The sample data in `backend/users.json` is:

```json
[{"name":"Adam Ondra","email":"adam.ondra@climb.ing"}]
```

## 4. Run the actual Chapter 2 HTTP server

### 4.1 Clone the repository

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals.git
Set-Location from-zero-to-fullstack-ch2-backend-fundamentals
code .
```

**Expected:** VS Code shows `backend/files.js`, `backend/mongodbweb.js`, `backend/users.json`, and a root `package.json`.

### 4.2 Check your working directory and Node version

```powershell
Get-Location
node --version
Get-Content backend/users.json
```

**Expected:** you are in `from-zero-to-fullstack-ch2-backend-fundamentals`, and the JSON file displays one example user.

**No `npm install` is required solely to run `backend/files.js`** because `http` and `fs` are built-in Node.js modules. You may run `npm.cmd ci` if you also wish to use the repository's existing React/Vite tooling. The MongoDB example, however, uses an external npm dependency and a running MongoDB service.

### 4.3 Start the server

```powershell
node backend/files.js
```

**Expected terminal output:**

```text
Server listening on http://localhost:3000
```

Keep this first terminal running. In a browser, open <http://localhost:3000/>. The browser should show the JSON array from `users.json`.

### 4.4 Test in a second PowerShell terminal

```powershell
curl.exe -i http://localhost:3000/
Invoke-RestMethod http://localhost:3000/
```

**Expected:** `HTTP/1.1 200 OK`, a `Content-Type: application/json` header, and the sample JSON. `Invoke-RestMethod` may present the parsed content as a PowerShell object instead of raw JSON text.

**PowerShell pitfall:** `curl` can be an alias in some PowerShell versions. Use **`curl.exe`** to call the native command-line HTTP client consistently.

### 4.5 Stop cleanly

Focus the terminal running Node and press **Ctrl + C**. Once stopped, requests to port `3000` should fail until you restart the server.

## 5. Read and understand our original server line by line

This is the **active server logic** from `backend/files.js`, with the original comments and spacing omitted but its behavior unchanged:

```js
import { createServer } from 'http'
import { readFileSync } from 'fs'

const server = createServer((req, res) => {
  res.statusCode = 200
  res.setHeader('Content-Type', 'application/json')
  res.end(readFileSync('backend/users.json'))
})

const host = 'localhost'
const port = 3000
server.listen(port, host, () => {
  console.log(`Server listening on http://${host}:${port}`)
})
```

| Line / function | Beginner explanation |
| --- | --- |
| `import ... from 'http'` | Load a Node.js built-in module (native HTTP API). |
| `import ... from 'fs'` | Load the built-in file system module. |
| `createServer((req, res) => {})` | Run a callback when an HTTP request arrives. |
| `req` | Incoming request; **present but not used** in this example. |
| `res.statusCode = 200` | Mark the response as successful. |
| `res.setHeader(...)` | Tell clients that the body contains JSON. |
| `readFileSync('backend/users.json')` | Read the file **synchronously**; this relative path depends on the process's current working directory. |
| `res.end(...)` | Write the body and **complete** the response. |
| `server.listen(3000, 'localhost', ...)` | Start listening locally on port 3000. |

### Why does the server return JSON?

The repository's `users.json` already contains a **JSON string**. `readFileSync()` returns a **Buffer** by default, which `res.end()` can send as response bytes. You therefore do **not** need to call `JSON.stringify()` a second time on those bytes.

When building a JavaScript object in memory, the situation is different:

```js
const result = { message: 'Hello' }
res.end(JSON.stringify(result)) // Convert the object to JSON text
```

### Current behavior to be aware of

Because the source server does not check `req.method` or `req.url`, `/`, `/users`, and any other URL will return the same content. It is an **introductory example**, not a fully validated production API. Reading a file on every request using `readFileSync()` also blocks the Node.js event loop during that I/O operation. We improve both behaviors in Section 7.

## 6. HTTP concepts you will reuse in later projects

### 6.1 Request methods

| Method | Typical intention |
| --- | --- |
| `GET` | Retrieve a resource. |
| `POST` | Submit data / create a resource. |
| `PUT` | Replace an entire resource. |
| `PATCH` | Partially modify a resource. |
| `DELETE` | Remove a resource. |

**Important:** these are HTTP conventions; using a method does not automatically make the Node handler implement its intended operation.

### 6.2 Response status codes

| Code | Meaning | Typical use |
| --- | --- | --- |
| `200` | OK | Successful read. |
| `201` | Created | New resource created. |
| `204` | No Content | Successful operation without body. |
| `400` | Bad Request | Invalid request. |
| `404` | Not Found | Resource path does not exist. |
| `405` | Method Not Allowed | Resource exists, but method is unsupported. |
| `500` | Internal Server Error | Unexpected server failure. |

### 6.3 Headers, body, and JSON

A successful exchange may look like this (header order and connection headers can vary):

```http
GET /users HTTP/1.1
Host: localhost:3000

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

[{"name":"Adam Ondra","email":"adam.ondra@climb.ing"}]
```

**`Content-Type`** helps the client interpret the response body. **`JSON.stringify()`** converts objects to JSON text. **`JSON.parse()`** converts valid JSON text to a JavaScript value. Chapter 2's dedicated File System cheatsheet will study file handling in greater depth.

### 6.4 Why `.js` imports work in this repository

The root `package.json` sets `"type": "module"`, so `backend/files.js` can use:

```js
import { createServer } from 'http'
```

In a standalone directory with **no** `package.json`, name your ESM entry file **`server.mjs`**, or create a `package.json` that declares `"type": "module"`. Modern Node.js may infer some module formats, but explicit configuration is clearer for beginners.

## 7. Reusable improvement — a server with a real route

**Educational improvement: the code below is NOT present in the Chapter 2 GitHub repository.** It introduces routing, method validation, asynchronous file I/O, and safe error handling. Try it in a separate practice folder, or save it as `backend/server-improved.js` within your **local** Chapter 2 clone.

```js
// backend/server-improved.js — optional practice code
import { createServer } from 'node:http'
import { readFile } from 'node:fs/promises'

const host = 'localhost'
const port = 3000
const dataFile = new URL('./users.json', import.meta.url)

const server = createServer(async (req, res) => {
  res.setHeader('Content-Type', 'application/json; charset=utf-8')

  if (req.url !== '/users') {
    res.statusCode = 404
    res.end(JSON.stringify({ error: 'Route not found' }))
    return
  }

  if (req.method !== 'GET') {
    res.statusCode = 405
    res.setHeader('Allow', 'GET')
    res.end(JSON.stringify({ error: 'Method not allowed' }))
    return
  }

  try {
    const contents = await readFile(dataFile, 'utf8')
    const users = JSON.parse(contents)
    res.statusCode = 200
    res.end(JSON.stringify(users))
  } catch (error) {
    console.error('Unable to load users:', error)
    res.statusCode = 500
    res.end(JSON.stringify({ error: 'Internal server error' }))
  }
})

server.on('error', (error) => {
  console.error('Cannot start server:', error.message)
})

server.listen(port, host, () => {
  console.log(`Open http://${host}:${port}/users`)
})
```

**Why it is better for learning:**

1. **Routing:** only `/users` is a known path.
2. **Method check:** only `GET` is allowed on `/users`.
3. **Async I/O:** `readFile()` does not synchronously block the event loop while waiting for the file.
4. **Portable file location:** `new URL('./users.json', import.meta.url)` resolves relative to this module, instead of your terminal's working directory.
5. **Failure handling:** a missing or invalid file produces JSON with status `500`, without revealing private filesystem paths to the client.

**Run it (stop the original server first):**

```powershell
node backend/server-improved.js
curl.exe -i http://localhost:3000/users
curl.exe -i http://localhost:3000/unknown
curl.exe -i -X POST http://localhost:3000/users
```

Expected statuses: **200** for `GET /users`, **404** for `GET /unknown`, and **405** for `POST /users`. If you receive `EADDRINUSE`, stop the earlier server using port 3000.

**Boundary:** This still is not a full production server. It lacks authentication, request body parsing, logging, rate limiting, CORS policy, validation, tests, and graceful deployment lifecycle handling. Those concepts come later.

## 8. Verify your understanding — three quick exercises

### Exercise A — change the JSON fixture

Edit the **local clone's** `backend/users.json` to contain two sample users, then refresh <http://localhost:3000/> while running the **original** server. It should return the updated array without restarting because the file is read on each request. Do not put real personal information in the example.

### Exercise B — use request metadata

Inside the server's callback, temporarily add:

```js
console.log(req.method, req.url)
```

Restart the server, then visit `/users`, `/test`, or run `curl.exe -I http://localhost:3000/`. Observe the requested method and URL printed in the terminal. The **original response remains unchanged**, because logging `req` does not introduce routing.

### Exercise C — write an in-memory JSON response

Use the first `server.mjs` example. Change the object to:

```js
{ message: 'Hello from my server', chapter: 2 }
```

Refresh the browser and confirm both properties appear as JSON. Stop the server with **Ctrl + C**.

## 9. Troubleshooting and common mistakes

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| `node` is not recognized | Missing Node.js installation or stale PATH | Install LTS, restart PowerShell, verify with `where.exe node`. |
| `Cannot use import statement outside a module` | `.js` lacks ESM configuration | Run inside the cloned project with its `"type": "module"`, or use `.mjs`. |
| `ENOENT: no such file or directory` | Wrong current directory for relative `backend/users.json` | Run `Get-Location`; start `node backend/files.js` from repo root. |
| `EADDRINUSE: address already in use :::3000` | Another program is listening on port 3000 | Stop its terminal, or choose another port in your practice server. |
| `ERR_CONNECTION_REFUSED` | Server stopped or listening on another port | Start the server and use the exact host/port printed. |
| Unexpected HTML at `localhost:3000` | A different service occupies the port | Inspect which application owns the port; stop it before starting Node. |
| `npm run backend` fails | No backend script exists in Chapter 2 `package.json` | Use `node backend/files.js`. |
| `curl` output looks strange in PowerShell | Shell alias or display behavior | Use `curl.exe -i` or `Invoke-RestMethod`. |
| Mongo connection refused on port `27018` | MongoDB service not running/configured | The database example is a **different** exercise; see Cheatsheet 08. |
| File changes do not update the server code | Running Node processes do not reload JS automatically | Stop and restart the original server after editing JS code. |

**Useful diagnostics (PowerShell):**

```powershell
Get-Location
node --version
Get-Content backend/users.json
curl.exe -i http://localhost:3000/
Get-NetTCPConnection -LocalPort 3000 -ErrorAction SilentlyContinue
```

The final command lists local TCP connections on that port where permissions permit. If it returns nothing, the server is probably not listening there.

## 10. Security, performance, and best practices

- **Only bind to `localhost`** for private development. Binding to `0.0.0.0` exposes the process on available interfaces; do so only with appropriate firewall and network controls.
- **Validate routes and methods.** The source code is intentionally simple and returns `200` for everything. A reusable server should not assume every request is valid.
- **Do not expose arbitrary file paths.** The improved example reads a fixed data file, not a path supplied by the client.
- **Avoid `readFileSync()` per request** for larger or concurrent workloads; synchronous filesystem calls block the Node.js event loop. Use `node:fs/promises` or an appropriate data layer instead.
- **Do not reveal stack traces** or local paths in HTTP error responses. Log diagnostics on the server.
- **Handle startup errors** such as occupied ports. Plan request validation, authentication, observability, and testing before public deployment.
- **Separate responsibilities:** HTTP handlers transport requests; later chapters introduce routes, services, database models, and API validation.
- **Use built-in modules first** when learning the basics. You do not need Express to understand `createServer()`.

## 11. Reuse checklist for your next project

- [ ] Node.js LTS and VS Code installed; `node --version` works.
- [ ] Git installed; project downloaded or cloned safely.
- [ ] I can tell Node.js, npm, HTTP, Express, and MongoDB apart.
- [ ] I can explain `req`, `res`, host, port, status, header, and body.
- [ ] The original Chapter 2 server runs using `node backend/files.js` from its root.
- [ ] I can get JSON with a browser, `curl.exe`, and `Invoke-RestMethod`.
- [ ] I know why the original server returns the same content for every route.
- [ ] I know how to identify `ENOENT` and `EADDRINUSE` errors.
- [ ] I understand when to use `JSON.stringify()` and why `res.end()` matters.
- [ ] I can create my own `server.mjs` using only built-in Node.js functionality.
- [ ] My reusable code handles unsupported routes and methods appropriately.
- [ ] I avoid leaking data and exposing development services unnecessarily.

## 12. Source evidence and references

**Verified repository files, reviewed 13 October 2026:**

- [Chapter 2 source repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals)
- [backend/files.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/files.js) — native HTTP + local JSON file
- [backend/users.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/users.json) — demo data
- [backend/mongodbweb.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/mongodbweb.js) — separate MongoDB HTTP example
- [package.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/package.json) — ESM configuration, scripts, and package dependencies

**Official references:**

- [Node.js download and LTS](https://nodejs.org/en/download)
- [Node.js HTTP documentation](https://nodejs.org/api/http.html)
- [Node.js File System documentation](https://nodejs.org/api/fs.html)
- [Node.js ECMAScript modules](https://nodejs.org/api/esm.html)
- [VS Code Windows setup](https://code.visualstudio.com/docs/setup/windows)
- [Git for Windows](https://git-scm.com/install/windows)

**Evidence boundary:** Repository-specific details are drawn from its published code. The minimal `server.mjs`, improved `/users` example, exercises, troubleshooting advice, and security recommendations are additional pedagogical material; they are **not claimed to be committed features** of Chapter 2. The example commands have been structured to be reproducible on Windows, but successful execution on every Windows system is not assumed.

---

**Next guides:** Cheatsheet 07 — Node.js File System & JSON Handling; Cheatsheet 08 — MongoDB Installation & Native Node.js Driver.  
[← Chapter 2 index](../README.md) · [Full cheatsheet library](../../README.md)