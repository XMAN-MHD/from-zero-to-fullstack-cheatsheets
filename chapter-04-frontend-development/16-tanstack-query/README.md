# Cheatsheet 16 — TanStack Query from Scratch

**From Zero to Fullstack · Chapter 4: Frontend Development**  
**Audience:** Absolute beginners in React + developers who want a reusable server-state workflow  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code · Node.js  
**Version:** 1.0 · 19 October 2026  
**Reference version:** `@tanstack/react-query` **v5**, consistent with the Chapter 4 project's declared dependency `^5.90.12`  
**Source project:** [Chapter 4 — Frontend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)  
**Learning method:** **Learn → Practice → Document → Reuse**

> **Evidence and scope.** **SOURCE PROJECT** means the code actually read from the user's Chapter 4 GitHub repository. **INDEPENDENT LAB** means the separately written teaching project included in the ZIP file, **not** code already committed to Chapter 4. **IMPROVEMENT** means a recommendation, **not** a source-project feature. Seven dependency-free HTTP tests passed for the lab's Node.js API. The lab's React/Vite build, the source application's browser UI and its MongoDB integration were **not run end-to-end** in this authoring environment. Execute them locally following the instructions below.

## 1. Learning outcomes

By the end of this cheatsheet, you should be able to:

1. Explain **local UI state** versus **server state** in a React application.
2. Install TanStack Query v5 and provide one stable `QueryClient` to the React tree.
3. Fetch data using `useQuery({ queryKey, queryFn })` and understand caching.
4. Handle **pending, fetching, success, empty and error** states correctly.
5. Keep all changing filters and sorting inputs inside a query key.
6. Create, edit and delete server data with `useMutation`.
7. Refresh cached lists after a successful mutation using `invalidateQueries()`.
8. Understand `staleTime`, `gcTime`, background refetching and retries.
9. Trace the exact implementation used in the existing Chapter 4 Blog application.
10. Rebuild and test the workflow in a small project that runs without MongoDB.

### Mental model

```text
React component (local state: input text, filters, UI visibility)
        |
        v
useQuery / useMutation from TanStack Query
        |
        v
QueryClient + query cache  <-- invalidation on successful mutation
        |
        v
fetch() in src/api/posts.js
        |
        v
HTTP REST API -> backend service -> MongoDB (source project)
                       OR
              native Node.js API + in-memory data (independent lab)
```

**React is responsible for rendering UI.** TanStack Query manages the lifecycle and cache of *remote* data; it does **not** replace every `useState`. It does not build the backend, automatically validate your API responses, or secure your database.

## 2. Prerequisites and installation from zero (Windows)

You should know functions, `async/await`, arrays, React components, props, `useState` and basic `fetch()`. Review [Cheatsheet 14](../14-react-components/README.md) and [Cheatsheet 15](../15-react-rest-api/README.md) first. You do **not** need to understand Redux.

Install tools using the official installers:

- **Node.js:** <https://nodejs.org/en/download> (use a supported Node.js 22 LTS or newer that meets Vite 7's engine requirements; Node 22.12+ or 20.19+ is supported by Vite 7).
- **Git:** <https://git-scm.com/downloads>.
- **Visual Studio Code:** <https://code.visualstudio.com/>.
- **Browser:** Chrome, Edge or Firefox and its DevTools.
- **MongoDB Community Server:** <https://www.mongodb.com/try/download/community> **only if reproducing the original MongoDB-backed Chapter 4**. The lab needs no MongoDB.

Open **PowerShell** in VS Code (`Terminal → New Terminal`) and verify:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

> **Windows-specific note:** when `npm.ps1` is blocked by an execution policy, prefer `npm.cmd` in PowerShell. This avoids changing a system-wide PowerShell security setting.

### Create a new React project from zero

```powershell
mkdir tanstack-query-first-project
cd tanstack-query-first-project
npm.cmd create vite@latest . -- --template react
npm.cmd install
npm.cmd install @tanstack/react-query
npm.cmd run dev
```

Open the local URL displayed by Vite (commonly <http://localhost:5173>). **Stop the development server** with `Ctrl+C`. The Vite starter does not contain a backend or the Blog demo yet; the following sections explain what to add. The **INDEPENDENT LAB** provides a complete runnable example so no snippets need to be combined by guesswork.

### Or clone the real Chapter 4 repository

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development.git
cd from-zero-to-fullstack-ch4-frontend-development
npm.cmd install
npm.cmd list @tanstack/react-query
```

The source `package.json` declares React `^19.1.1`, Vite `^7.1.7`, and `@tanstack/react-query` `^5.90.12`. Because this is a repository with a lockfile, **`npm.cmd ci`** is also appropriate for a clean reproducible installation when the lockfile is synchronized with `package.json`. A project's **frontend** dependencies are installed at the repository root; its **backend** dependencies belong in the separate `backend/` folder.

### Verify the original backend separately

The real Chapter 4 has two npm projects: the React frontend in the repository root and the Express backend in `backend/`.

```powershell
# In Terminal A, from the Chapter 4 repository root
cd .\backend
npm.cmd ci
# The source backend expects MongoDB. Set a local, non-secret URL only if needed:
Set-Content .env 'DATABASE_URL=mongodb://localhost:27017/blog'
npm.cmd run dev
```

In **Terminal B** return to the **repository root**, and supply an ignored, local override:

```powershell
Set-Content .env.local 'VITE_BACKEND_URL=http://localhost:3000/api/v1'
npm.cmd run dev
```

The source connection helper defaults to **`mongodb://localhost:27018/blog`**, while a standard local MongoDB installation commonly uses port **27017**. Choose the port **actually used** by your MongoDB server. If you already use port 27018, adjust the example accordingly. The backend uses `cors()` in `backend/src/app.js` and serves `GET /api/v1/posts`. The environment variable must include **`/api/v1`**, because `src/api/posts.js` appends `/posts`. Do not put passwords or secret access tokens in a `VITE_*` variable: Vite exposes it to the browser.

> **Source-project troubleshooting before launch:** the source imports `prop-types` in components, but its root `package.json` does not declare it; `npm.cmd install prop-types` may be required. Its `src/components/DeletePost.jsx` ends with `DeletePostButton.propTypes = ...` even though the component is named `DeletePost`: this is a source-code reference error, not a TanStack Query fault. Use `DeletePost.propTypes = ...` when working in your own checkout. No modifications to Chapter 4 are implied by this cheatsheet.

## 3. The three building blocks: client, provider, query

### 3.1 `QueryClient`: the cache manager

A `QueryClient` manages query caches, invalidations, default options and query lifecycle. Create it **once**, outside the component render function, so rerenders do not reset your cache.

**SOURCE PROJECT — `src/App.jsx` (essential original structure):**

```jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { Blog } from './Blog.jsx'

const queryClient = new QueryClient()

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Blog />
    </QueryClientProvider>
  )
}
```

`QueryClientProvider` makes the client available to all descendant components that use TanStack Query hooks. Calling `useQuery` outside a matching provider normally produces a **No QueryClient set** error.

**IMPROVEMENT — optional defaults for a new app:**

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,  // data stays fresh for 30 s
      retry: 1,           // limit retries for the teaching lab
    },
  },
})
```

This option is used in the **INDEPENDENT LAB**, not in the source's `src/App.jsx`.

### 3.2 `useQuery`: read server data

```jsx
import { useQuery } from '@tanstack/react-query'

function PostTitles() {
  const query = useQuery({
    queryKey: ['posts'],
    queryFn: async () => {
      const response = await fetch('http://localhost:3000/api/v1/posts')
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      return response.json()
    },
  })

  if (query.isPending) return <p>Loading posts…</p>
  if (query.isError) return <p role="alert">{query.error.message}</p>
  return <ul>{query.data.map(post => <li key={post._id}>{post.title}</li>)}</ul>
}
```

**Explanation:** `queryKey` identifies the cached data; `queryFn` returns a **Promise** resolving to that data; `useQuery` subscribes this component to loading/data/error changes. The first fetch may take time; TanStack Query can then reuse cached results for the same key. `queryFn` **must throw/reject** for error states to work as expected.

### 3.3 Reading the v5 query states correctly

| Property | What it tells you | Typical UI |
|:--|:--|:--|
| `isPending` | There is no successful data available yet (also possible if query is disabled) | First-load or not-ready UI |
| `isLoading` | First fetch is actively running (`isPending && isFetching`) | Show a spinner for actual first fetch |
| `isFetching` | Any network fetch is active, including a background refresh | Small "Refreshing" indicator |
| `isSuccess` | Query has successful data | Render the list |
| `isError` / `error` | Query is in an error state | Display a helpful failure message |
| `data` | Latest successful resolved value, or `undefined` before one exists | Iterate only when appropriate |
| `refetch()` | Manually triggers another fetch | Retry button |

For ordinary initial fetches, `isPending` and `isLoading` can look similar. For a **disabled query**, `isPending` may be true although nothing is fetching; `isLoading` is more precise in that case. For background refetches, `isFetching` may be true while the **previous successful data remains available**.

## 4. Source walkthrough: our Chapter 4 Blog application

**SOURCE PROJECT:** `src/Blog.jsx` imports `useQuery`, stores four local filters, and calls `getPosts` through the API module.

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

**Why it matters:** local `useState` values control what the reader wants. `useDebounce(author)` delays the server-facing author filter (the source hook's default delay is **3000 ms**). TanStack Query caches a different data set for each combination of filter and sorting values. `postsQuery.data ?? []` provides an empty-array fallback before a successful response.

The source UI uses `if (postsQuery.isLoading) return <div>Loading...</div>` and `if (postsQuery.error) return <div>Error: {postsQuery.error.message}</div>`. This is source behavior, not an assurance that every case is handled optimally. For your next app, prefer deliberate `isPending` / `isError` / `isFetching` handling and a separate empty-state message.

### Follow one request through the source files

```text
src/Blog.jsx
  useQuery({ queryKey: ['posts', { author, tags, sortBy, sortOrder }] })
       |
       v
src/api/posts.js
  fetch(`${VITE_BACKEND_URL}/posts?${URLSearchParams(...)}`)
       |
       v
backend/src/routes/posts.js
  GET /api/v1/posts  (filter / sort query parameters)
       |
       v
backend/src/services/posts.js  ->  backend/src/db/models/post.js
       |
       v
MongoDB -> JSON response -> TanStack Query cache -> React UI
```

**SOURCE LIMITATION:** `getPosts` currently calls `response.json()` without testing `response.ok`. A `400` or `500` JSON response can therefore be treated as resolved query data instead of a rejected query. That can make a view attempt `posts.map(...)` on an error object. **IMPROVEMENT:** always detect non-success statuses, throw errors and validate expected response shapes where appropriate.

```js
// Suggested replacement for the READ helper, not source code already committed:
export async function getPosts(filters, signal) {
  const base = import.meta.env.VITE_BACKEND_URL
  if (!base) throw new Error('VITE_BACKEND_URL is not configured')
  const qs = new URLSearchParams(filters)
  const response = await fetch(`${base}/posts?${qs}`, { signal })
  if (!response.ok) throw new Error(`Cannot load posts: HTTP ${response.status}`)
  const data = await response.json()
  if (!Array.isArray(data)) throw new Error('Expected an array of posts')
  return data
}
```

The **real source backend** deliberately rejects simultaneous `author` and `tags` filters with `400`. Users should select **one** of these filters, or revise the backend contract intentionally. Do not misdiagnose that `400` as a TanStack Query bug.

## 5. Query keys: how the cache knows what is different

Query keys must be **arrays**. Include every variable your query function uses to select remote data. Examples:

```jsx
['posts']
['posts', { author: 'Amina' }]
['posts', { author: 'Babacar' }]
['posts', { author: 'Amina', sortOrder: 'ascending' }]
['post', '65f...']
```

If `author` changes but the key remains `['posts']`, the query may return results cached for a **different author**. The safe pattern is to derive the key from the query inputs:

```jsx
const filters = { author, sortBy, sortOrder }
const postsQuery = useQuery({
  queryKey: ['posts', filters],
  queryFn: ({ signal }) => getPosts(filters, signal),
})
```

Top-level key order matters (`['posts', 'Amina']` differs from `['Amina', 'posts']`). Within a serializable **object** inside the key, property order does not change the deterministic hash. Prefix matching makes `['posts']` useful for invalidating all filtered versions.

### Cache lifecycle in plain language

1. A component mounts with key `['posts', { author: '' }]` and TanStack Query requests data.
2. The query succeeds; the result is stored under that key.
3. A user changes the author filter, creating another key and possibly a second cached result.
4. Returning to a previous key can show its cached data while a background refresh runs.
5. When the user creates/edits/deletes a post, invalidating keys beginning with `['posts']` marks those results stale and prompts active matching queries to refetch.

| Option | Recommended mental model |
|:--|:--|
| `staleTime` | Successful data stays **fresh** for the configured time; default `0`. |
| `gcTime` | **Inactive** cached data can be garbage-collected; browser default is about five minutes. |
| `retry` | Controls attempts after a failed request; reduce for easier debugging. |
| `refetchOnWindowFocus` | Stale queries may refresh when a browser tab regains focus. |
| `enabled` | Defer a query until a required parameter is ready. |

**Distinction:** `staleTime` is not an expiration date for data and `gcTime` is not the time after which active queries disappear. An invalidation can mark data stale even before `staleTime` expires. Cache reuse and refetch behavior depend on whether data is still cached and a query is active.

## 6. `useMutation`: change server data

**SOURCE PROJECT — `src/components/CreatePost.jsx`:** the component has form fields managed with `useState`, and the mutation creates a post. After successful creation, it resets fields and invalidates every query whose key starts with `['posts']`.

```jsx
const queryClient = useQueryClient()
const createPostMutation = useMutation({
  mutationFn: () => createPost({ title, author, contents, tags: [] }),
  onSuccess: () => {
    setTitle('')
    setAuthor('')
    setContents('')
    setTags('')
    queryClient.invalidateQueries({ queryKey: ['posts'] })
  },
})

// Inside submit handler:
createPostMutation.mutate()
```

The actual source additionally converts comma-separated tags into an array using `.split(',').map(...).filter(Boolean)`. It disables the submit button while `createPostMutation.isPending` is true. Its **EditPost** and **DeletePost** components use the same invalidate-after-success pattern.

**IMPROVEMENT — return the invalidation promise** from `onSuccess` so the mutation can remain pending until related active queries have finished refreshing:

```jsx
const createMutation = useMutation({
  mutationFn: createPost,
  onSuccess: () => {
    setTitle('')
    return queryClient.invalidateQueries({ queryKey: ['posts'] })
  },
})
```

This does not make a remote write atomic with the refetch; it simply coordinates the UI's mutation lifecycle. Use it deliberately, not as an automatic rule for every app.

### A complete form pattern

```jsx
function AddPost() {
  const [title, setTitle] = useState('')
  const queryClient = useQueryClient()
  const mutation = useMutation({
    mutationFn: createPost,
    onSuccess: () => {
      setTitle('')
      return queryClient.invalidateQueries({ queryKey: ['posts'] })
    },
  })

  return <form onSubmit={event => {
    event.preventDefault()
    if (title.trim()) mutation.mutate({ title: title.trim() })
  }}>
    <input value={title} onChange={e => setTitle(e.target.value)} />
    <button disabled={!title.trim() || mutation.isPending}>Add</button>
    {mutation.isError && <p role="alert">{mutation.error.message}</p>}
  </form>
}
```

`useMutation` is for operations such as create/update/delete. Use `useQuery` for reads. A **DELETE `204 No Content`** response has no JSON body; do **not** blindly call `response.json()` in a delete helper. The **INDEPENDENT LAB** handles this correctly.

## 7. Build the independent lab (no MongoDB required)

The ZIP includes a **complete** project under `lab/` with two programs:

```text
lab/
├── README.md
├── api/
│   ├── server.mjs          # Native Node HTTP API; posts in memory
│   └── server.test.mjs     # Seven automated HTTP tests, no npm required
└── frontend/
    ├── .env.example        # Safe public URL setting
    ├── .gitignore
    ├── index.html
    ├── package.json        # React 19, Vite 7, TanStack Query v5
    ├── vite.config.js
    └── src/
        ├── main.jsx       # One stable QueryClient and provider
        ├── Blog.jsx       # Query, mutations, cache invalidation, UI
        ├── api.js         # HTTP helpers, AbortSignal and errors
        └── style.css
```

### Terminal A: start and test the API

Unzip the package and open VS Code inside its `lab` folder. Then:

```powershell
cd .\api
node --test .\server.test.mjs
node .\server.mjs
```

Expect **7 passing tests**, followed by an HTTP server at <http://localhost:3000/api/v1/posts>. Open that URL: you should see JSON with two sample blog posts. The server has no MongoDB dependency, so data resets when you stop/restart it.

### Terminal B: start the React app

In a **new terminal**, starting from the lab folder:

```powershell
cd .\frontend
Copy-Item .env.example .env
npm.cmd install
npm.cmd run dev
```

Open <http://localhost:5173>. A working network connection to the npm registry is needed on first installation. The backend accepts that Vite origin for CORS; if Vite picks another port, fix the conflict or align CORS intentionally. Keep API and Vite running in separate terminals.

### Lab tasks and expected results

| Action in browser | TanStack Query behavior | Expected result |
|:--|:--|:--|
| Open page | `useQuery` fetches `['posts', filters]` | Two posts are displayed |
| Type `Amina` in Author | Changing `queryKey` selects/refetches matching data | One post remains |
| Clear author filter | Old key becomes active again | All posts are available |
| Create a post | `useMutation`, then `invalidateQueries(['posts'])` | Newly created post appears |
| Edit a post title | Update mutation + invalidation | New title is shown |
| Delete a post | Delete mutation + invalidation | Removed post disappears |
| Stop API, then refetch | Request rejects; query can become error after retries | Error message appears |

**Source vs lab:** the real Chapter 4 filters by **author and tags**, and uses a **3-second debounce** for author; the independent lab intentionally uses a simpler author/sorting filter, with no debounce, to isolate TanStack Query's fundamentals. It accepts posts from an in-memory Node API rather than MongoDB. It implements improved HTTP error checks and `AbortSignal` support.

### Watch the network and cache behavior

1. Press **F12** and open the browser's **Network** panel.
2. Change the author filter. Notice the request URL and resulting JSON.
3. Inspect `['posts', filters]` in the `src/Blog.jsx` source.
4. Create an item and observe a `POST` followed by an appropriate `GET` after invalidation.
5. Change `staleTime` in `src/main.jsx` from `30_000` to `0`, refresh and compare behavior.
6. Set `retry: 0` temporarily while deliberately turning the API off, to see failure more quickly.
7. Optional: add `@tanstack/react-query-devtools` and use its v5 Devtools component to inspect the client cache, following the official documentation.

## 8. Tests and verification strategy

Tests make an application more reliable, but be precise about **which part** passed a test.

**Verified during authoring (INDEPENDENT LAB):**

```powershell
node --test .\api\server.test.mjs
```

Seven tests passed on the Node.js native HTTP API: default ordering, author filtering, POST and GET visibility, validation `400`, PATCH plus DELETE `204`, CORS preflight, and unknown-route `404`. These tests do **not** demonstrate that the React UI rendered successfully; they verify only the lab API contract.

**To verify the React frontend on your own machine:**

```powershell
# From lab/frontend after npm.cmd install
npm.cmd run build
npm.cmd run dev
```

Then perform the seven browser actions in the preceding table. The frontend build and browser UI were **not** executed in this document-generation environment, because React/Vite packages were not available there. The source project plus MongoDB was likewise not run end-to-end here.

### A useful error-contract test

```js
// This is intentionally a plain JavaScript helper test idea.
const response = await fetch('http://localhost:3000/api/v1/not-found')
if (response.ok) throw new Error('Expected an unsuccessful response')
console.log(response.status) // Expected: 404
```

A fetch call generally rejects for **network errors**; it does not inherently reject for HTTP `404` or `500`. This is why your API helper checks `response.ok` and throws before returning `response.json()`.

## 9. Troubleshooting (Windows + React + API)

| Problem | What to check or fix |
|:--|:--|
| `No QueryClient set` | Wrap the relevant tree in `QueryClientProvider`. |
| Old author's posts | Add `author` and sort inputs to the query key. |
| HTTP `404` for `/posts` | API base URL needs the `/api/v1` prefix. |
| CORS blocked | Allow the frontend origin `http://localhost:5173` at the API. |
| `data.map is not a function` | API returned an object; throw on non-2xx and validate arrays. |
| HTTP `400` does not show as an error | `fetch` resolved; explicitly check `response.ok`. |
| Error appears slowly | Retry may be active; temporarily configure `retry: 0`. |
| List remains stale after a write | Invalidate keys beginning with `['posts']` after success. |
| Source `DeletePostButton` error | In your checkout, change the identifier to `DeletePost`. |
| Missing `prop-types` module | Install `prop-types` from the frontend root if needed. |
| npm blocked in PowerShell | Run `npm.cmd` rather than `npm.ps1`. |
| MongoDB refused | Check server state and configured 27017 versus 27018 port. |
| React app unreachable | Check the Vite terminal and listening port. |
| `VITE_BACKEND_URL` undefined | Set `.env.local`, then restart the Vite server. |

**Safe debugging order:** verify API directly in a browser → verify correct URL prefix → inspect **Network** status/body → verify `response.ok` handling → inspect `queryKey` → verify provider location → inspect mutation `onSuccess` invalidation.

## 10. Security, maintainability and performance

**Real projects should:**

- Keep passwords, MongoDB URLs with credentials, JWT signing secrets and private API keys on the **backend**, never in `VITE_*` variables or committed client files.
- Require authentication/authorization for protected operations; a client-side disabled button is **not** an authorization check.
- Validate incoming API data on the server and avoid rendering untrusted HTML with `dangerouslySetInnerHTML`.
- Limit and validate filter/sort fields on the backend; do not accept arbitrary object keys as database sort fields.
- Keep network requests in `src/api/`, not duplicated in every form component.
- Design stable **query key factories** for large applications and invalidate only what changed.
- Tune `staleTime` by business freshness requirements; avoid blindly refetching expensive endpoints.
- Pass TanStack Query's `AbortSignal` to `fetch()` when your adapter supports cancellation.
- Show human-readable errors, loading states and empty results; treat 204 as no body.
- Use automated tests and monitor network activity before describing an integration as production-ready.

**Advanced patterns to study after this guide:** pagination and infinite queries, `enabled` for dependent queries, optimistic updates, cancellation, prefetching, Devtools, default query options, network/offline mode and server-side rendering. They are **not** required to complete this beginner cheatsheet.

## 11. Quick reference (copy to a new project)

| Objective | Snippet |
|:--|:--|
| Install | `npm.cmd install @tanstack/react-query` |
| Create client | `const queryClient = new QueryClient()` |
| Provide it | `<QueryClientProvider client={queryClient}>...</QueryClientProvider>` |
| Read list | `useQuery({ queryKey: ['posts'], queryFn: getPosts })` |
| Read filtered list | `useQuery({ queryKey: ['posts', filters], queryFn: () => getPosts(filters) })` |
| Create / update / delete | `useMutation({ mutationFn: createPost })` |
| Access cache client | `const queryClient = useQueryClient()` |
| Refresh all post queries | `queryClient.invalidateQueries({ queryKey: ['posts'] })` |
| Check first-load | `query.isPending` or carefully `query.isLoading` |
| Check background activity | `query.isFetching` |
| Show request failure | `query.isError` and `query.error.message` |
| Handle DELETE no-content | `if (res.status === 204) return null` |
| API base URL | `VITE_BACKEND_URL=http://localhost:3000/api/v1` |
| API tests | `node --test .\api\server.test.mjs` |

### New-project reuse checklist

- [ ] Install Node.js, VS Code and Git, then verify `node` and `npm.cmd`.
- [ ] Create a React/Vite app; install `@tanstack/react-query` v5.
- [ ] Keep a stable `QueryClient` outside render and wrap the app with `QueryClientProvider`.
- [ ] Define a single `src/api/` module with `response.ok` checks and safe JSON handling.
- [ ] Put all changing API inputs in `queryKey`.
- [ ] Render pending, success, empty and failure states explicitly.
- [ ] Use `useMutation` for writes; invalidate or update relevant cached queries on success.
- [ ] Configure `.env.local` using a public API URL only (no credentials).
- [ ] Run API tests, `npm.cmd run build`, and verify React behavior in a browser.
- [ ] Record source version and distinguish live behavior from ideas to implement later.

## 12. Official references and source files

**Official documentation:**

- [TanStack Query v5 — React Overview](https://tanstack.com/query/latest/docs/framework/react/overview)
- [TanStack Query v5 — useQuery reference](https://tanstack.com/query/latest/docs/framework/react/reference/functions/useQuery)
- [TanStack Query v5 — Query Keys](https://tanstack.com/query/latest/docs/framework/react/guides/query-keys)
- [TanStack Query v5 — Query Functions](https://tanstack.com/query/latest/docs/framework/react/guides/query-functions)
- [TanStack Query v5 — Invalidations from Mutations](https://tanstack.com/query/latest/docs/framework/react/guides/invalidations-from-mutations)
- [TanStack Query v5 — Query Invalidation](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation)
- [React — Learn](https://react.dev/learn)
- [Vite — Getting Started](https://vite.dev/guide/)
- [MDN — Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)

**Project files checked in the actual Chapter 4 GitHub repo:** `package.json`, `src/App.jsx`, `src/Blog.jsx`, `src/api/posts.js`, `src/components/CreatePost.jsx`, `src/components/EditPost.jsx`, `src/components/DeletePost.jsx`, and `src/hooks/useDebounce.js`. The backend route prefix, default MongoDB port and Express CORS setup were also verified against the Chapter 4 backend files during the previous React/API documentation work. See the [source repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development) for its current state.

---

*Build it, understand it, document it, reuse it. — From Zero to Fullstack.*