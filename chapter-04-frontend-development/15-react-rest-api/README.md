# Cheatsheet 15 — Connecting React to a REST API

**From Zero to Fullstack · Chapter 4: Frontend Development**  
**Audience:** Absolute beginners learning React + API integration; returning developers needing a reusable workflow  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code · Node.js  
**Version:** 1.0 · 19 October 2026  
**Source project:** [Chapter 4 — Frontend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)  
**Method:** **Learn → Practice → Document → Reuse**

> **Accuracy note.** **SOURCE PROJECT** refers to code read directly from the Chapter 4 GitHub repository. **INDEPENDENT LAB** refers to the additional working demo included in this guide's ZIP. **IMPROVEMENT** is a recommendation, not a change already committed to the source code. The lab's native Node.js API passed **6 automated HTTP tests** during authoring. The actual Chapter 4 frontend and its MongoDB-backed application were **not run end-to-end** here.

## 1. Goals: what you will build and learn

After completing this guide, you should be able to:

1. Explain how a browser, React, HTTP, an API, and MongoDB relate to one another.
2. Install and verify Node.js, npm, Git and VS Code on Windows.
3. Start the Chapter 4 **frontend and backend from different folders and terminals**.
4. Configure `VITE_BACKEND_URL` with the **correct API prefix**.
5. Understand `fetch()`, `await`, query parameters, `response.ok` and JSON parsing.
6. Handle loading, error, empty and success states in React.
7. Distinguish local React state from server state; recognize where TanStack Query is already used.
8. Debug CORS, `404`, invalid URLs, connection failures and `204 No Content`.
9. Reproduce a working miniature React-to-API connection without MongoDB.
10. Reuse the same workflow in a new frontend/backend application.

### The three programs involved

```text
Browser / React frontend (Vite, usually localhost:5173)
   |
   | fetch('http://localhost:3000/api/v1/posts')
   | HTTP request + JSON response
   v
Node.js + Express API (localhost:3000)
   |
   | Mongoose queries
   v
MongoDB database (source default localhost:27018)
```

The React app **does not connect directly to MongoDB**. React sends HTTP requests to the backend, which owns the database connection. The **INDEPENDENT LAB** replaces MongoDB and Express with a small native-Node server storing temporary demo posts in memory, so you can learn the HTTP link first.

| Responsibility | Chapter 4 source file | What it does |
|:--|:--|:--|
| Start React | `src/main.jsx` | Mounts `<App />` inside React `StrictMode` |
| Provide server-state tools | `src/App.jsx` | Wraps `<Blog />` in `QueryClientProvider` |
| Read and display posts | `src/Blog.jsx` | Uses `useQuery` and passes data to `<PostList />` |
| Perform HTTP requests | `src/api/posts.js` | Uses `fetch`, `URLSearchParams`, and `VITE_BACKEND_URL` |
| Expose API endpoints | `backend/src/routes/posts.js` | Implements `/api/v1/posts` CRUD routes |
| Enable browser cross-origin access | `backend/src/app.js` | Configures `cors()` and `express.json()` |
| Connect to database | `backend/src/db/init.js` | Calls `mongoose.connect()` |

## 2. Prerequisites: install and verify from zero

Install these tools from their **official websites**:

- Node.js: <https://nodejs.org/en/download>
- Git: <https://git-scm.com/downloads>
- VS Code: <https://code.visualstudio.com/>
- MongoDB Community Server, for the **source project only**: <https://www.mongodb.com/try/download/community>
- MongoDB Shell: <https://www.mongodb.com/try/download/shell>

Use Node.js **20.19+ or 22.12+** for the Vite 7 toolchain used in Chapter 4; a supported recent LTS version is recommended. Open **PowerShell** in VS Code:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

If PowerShell blocks `npm.ps1`, use **`npm.cmd`** rather than changing machine-wide execution policies. Commands below use `npm.cmd` for consistency. If `code --version` fails, launch VS Code from the Start menu and open the folder using **File → Open Folder**.

### Clone the real source repository

```powershell
New-Item -ItemType Directory -Force "$HOME\projects" | Out-Null
Set-Location "$HOME\projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development.git
Set-Location .\from-zero-to-fullstack-ch4-frontend-development
Get-ChildItem
```

The **SOURCE PROJECT** has **two package.json files**:

```text
from-zero-to-fullstack-ch4-frontend-development/
├── package.json                   # React / Vite FRONTEND
├── src/
│   ├── App.jsx
│   ├── Blog.jsx
│   ├── api/posts.js
│   └── components/...
└── backend/
    ├── package.json               # Express / Mongoose BACKEND
    └── src/
        ├── app.js
        ├── index.js
        ├── routes/posts.js
        └── db/init.js
```

**Never run one `npm install` in the root expecting it to install both apps.** They are separate Node.js projects with separate dependencies and lockfiles.

## 3. Understand the API contract before touching React

**SOURCE PROJECT — `backend/src/routes/posts.js`** implements the following REST endpoints:

| HTTP method | Path | Purpose | Expected normal status |
|:--|:--|:--|:--|
| `GET` | `/api/v1/posts` | List, filter, sort posts | `200` JSON array |
| `GET` | `/api/v1/posts/:id` | Read one post | `200` or `404` |
| `POST` | `/api/v1/posts` | Create one post | `201` JSON object |
| `PATCH` | `/api/v1/posts/:id` | Modify some fields | `200` or `404` |
| `DELETE` | `/api/v1/posts/:id` | Delete one post | `204` (no JSON body) or `404` |

List endpoint options include `author`, `tags`, `sortBy`, and `sortOrder`. The backend deliberately rejects the combination of both `author` **and** `tags` (`400`). Source code uses **`tags` (plural)**, not `tag`.

Example JSON returned by the API (illustrative format, not a captured response):

```json
[
  {
    "_id": "507f1f77bcf86cd799439011",
    "title": "My first post",
    "author": "A beginner",
    "contents": "Hello from MongoDB",
    "tags": ["react", "api"]
  }
]
```

**Important:** An `_id` from MongoDB is generally a string in the HTTP JSON response. The example ID above is invented for instruction, not copied from real user data.

### Why the complete URL matters

The frontend source builds requests as:

```js
`${import.meta.env.VITE_BACKEND_URL}/posts`
```

Therefore configure the variable as **`http://localhost:3000/api/v1`**, not `http://localhost:3000` and not a URL already ending in `/posts`.

| `VITE_BACKEND_URL` | Resulting request | Outcome |
|:--|:--|:--|
| `http://localhost:3000/api/v1` | `/api/v1/posts` | Correct |
| `http://localhost:3000` | `/posts` | Wrong path (usually 404) |
| `http://localhost:3000/api/v1/posts` | `/api/v1/posts/posts` | Duplicated segment |
| missing | `undefined/posts` | Invalid or unexpected URL |

## 4. Run the original Chapter 4 backend

### Install backend dependencies separately

Open **Terminal A** in VS Code:

```powershell
Set-Location "$HOME\projects\from-zero-to-fullstack-ch4-frontend-development\backend"
npm.cmd ci
npm.cmd run
```

`npm ci` uses the committed `package-lock.json` to install reproducibly. The backend declares Express 5, `cors`, `mongoose`, `dotenv` and development tools.

### Prepare MongoDB and local configuration

**SOURCE PROJECT:** `backend/src/db/init.js` defaults to:

```js
const DATABASE_URL =
  process.env.DATABASE_URL || 'mongodb://localhost:27018/blog'
```

MongoDB's usual default local port is **27017**, not 27018. If your local MongoDB is installed on **27017**, create a **local-only** `backend/.env` containing:

```dotenv
DATABASE_URL=mongodb://127.0.0.1:27017/blog
PORT=3000
```

If you specifically configured MongoDB on port 27018, adapt the URL accordingly. Verify MongoDB separately with `mongosh` and the port actually in use. The source `backend/.gitignore` ignores `.env`; still check the Git status before any push.

Start the backend:

```powershell
npm.cmd run dev
```

Expected *when MongoDB is available*: `express server running on http://localhost:3000`. If MongoDB fails to connect, investigate the error rather than expecting the API to be ready.

In **Terminal B**, check the API before opening React:

```powershell
Invoke-RestMethod -Uri 'http://localhost:3000/api/v1/posts' -Method Get
```

A healthy empty database may legitimately return **`[]`**. If you want a test post, you may create one:

```powershell
$payload = @{ title = 'Hello API'; author = 'Learner'; contents = 'It works'; tags = @('react') } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://localhost:3000/api/v1/posts' -Method Post -ContentType 'application/json' -Body $payload
```

Only execute this on a database you are entitled to modify.

### CORS in the original backend

**SOURCE PROJECT — `backend/src/app.js`:**

```js
import cors from 'cors'
const app = express()
app.use(cors())
app.use(express.json())
```

Because frontend and backend have **different origins** (different ports), the browser enforces CORS. `cors()` with its default settings is **permissive**; it is convenient during local development but not a sufficient production-origin policy. A frontend origin restriction in production is an **IMPROVEMENT**:

```js
app.use(cors({ origin: 'https://your-frontend.example' }))
```

CORS is a **browser security policy**, not authorization. You still need proper authentication/authorization for protected routes.

## 5. Run the original React frontend

Open **Terminal C** in VS Code from the **repository root**:

```powershell
Set-Location "$HOME\projects\from-zero-to-fullstack-ch4-frontend-development"
npm.cmd ci
```

Create a **local** frontend `.env` (do **not** replace the repository's file without inspecting your local copy, and never paste secrets here):

```dotenv
VITE_BACKEND_URL=http://localhost:3000/api/v1
```

**Important security finding:** GitHub's tree for the original Chapter 4 currently includes a committed **root `.env`**. This guide intentionally does **not** read, reproduce, or redistribute its contents. Audit that file for secrets; rotate and remove any exposed secrets if found. Future `.env` files should be excluded from commits. Even when placed in an ignored file, **`VITE_` variables are bundled into client-side code** and must contain only public information such as a public API base URL.

The source frontend declares React `^19.1.1`, Vite `^7.1.7`, and `@tanstack/react-query` `^5.90.12` (declared ranges, not exact installed versions).

### Source-specific blockers to check before running

**SOURCE PROJECT — verified issues from the committed code:**

1. Several components import **`prop-types`**, but the root `package.json` does **not** declare it. In the working copy, install it via `npm.cmd install prop-types`.
2. The end of `src/components/DeletePost.jsx` refers to **`DeletePostButton.propTypes`**; however, the declared component is `DeletePost`. This can cause a `ReferenceError` during module evaluation. In your working copy, replace that line with **`DeletePost.propTypes = { ... }`**. A fix is being *recommended*, not asserted as already committed.
3. `src/components/PostSorting.jsx` declares a prop type for `field`, while the actual component expects `fields`. Correcting this mismatch is a useful quality improvement.

After applying any necessary fixes **locally**, start Vite:

```powershell
npm.cmd run dev
```

Open the address printed by Vite (usually `http://localhost:5173`). When running frontend and backend together, keep **both terminals open**.

**Note:** The root `.env` file may contain an existing URL different from the one above; use your local configuration intentionally and do not commit secrets. Restart Vite after changing `.env`.

## 6. Understanding `fetch()` through the real source code

**SOURCE PROJECT — `src/api/posts.js`** contains this `getPosts` implementation:

```js
export const getPosts = async (queryParams) => {
  const res = await fetch(
    `${import.meta.env.VITE_BACKEND_URL}/posts?` +
      new URLSearchParams(queryParams),
  )
  return await res.json()
}
```

Step by step:

1. `import.meta.env.VITE_BACKEND_URL` reads public frontend build configuration from Vite.
2. The `/posts` segment selects the API resource.
3. `URLSearchParams` encodes `author`, `tags`, `sortBy`, and `sortOrder` into a query string.
4. `fetch()` sends the HTTP request and returns a `Response` object.
5. `res.json()` **asynchronously parses** the response body and returns JavaScript data.

Example:

```js
new URLSearchParams({ author: 'Awa Ndiaye', sortBy: 'createdAt' }).toString()
// author=Awa+Ndiaye&sortBy=createdAt
```

### Why `response.ok` matters

A `404` or `500` HTTP response **does not automatically reject** the `fetch()` promise. Only network-level errors and some other failures cause a rejection. Always check `response.ok` for client-side API wrappers. The original `getPosts()` does **not** check it; the original `createPost`, `deletePost`, and `updatePost` functions **do**.

**IMPROVEMENT — more robust list request:**

```js
export async function getPosts({ author = '', tags = '', signal } = {}) {
  const base = import.meta.env.VITE_BACKEND_URL
  if (!base) throw new Error('VITE_BACKEND_URL is missing')
  const url = new URL(`${base}/posts`)
  if (author.trim()) url.searchParams.set('author', author.trim())
  if (tags.trim()) url.searchParams.set('tags', tags.trim())
  const response = await fetch(url, { signal })
  if (!response.ok) throw new Error(`HTTP ${response.status}: posts request failed`)
  return await response.json()
}
```

**Why this is different:** it checks configuration, filters out empty parameters, checks HTTP status and allows cancellation via `AbortSignal`. You must still handle server responses as **untrusted input**; client-side validation does not replace backend validation.

### JSON request for `POST` (already in source)

```js
const res = await fetch(`${import.meta.env.VITE_BACKEND_URL}/posts`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(post),
})
if (!res.ok) throw new Error('Failed to create post')
return await res.json()
```

The `Content-Type` tells the API the body is JSON, while `JSON.stringify()` serializes a JavaScript object into a string. On success, the source backend returns **`201 Created`** and a JSON document.

### `DELETE` and the empty-body rule

**SOURCE PROJECT:** the backend responds with **`204 No Content`** after a successful delete. The source frontend correctly returns `res` without trying to call `res.json()`:

```js
if (!res.ok) throw new Error('Failed to delete post')
return res
```

Do **not** call `response.json()` for a successful `204` response: there is no JSON body to parse.

## 7. React state: loading, errors and empty results

When your UI requests remote data, four states deserve attention:

| UI state | When it occurs | What the user should see |
|:--|:--|:--|
| Loading | Request is pending | Loading indicator |
| Error | Network / HTTP / parsing error | Clear retryable message |
| Empty | Request succeeded with no posts | “No posts yet” message |
| Success | Request succeeded with posts | Rendered posts list |

An **INDEPENDENT LAB** version without TanStack Query helps you see the mechanics:

```jsx
import { useEffect, useState } from 'react'
import { getPosts } from './api/posts.js'

export function App() {
  const [posts, setPosts] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState('')

  useEffect(() => {
    const controller = new AbortController()
    setLoading(true)
    getPosts({ signal: controller.signal })
      .then(setPosts)
      .catch(err => { if (err.name !== 'AbortError') setError(err.message) })
      .finally(() => { if (!controller.signal.aborted) setLoading(false) })
    return () => controller.abort()
  }, [])

  if (loading) return <p>Loading...</p>
  if (error) return <p role="alert">{error}</p>
  if (posts.length === 0) return <p>No posts yet.</p>
  return <ul>{posts.map(post => <li key={post._id}>{post.title}</li>)}</ul>
}
```

**Small caveat:** this minimal teaching snippet doesn't reset `error` before a retry. The complete downloadable lab includes a more complete refresh flow and form. With `React.StrictMode`, effects may be mounted, cleaned up and rerun during development to reveal unsafe assumptions; cancelling requests helps avoid stale updates.

## 8. How the *real* project uses TanStack Query

**SOURCE PROJECT — `src/App.jsx`:**

```jsx
const queryClient = new QueryClient()

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Blog />
    </QueryClientProvider>
  )
}
```

**SOURCE PROJECT — `src/Blog.jsx`:**

```jsx
const postsQuery = useQuery({
  queryKey: ['posts', { author: debouncedAuthor, tags, sortBy, sortOrder }],
  queryFn: () => getPosts({ author: debouncedAuthor, tags, sortBy, sortOrder }),
})
const posts = postsQuery.data ?? []
if (postsQuery.isLoading) return <div>Loading...</div>
if (postsQuery.error) return <div>Error: {postsQuery.error.message}</div>
```

`QueryClientProvider` makes React Query available to child components. `useQuery` manages the remote request and its cached result. A **query key** identifies the data: changing the author, tags or sorting options produces a different key. The `?? []` operator provides an empty-array fallback while data is not yet present.

The source also uses `useMutation` and `queryClient.invalidateQueries({ queryKey: ['posts'] })` in post creation, editing and deletion components to refresh outdated lists.

**Scope:** this cheatsheet concentrates on the **HTTP integration contract**. The detailed TanStack Query caching lifecycle, query keys, mutation invalidation and refetch behavior belong to **Cheatsheet 16 — TanStack Query from Scratch**.

## 9. INDEPENDENT LAB — working React ↔ API from scratch

The ZIP includes a small **separate** project. It deliberately uses native Node.js and an in-memory array instead of MongoDB so learners can test the HTTP connection immediately.

```text
lab/
├── README.md
├── backend/
│   ├── server.mjs                 # Node.js HTTP API, CORS, GET/POST
│   └── server.test.mjs            # Six Node.js HTTP tests
└── frontend/
    ├── package.json              # React, Vite and npm scripts
    ├── .env.example              # Public VITE_BACKEND_URL
    ├── .gitignore                # Ignores local .env
    ├── vite.config.js
    ├── index.html
    └── src/
        ├── main.jsx
        ├── App.jsx                # Loading, errors, create and filter
        ├── style.css
        └── api/posts.js           # getPosts() and createPost()
```

### Step A: start the independent API

Extract the ZIP somewhere such as `C:\Users\YourName\projects\cheatsheet-15\`. Open the lab folder in VS Code.

**Terminal 1** — execute from `lab/backend`:

```powershell
node .\server.mjs
```

Expected: `Lab API ready: http://localhost:3000/api/v1/posts`.

In another terminal, verify:

```powershell
Invoke-RestMethod 'http://localhost:3000/api/v1/health'
Invoke-RestMethod 'http://localhost:3000/api/v1/posts'
```

The demo starts with two fictional posts and **loses changes when restarted**; this is intentional. It does not need Express or MongoDB.

### Step B: launch the independent React frontend

**Terminal 2** — execute from `lab/frontend`:

```powershell
Copy-Item .env.example .env
npm.cmd install
npm.cmd run dev
```

Open the Vite address, usually **http://localhost:5173**. You should see a list of sample posts. Try creating a new post, filtering by author, and clicking **Reload**. If another program already uses port 5173, Vite may choose a different one; update the lab API's allowed origins or run Vite with `npm.cmd run dev -- --port 5173` after freeing the port.

**Why install instead of `npm ci` in the lab?** The independently packaged lab intentionally has no `package-lock.json` until you install dependencies. `npm.cmd install` creates it. In the actual Chapter 4 repos, use `npm.cmd ci` because lockfiles exist.

### Step C: run the automated API tests

From `lab/backend`:

```powershell
node --test .\server.test.mjs
```

During preparation of this guide, **6/6 HTTP tests passed** under Node.js. These verify list retrieval, author filtering, creation, client validation (`400`), CORS, and unknown-route (`404`) handling. They **do not verify** the original MongoDB application or the browser rendering of React; you should run those separately.

### Step D: verify the browser integration

Open Developer Tools (**F12 → Network → Fetch/XHR**):

1. Reload the page and select the request to `/api/v1/posts`.
2. Confirm the **request URL**, **HTTP status**, and **JSON response**.
3. Submit the form and verify a `POST` request with JSON body.
4. Confirm the `GET` request refreshes the list afterward.
5. Temporarily stop the API; reload and verify an error message appears.

The last step is a useful negative test: a frontend should not silently appear successful when its backend is unavailable.

## 10. Troubleshooting on Windows

| Symptom | Likely cause | Check / fix |
|:--|:--|:--|
| `npm.ps1 cannot be loaded` | PowerShell execution policy | Run `npm.cmd` |
| `npm ERR! ERESOLVE` / install fails | Unsupported Node or package dependency conflict | Check `node -v` and lockfile; avoid blindly using `--force` |
| Blank React page + console `DeletePostButton is not defined` | Typo in committed component | Fix `DeletePostButton.propTypes` to `DeletePost.propTypes` in local copy |
| `Failed to resolve import "prop-types"` | Root dependency undeclared | `npm.cmd install prop-types` in frontend root |
| GET `/posts` returns `404` | Missing `/api/v1` prefix | Set `VITE_BACKEND_URL=http://localhost:3000/api/v1` |
| `ERR_CONNECTION_REFUSED` / `Failed to fetch` | Backend not running or wrong port | Confirm `npm.cmd run dev` in backend and test API directly |
| CORS error in browser | Origin headers missing, proxy mismatch or frontend on different origin | Check browser Network tab and backend CORS config |
| MongoDB server selection timeout | MongoDB not running / port mismatch | Compare `27017` with source fallback `27018`; set backend `DATABASE_URL` |
| Response is HTML, `.json()` throws | Wrong URL/route returns HTML page | Inspect Network response body and URL |
| `Unexpected end of JSON input` | Tried to parse empty `204` | Do not call `response.json()` for successful delete |
| Request succeeds in PowerShell but fails in browser | CORS affects browsers; not PowerShell | Check `Access-Control-Allow-Origin` |
| API URL not updating after editing `.env` | Vite needs restart | Stop and restart `npm.cmd run dev` |
| Empty posts list | Database contains no entries | Create a test post on local test database |

**Debug in the correct order:** API process → direct API request → API URL in `.env` → browser Network tab → React component state. This quickly separates backend, connectivity and frontend bugs.

## 11. Security, quality, and production readiness

- **Never put passwords or database connection strings in `VITE_` variables.** Browser bundles expose those values to users.
- **Do not commit `.env` containing credentials.** GitHub's Chapter 4 tree currently contains a tracked root `.env`; audit it and rotate secrets if any were ever stored there. Ignoring a file later does **not** erase existing Git history.
- **Do not connect React directly to MongoDB.** Keep database operations and authorization on the server.
- **CORS is not authentication.** Production apps must verify authorization at API endpoints.
- **Handle `response.ok`** before assuming `response.json()` contains the data you expect.
- **Treat remote JSON as untrusted.** Validate expected fields and types in important applications.
- **Use HTTPS** for publicly deployed APIs; avoid embedding tokens in URL query parameters.
- **Avoid leaking internal errors** to end users; show a helpful error message and log details appropriately.
- **Document environment differences** across development, staging and production.
- **Use a proxy or allowed-origin list** as appropriate; never indiscriminately relax cross-origin security in production.
- **Consider aborting outdated requests**, particularly when filters change quickly.
- **Use server-state libraries when needed**; the original Chapter 4 already uses TanStack Query.

## 12. Quick reference — commands and patterns

| Purpose | Command / pattern |
|:--|:--|
| Clone source | `git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development.git` |
| Install frontend | From root: `npm.cmd ci` |
| Install backend | From `backend/`: `npm.cmd ci` |
| Start frontend | From root: `npm.cmd run dev` |
| Start backend | From `backend/`: `npm.cmd run dev` |
| Verify API | `Invoke-RestMethod 'http://localhost:3000/api/v1/posts'` |
| Frontend base URL | `VITE_BACKEND_URL=http://localhost:3000/api/v1` |
| Read list | `await fetch(url)` → check `.ok` → `await res.json()` |
| Create record | `fetch(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(data) })` |
| Delete record | `DELETE /api/v1/posts/:id` → HTTP `204` → no body |
| Test independent API | From lab backend: `node --test server.test.mjs` |
| Check frontend build | From root: `npm.cmd run build` |

## 13. Reuse checklist for your next project

- [ ] Verify Node.js and npm versions supported by your Vite version.
- [ ] Keep distinct frontend and backend dependency manifests / lockfiles.
- [ ] Document backend host, port, API prefix and allowed origins.
- [ ] Make a small API smoke request before building React screens.
- [ ] Put only a **public** API base URL in a `VITE_` variable.
- [ ] Add `.env` and local env variants to `.gitignore` **before first commit**.
- [ ] Create a small `src/api/` module that owns all URLs and requests.
- [ ] Check `response.ok`; handle JSON and no-content responses correctly.
- [ ] Show loading, error, empty, and success states.
- [ ] Refresh stale data after mutations, using React Query when appropriate.
- [ ] Add HTTP and UI tests; verify failures, not just happy paths.
- [ ] Check CORS, validation and authentication separately.
- [ ] Run `npm.cmd run build` and review browser console before deploying.
- [ ] Never describe unrun integration tests as proven successful.

## 14. Further learning and official references

**Project evidence:**

- [Chapter 4 — source repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)
- [Source HTTP client (`src/api/posts.js`)](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development/blob/main/src/api/posts.js)
- [Source Blog view (`src/Blog.jsx`)](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development/blob/main/src/Blog.jsx)
- [Source backend routes (`backend/src/routes/posts.js`)](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development/blob/main/backend/src/routes/posts.js)

**Primary references:**

- [MDN — Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [MDN — CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Vite — Environment Variables and Modes](https://vite.dev/guide/env-and-mode)
- [React — Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [TanStack Query — React overview](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Node.js — Test Runner](https://nodejs.org/api/test.html)

---

**Learning sequence:** Cheatsheet 14 (React foundations — **still planned**) → **15 (this guide — published next)** → 16 (TanStack Query) → 17 (React CRUD) → 18 (Filtering and Hooks). Reading 14 first is recommended for absolute beginners, but 15 includes enough explanations to follow independently.

**Remember:** **Learn → Practice → Document → Reuse.**