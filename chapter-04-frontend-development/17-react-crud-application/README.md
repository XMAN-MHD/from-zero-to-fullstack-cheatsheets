# Cheatsheet 17 — Complete React CRUD Application

**From Zero to Fullstack · Chapter 04 — Frontend Development**  
**Audience:** React beginners and returning full-stack developers  
**Primary environment:** Windows 10/11 · PowerShell · Visual Studio Code · Node.js LTS  
**Version:** 1.0 · 20 October 2026  
**Source:** [Chapter 04 — Frontend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)  
**Method:** **Learn → Practice → Document → Reuse**

> **Evidence policy.** **SOURCE PROJECT** means files inspected directly in the existing Chapter 04 GitHub repository. **INDEPENDENT LAB** means an original practice application distributed in this guide's ZIP, not committed to the source repository. **IMPROVEMENT** means suggested code, not an already implemented feature. The lab's 10 dependency-free HTTP tests passed during authoring. Neither the lab's Vite/React build nor the MongoDB-backed source application was run end-to-end here: perform those checks on your machine.

## 1. What you will learn

By the end, you should be able to:

1. Explain the CRUD lifecycle across a React interface, HTTP API, services and a database.
2. Install and launch the actual Chapter 04 frontend and its separate backend on Windows.
3. Trace an existing Create, Read, Update and Delete action through concrete source files.
4. Connect a form to an HTTP mutation using TanStack Query v5.
5. Read and filter posts with stable query keys and `useQuery`.
6. Refresh cached lists after mutations and understand why `invalidateQueries` matters.
7. Handle pending, success, error, empty-list, confirmation and `204 No Content` states.
8. Avoid common mistakes in client-side validation, API error handling and identifiers.
9. Complete a working standalone React + Node.js CRUD laboratory without MongoDB.
10. Reuse a systematic test and deployment-readiness checklist.

### Architecture in one diagram

```text
                           USER / BROWSER
                                  |
           create / filter / edit / delete / reload
                                  |
                                  v
  React: Blog, CreatePost, PostList, Post, EditPost, DeletePost
                                  |
                        TanStack Query v5
             useQuery / useMutation / QueryClient
                                  |
                  src/api/posts.js -> fetch()
                                  |
             HTTP /api/v1/posts   GET POST PATCH DELETE
                                  |
                    Express backend (source)
                                  |
                routes/posts.js -> services/posts.js
                                  |
                   Mongoose -> MongoDB

INDEPENDENT LAB: same HTTP CRUD idea but a native Node.js
in-memory server, so no MongoDB installation is required.
```

**CRUD** stands for **Create**, **Read**, **Update** and **Delete**. A modern React CRUD app must manage *both* the user interface state (form fields, edit mode) and the server state (posts returned by the API).

## 2. Prerequisites and software from scratch

Install these tools on Windows:

| Tool | Why it is needed | Verification command |
| --- | --- | --- |
| Node.js LTS (version compatible with Vite 7; Node >=20.19 or >=22.12) | Node runtime + npm | `node --version` |
| npm | Install frontend and backend packages | `npm.cmd --version` |
| Git for Windows | Clone code and track changes | `git --version` |
| Visual Studio Code | Edit the React and Node.js code | `code --version` |
| Browser developer tools | Inspect requests, console and UI | F12 -> Network |
| MongoDB locally or reachable server | Required **only for the source project**, not the lab | Test the matching connection URI |

Download packages from official websites: <https://nodejs.org/>, <https://git-scm.com/downloads/win>, <https://code.visualstudio.com/> and (if needed) <https://www.mongodb.com/try/download/community>.

**PowerShell tip:** If PowerShell blocks `npm.ps1` with an execution-policy warning, use **`npm.cmd`** (as in every command below). You do not need to loosen the system-wide execution policy.

Before running any project, verify:

```powershell
node --version
npm.cmd --version
git --version
```

## 3. Inspect the actual Chapter 04 structure

The source repository is **not** a single-package application. It contains a React frontend in the root and a separate Express backend inside `backend/`.

```text
from-zero-to-fullstack-ch4-frontend-development/
|-- package.json               # Frontend: React 19 + Vite 7
|-- src/
|   |-- main.jsx               # React entry
|   |-- App.jsx                # QueryClientProvider
|   |-- Blog.jsx               # Filters and useQuery
|   |-- api/posts.js           # fetch() functions
|   |-- components/
|   |   |-- CreatePost.jsx      # POST /posts
|   |   |-- PostList.jsx        # List rendering
|   |   |-- Post.jsx            # Card / edit switch
|   |   |-- EditPost.jsx        # PATCH /posts/:id
|   |   |-- DeletePost.jsx      # DELETE /posts/:id
|   |   |-- PostFilter.jsx
|   |   `-- PostSorting.jsx
|   `-- hooks/useDebounce.js
`-- backend/
    |-- package.json           # Separate backend install
    `-- src/
        |-- index.js           # DB connection + listen
        |-- app.js             # cors(), express.json()
        |-- routes/posts.js    # /api/v1/posts endpoints
        |-- services/posts.js  # MongoDB operations
        `-- db/
            |-- init.js
            `-- models/post.js
```

**Actual package versions in source:** React `^19.1.1`, React DOM `^19.1.1`, TanStack React Query `^5.90.12`, Vite `^7.1.7`, Express `^5.1.0`, Mongoose `^8.0.2`.

## 4. Run the actual Chapter 04 project on Windows

Open **two separate PowerShell terminals** in VS Code. First clone and install the frontend:

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development.git
cd .\from-zero-to-fullstack-ch4-frontend-development
npm.cmd ci
```

**Known issue in the source:** `Post.jsx`, `PostList.jsx`, `PostFilter.jsx`, `PostSorting.jsx` and `DeletePost.jsx` import `prop-types`, but the frontend `package.json` does **not** declare it. Install it locally:

```powershell
npm.cmd install prop-types
```

**Known issue in `src/components/DeletePost.jsx`:** the file declares `export default function DeletePost(...)`, but its final line refers to an undefined `DeletePostButton` variable. Replace:

```js
DeletePostButton.propTypes = {
  postId: PropTypes.string.isRequired,
}
```

with:

```js
DeletePost.propTypes = {
  postId: PropTypes.string.isRequired,
}
```

This is a local correction procedure, **not** a claim that the GitHub source has been changed.

### 4.1 Configure the frontend API base URL

Create `.env.local` in the repository **root** and use:

```dotenv
VITE_BACKEND_URL=http://localhost:3000/api/v1
```

**Why `/api/v1`?** The frontend's `getPosts()` appends `/posts`, and the backend route is `/api/v1/posts`. Setting only `http://localhost:3000` yields a **404** at `/posts`.

The source repository already contains a tracked root `.env` file; avoid committing environment changes or any sensitive values. `.env.local` is a Vite-specific local override, but check `.gitignore` before committing anything. Restart the Vite dev server after changing environment variables. `VITE_` variables are embedded in the public browser bundle; **never store secrets** in them.

### 4.2 Prepare the source backend and MongoDB

In Terminal A:

```powershell
cd .\backend
npm.cmd ci
```

The real source code defaults to `mongodb://localhost:27018/blog` in `backend/src/db/init.js`. This is **not** MongoDB's typical local port `27017`. Either start your MongoDB instance on port `27018`, or create `backend/.env` containing the URI that matches your actual setup:

```dotenv
DATABASE_URL=mongodb://127.0.0.1:27017/blog
PORT=3000
```

**Never commit** the backend `.env`. The source `backend/.gitignore` ignores it. The example above is only for local, unauthenticated development; production databases should use least-privilege credentials and secure networking.

Then:

```powershell
npm.cmd run dev
```

You should see a database connection confirmation and the HTTP server address if MongoDB is reachable. If MongoDB is absent, the backend cannot complete normal startup.

### 4.3 Start the frontend

In Terminal B, from the **repository root**:

```powershell
npm.cmd run dev
```

Open the localhost URL printed by Vite (normally `http://localhost:5173`). Try listing posts, creating one, editing it and deleting it. Use F12 -> **Network** and locate requests to `/api/v1/posts`.

**Verify instead of assuming:** `npm.cmd run build`, `npm.cmd run lint`, and the backend `npm.cmd test` must be executed on your machine. Their results are not claimed here. The original backend's Jest setup may download a MongoDB test binary; this needs network access and compatible operating-system support.

## 5. The HTTP CRUD contract

The existing Express backend registers these routes in `backend/src/routes/posts.js`:

| Action | HTTP | Endpoint | Source UI file | Expected success |
| --- | --- | --- | --- | --- |
| Read all | `GET` | `/api/v1/posts` | `Blog.jsx` -> `getPosts()` | `200` JSON array |
| Read one | `GET` | `/api/v1/posts/:id` | Backend supports; frontend list does not call it directly | `200` JSON object |
| Create | `POST` | `/api/v1/posts` | `CreatePost.jsx` | `201` JSON object |
| Update | `PATCH` | `/api/v1/posts/:id` | `EditPost.jsx` | `200` updated JSON object |
| Delete | `DELETE` | `/api/v1/posts/:id` | `DeletePost.jsx` | `204` empty body |

A typical post JSON object:

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "title": "Full-stack learning",
  "author": "Amina",
  "contents": "React talks to Express through HTTP.",
  "tags": ["react", "api"],
  "createdAt": "2026-10-20T09:00:00.000Z",
  "updatedAt": "2026-10-20T09:00:00.000Z"
}
```

This is an **illustrative response**, not a document retrieved from the user's database. MongoDB normally generates `_id` as an ObjectId; the independent lab instead uses simple string IDs.

**Source-specific filtering rule:** the current backend reads `author` and `tags` (plural) query parameters. It **returns 400 if both are provided simultaneously**. Tag filtering uses the comma-separated value to build a MongoDB `$in` query. The standalone lab deliberately allows combined filters as an **improvement**; do not confuse that with the source project.

## 6. READ — display and synchronize posts

The actual `src/Blog.jsx` uses four local state values and one TanStack Query request:

```jsx
const [author, setAuthor] = useState('')
const [tags, setTags] = useState('')
const [sortBy, setSortBy] = useState('createdAt')
const [sortOrder, setSortOrder] = useState('descending')

const debouncedAuthor = useDebounce(author)

const postsQuery = useQuery({
  queryKey: ['posts', { author: debouncedAuthor, tags, sortBy, sortOrder }],
  queryFn: () => getPosts({ author: debouncedAuthor, tags, sortBy, sortOrder }),
})
const posts = postsQuery.data ?? []
```

**Key lesson:** a query key must represent the values used by its query function. When a filter changes, TanStack Query can fetch/cache a different result. The source debounces **author** only (`useDebounce` defaults to 3000 ms); tag and sorting changes are immediate.

`PostList.jsx` loops through `posts` and renders `<Post {...post} />` with a stable `_id` key. Each `Post` displays the article, author, tags and edit/delete controls.

### Handle all read states

```jsx
if (postsQuery.isPending) return <p>Loading posts…</p>
if (postsQuery.isError) return <p role="alert">{postsQuery.error.message}</p>
if (posts.length === 0) return <p>No posts found.</p>
return <PostList posts={posts} />
```

For a better UI, show a small 'Updating…' indicator when `isFetching` is true and cached data remains visible. Avoid treating every refetch as a full-page loading event. These are **improvement examples**.

## 7. CREATE — form -> POST -> invalidate list

The source `CreatePost.jsx` stores `title`, `author`, `contents` and `tags` using `useState`. The tags input becomes an array using `.split(',').map(...trim()).filter(Boolean)`.

Its mutation is conceptually:

```jsx
const queryClient = useQueryClient()
const createPostMutation = useMutation({
  mutationFn: () => createPost({ title, author, contents, tags: parsedTags }),
  onSuccess: () => {
    // clear form fields
    queryClient.invalidateQueries({ queryKey: ['posts'] })
  },
})
```

The real component clears the fields in `onSuccess`, disables the submit button when `!title` or pending, and reports a success message. **Improvement:** use `!title.trim()` to reject whitespace-only titles and render `createPostMutation.isError` so the user sees failed requests.

### What happens on submit?

```text
User types into controlled inputs
  -> onSubmit prevents browser reload
  -> useMutation sends POST /api/v1/posts
  -> Express calls createPost() service
  -> Mongoose validates required title and saves post
  -> API returns 201 + JSON
  -> onSuccess resets form and invalidates ['posts'] queries
  -> active post list refetches and renders latest server data
```

## 8. UPDATE — local edit mode -> PATCH -> refresh

`Post.jsx` manages `isEditing` with local `useState(false)`. Clicking **Edit** swaps the article presentation for `<EditPost post={...} onClose={...} />`.

`EditPost.jsx` initializes its form values from `post` and sends a partial update:

```jsx
const updateMutation = useMutation({
  mutationFn: (updates) => updatePost(post._id, updates),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['posts'] })
    onClose?.()
  },
})

updateMutation.mutate({
  title,
  author,
  contents,
  tags: tags.split(',').map(s => s.trim()).filter(Boolean),
})
```

`PATCH` updates selected fields. The backend service uses an allowlist of `title`, `author`, `contents`, `tags` and `runValidators: true`, preventing arbitrary fields from being modified through that service.

**Improvement:** block whitespace-only titles; disable Save while pending; display a friendly validation error; keep the form open if the request fails; consider resetting form values when a *different* post is selected (use a stable `key` or controlled reset).

## 9. DELETE — confirm -> DELETE -> 204

`DeletePost.jsx` shows a simple custom confirmation overlay and then triggers a mutation:

```jsx
const deleteMutation = useMutation({
  mutationFn: () => deletePost(postId),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['posts'] }),
})
```

The backend returns `204 No Content` when successful. **Do not run `response.json()` on a 204:** there is no body to parse. The source `deletePost` helper correctly returns the `Response` itself after checking `res.ok`.

**User-experience improvement:** keep the confirmation open or show an actionable message until the delete result is known. In the source, the confirmation overlay closes immediately after `mutate()` is called, even before the network response arrives.

## 10. The reusable API layer: handling failures correctly

In source `src/api/posts.js`, `createPost`, `updatePost` and `deletePost` check `!res.ok`, but **`getPosts` does not**. Because `fetch()` only rejects on network errors (not ordinary 400/500 HTTP statuses), a 400/500 JSON body can be treated as successful query data if it is parsed without checking `ok`.

**Recommended, not yet in source:** centralize error handling:

```js
const baseUrl = import.meta.env.VITE_BACKEND_URL

async function apiFetch(path, init) {
  const response = await fetch(`${baseUrl}${path}`, init)
  if (!response.ok) {
    throw new Error(`API request failed: HTTP ${response.status}`)
  }
  if (response.status === 204) return undefined
  return response.json()
}

export function getPosts(filters = {}) {
  const params = new URLSearchParams()
  for (const [key, value] of Object.entries(filters)) {
    if (value !== '' && value != null) params.set(key, value)
  }
  return apiFetch(`/posts?${params}`)
}

export function createPost(post) {
  return apiFetch('/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(post),
  })
}
```

Extend similarly for `PATCH` and `DELETE`, using `encodeURIComponent(id)` for URL path values. The independent lab implements a version of this central helper in `lab/src/api.js`.

## 11. TanStack Query v5: mutations, caching and consistency

The existing application creates **one stable `QueryClient`** in `src/App.jsx` and wraps `<Blog />` in `<QueryClientProvider>`.

| Tool | Responsibility | In our source project |
| --- | --- | --- |
| `useState` | Local UI state | Inputs, editing mode, filters |
| `useQuery` | Fetch/read and cache server data | Listing posts |
| `queryKey` | Distinguish different result sets | `['posts', { author, tags, sortBy, sortOrder }]` |
| `useMutation` | Execute write request | Create, edit, delete |
| `invalidateQueries` | Mark matching queries stale, refetch active ones | Called after writes |
| `isPending` | Mutation pending; on a query it means no successful cached result yet | Disable Save/Delete actions |
| `isFetching` | Network fetch currently running | Good for small background indicator |

**Query key gotcha:** invalidating `['posts']` matches the list keys beginning with `posts`, including differently filtered variants; that is intentional after a write.

**Caching is not persistence.** The query cache does not replace the database; reloads or garbage collection can remove cached values. Server data remains the source of truth.

**Scope note:** Chapter 04 already implements basic TanStack Query interactions. Complex optimistic updates, pagination, authentication and offline synchronization are not implemented by the inspected source.

## 12. Independent lab — full CRUD without MongoDB

The ZIP accompanying this guide includes an **independent** working practice project. It has the same main concepts but **does not modify the GitHub source**.

### 12.1 Files

```text
lab/
|-- package.json                  # React, Vite, TanStack Query v5
|-- .env.example                  # Public API base URL
|-- index.html
|-- vite.config.js
|-- server/
|   |-- server.mjs               # Native Node.js HTTP CRUD API
|   `-- server.test.mjs          # Dependency-free HTTP tests
|-- src/
|   |-- main.jsx                 # QueryClientProvider
|   |-- App.jsx                  # Read, filter, create, edit, delete
|   |-- api.js                   # fetch, HTTP errors, 204 response
|   |-- styles.css
|   `-- components/
|       `-- PostForm.jsx         # Reusable create/edit form
`-- README.md
```

**Differences from the source project:** lab data is stored in memory (lost when API restarts); IDs are plain strings; the lab accepts combined author + tag filters; CORS is limited to the lab's local Vite origin. No authentication or database is present. The lab is a teaching example, not a production backend.

### 12.2 Install and start on Windows

Extract the ZIP and open a terminal in its `lab` folder:

```powershell
cd .\lab
npm.cmd install
Copy-Item .env.example .env.local
```

Open **two terminals**, both from the `lab` directory:

**Terminal A (API):**

```powershell
npm.cmd run api
```

**Terminal B (React frontend):**

```powershell
npm.cmd run dev
```

Open `http://127.0.0.1:5173` (or check Vite output). API: `http://127.0.0.1:3001/api/v1/posts`. Make sure port 3001 is free before starting the API.

Test the entire flow: see two seeded posts; add a post; filter by author/tag; switch sorting; edit; cancel; delete after confirmation; reload. Restarting the API **resets** its records.

### 12.3 Core frontend flow in the lab

```jsx
const client = useQueryClient()
const filters = { author, tags, sortBy: 'createdAt', sortOrder }

const postsQuery = useQuery({
  queryKey: ['posts', filters],
  queryFn: () => listPosts(filters),
})

const addMutation = useMutation({
  mutationFn: createPost,
  onSuccess: () => client.invalidateQueries({ queryKey: ['posts'] }),
})

const editMutation = useMutation({
  mutationFn: ({ id, changes }) => updatePost(id, changes),
  onSuccess: () => client.invalidateQueries({ queryKey: ['posts'] }),
})

const removeMutation = useMutation({
  mutationFn: deletePost,
  onSuccess: () => client.invalidateQueries({ queryKey: ['posts'] }),
})
```

The full lab code also handles form resets, error notices, edit mode, deletion confirmation and a `204` response.

### 12.4 Run the API tests and a frontend build

```powershell
npm.cmd run test:api
npm.cmd run build
```

The API test script uses **Node.js's built-in test runner**, so you can also execute `node --test server/server.test.mjs` without installing any npm dependencies. It verifies ten cases: GET list, author filter, tag/sort, POST success, POST validation error, PATCH success, PATCH validation error, DELETE success/no body, unknown ID and CORS preflight.

**Authoring verification:** **10/10 HTTP tests passed**. The frontend build and real browser-based CRUD were **not** executed here; success depends on installing React/Vite and running the browser workflow locally.

### 12.5 Optional PowerShell API smoke tests

With the lab API running:

```powershell
$base = 'http://127.0.0.1:3001/api/v1/posts'
Invoke-RestMethod -Uri $base -Method Get

$post = @{ title = 'Created from PowerShell'; author = 'Tester'; tags = @('demo') } | ConvertTo-Json
$created = Invoke-RestMethod -Uri $base -Method Post -ContentType 'application/json' -Body $post
$created._id

$body = @{ title = 'Updated from PowerShell' } | ConvertTo-Json
Invoke-RestMethod -Uri "$base/$($created._id)" -Method Patch -ContentType 'application/json' -Body $body
Invoke-RestMethod -Uri "$base/$($created._id)" -Method Delete
```

`DELETE` succeeds with an **empty** `204` response; no JSON payload is expected.

## 13. Validate the complete CRUD experience

| Scenario | Action | Expected UI / network behavior |
| --- | --- | --- |
| Initial read | Open the page | `GET` request returns array, cards render |
| Empty list | Apply unmatched filter | Friendly empty state (not a crash) |
| Create | Submit valid title | `POST 201`, form clears after success, list refreshes |
| Invalid create | Empty/whitespace title | Form blocks it, or API returns `400` in lab |
| Edit | Update an article | `PATCH 200`, updated contents appear |
| Cancel edit | Click Cancel | Existing article remains unchanged |
| Delete cancel | Cancel confirmation | No DELETE network request |
| Delete confirm | Confirm deletion | `DELETE 204`, post disappears after refetch |
| Server offline | Stop API, refetch | Visible error and retry path |
| Reopen | Reload the tab | Data fetched again; lab API resets only when server restarts |

**Inspect network requests:** F12 -> Network -> Fetch/XHR. Inspect the method, path, request body, status and response content. Ensure the frontend uses the correct API prefix and no secrets appear in the browser bundle.

## 14. Security, validation and production boundaries

**Source findings:** the existing Chapter 04 Express app uses unrestricted `cors()` (no explicit allowed-origin policy), and API routes do not implement authentication/authorization. Error catches often respond with generic `500`, including validation or malformed-ID failures that should usually be client-side `400` responses.

**Recommendations (not yet implemented):**

1. Enforce server-side validation of `title`, length, tags, supported query parameters and IDs. Client-side validation improves UX but is not a security boundary.
2. Authenticate users and authorize each write; showing/hiding a Delete button in React does not protect the endpoint.
3. Restrict CORS to actual deployed origins. CORS is not authentication.
4. Avoid returning stack traces or sensitive database details in API error JSON.
5. Use least-privilege MongoDB credentials and protect `.env` / `.env.local`; never embed private tokens in `VITE_` values.
6. Bound request body sizes, paginate large post lists and rate-limit write endpoints.
7. Use allowlists for allowed updates and sorting fields; do not allow arbitrary client input to become a MongoDB query operator.
8. Protect against CSRF when cookie-based browser authentication is introduced; configure cookie and CORS policies together.
9. Treat the independent lab as **local practice only**: its in-memory data, no-auth routes and lack of persistent storage make it unsuitable for public hosting.

**React and XSS:** JSX normally escapes ordinary interpolated strings such as `{post.contents}`. Avoid `dangerouslySetInnerHTML` for untrusted post data unless HTML is sanitized with a robust policy.

## 15. Troubleshooting

| Symptom | Most likely cause | What to check |
| --- | --- | --- |
| `Module not found: prop-types` | Source imports an undeclared package | `npm.cmd install prop-types` at frontend root |
| `DeletePostButton is not defined` | Incorrect name in source `DeletePost.jsx` | Change `DeletePostButton.propTypes` to `DeletePost.propTypes` |
| `404` when reading posts | API prefix omitted | `VITE_BACKEND_URL=http://localhost:3000/api/v1` |
| Blank list or odd query error | Source `getPosts()` does not check `res.ok` | Add `if (!res.ok) throw ...` before JSON parsing |
| CORS error | Client origin, server policy, port or API down | Check backend status and response headers |
| `ECONNREFUSED` / failed to fetch | Wrong port, API not started | Check which port the backend actually listens on |
| MongoDB cannot connect | DB not running or wrong port (`27018` vs `27017`) | Match `DATABASE_URL` to the actual database instance |
| Can't create a post | Missing/blank title, backend validation, API error | Inspect POST payload/status and show mutation error |
| Delete JSON parse error | Trying to parse the 204 empty body | For status 204, return `undefined` / the response |
| Changes not visible | Cache not invalidated | Check `onSuccess` and `invalidateQueries({queryKey:['posts']})` |
| Tags + author yield `400` in source | Source intentionally forbids combined filters | Clear one filter; lab differs intentionally |
| PowerShell `npm.ps1` blocked | ExecutionPolicy disallows `.ps1` | Use `npm.cmd` |
| `.env` changes have no effect | Vite read env on startup | Restart dev server; check correct file/location |
| Frontend build fails | Uninstalled packages / syntax / version mismatch | `npm.cmd ci`; check Node version and browser console |

## 16. Quick reference

```powershell
# Source project (frontend root)
node --version
npm.cmd ci
npm.cmd install prop-types
npm.cmd run dev
npm.cmd run build

# Source project backend (separate shell)
cd .\backend
npm.cmd ci
npm.cmd run dev

# Standalone ZIP lab, from lab/
npm.cmd install
npm.cmd run api           # terminal A
npm.cmd run dev           # terminal B
npm.cmd run test:api      # API test runner
npm.cmd run build         # React/Vite build
```

```js
// CRUD overview
GET    /api/v1/posts         // read list
GET    /api/v1/posts/:id     // read one
POST   /api/v1/posts         // create
PATCH  /api/v1/posts/:id     // partial update
DELETE /api/v1/posts/:id     // delete, 204 empty

// TanStack Query essentials
useQuery({ queryKey: ['posts', filters], queryFn: fetchPosts })
useMutation({ mutationFn: createPost, onSuccess: () => client.invalidateQueries({ queryKey: ['posts'] }) })
```

## 17. Reuse checklist

- [ ] Frontend and backend packages installed in the **correct directories**.
- [ ] Local API URLs and version prefixes confirmed in Network dev tools.
- [ ] API contract written down: methods, input fields, status codes and errors.
- [ ] `fetch` helper checks `response.ok` and handles `204` without JSON parsing.
- [ ] `useQuery` has a query key matching all active request parameters.
- [ ] Create/Edit/Delete mutations invalidate the expected list queries.
- [ ] Forms use controlled inputs, `preventDefault()` and visible errors.
- [ ] Stable IDs are used for React `key` props and route parameters.
- [ ] Loading, empty, success, error and pending states verified.
- [ ] Write operations include confirmation or safe cancellation where appropriate.
- [ ] Backend checks permissions and validates input; no secrets in Vite env.
- [ ] API tests, frontend build, lint and actual browser CRUD tested locally.
- [ ] Distinguish **SOURCE PROJECT** from **INDEPENDENT LAB** in documentation.

## 18. References and next steps

- [Chapter 04 original source](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)
- [React: Thinking in React](https://react.dev/learn/thinking-in-react)
- [React: State and lifecycle](https://react.dev/learn/state-a-components-memory)
- [TanStack Query v5 — React overview](https://tanstack.com/query/latest/docs/framework/react/overview)
- [TanStack Query — invalidation after mutations](https://tanstack.com/query/latest/docs/framework/react/guides/invalidations-from-mutations)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [Vite environment variables](https://vite.dev/guide/env-and-mode)
- [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)

**Next guide:** **Cheatsheet 18 — Filtering, Sorting & Custom Hooks**, the final planned guide in Chapter 04.

---

*Learn → Practice → Document → Reuse.*