# Cheatsheet 18 — Filtering, Sorting & Custom Hooks

**From Zero to Fullstack · Chapter 04: Frontend Development**  
**Audience:** React beginners, JavaScript learners, and developers who want reusable data-list patterns  
**Platform:** Windows 10/11 · PowerShell · Visual Studio Code · Node.js · Vite 7 · React 19  
**Version:** 1.0 · 20 October 2026  
**Source:** [Chapter 4 — Frontend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)  
**Method:** **Learn → Practice → Document → Reuse**

> **Evidence and scope.** **SOURCE PROJECT** means code examined directly in the Chapter 4 GitHub repository. **INDEPENDENT LAB** means the separate runnable sample supplied in the ZIP; its features are **not** claimed to be present in Chapter 4. **IMPROVEMENT** means a recommended change, not an implemented change. The lab's 14 dependency-free Node.js tests passed when this guide was prepared. React/Vite browser interaction and the original MongoDB-backed project have **not** been tested end-to-end here.

## 1. Goals: what you will learn

By the end of this guide, you will be able to:

1. Explain filtering, sorting, debouncing, and custom React hooks with simple examples.
2. Identify the exact source files implementing filters and sorting in our React Blog.
3. Separate **local UI state** from **server-managed post data**.
4. Implement controlled inputs for an author query and comma-separated tag query.
5. Build sorting controls with an allowlisted field and a direction.
6. Understand how `URLSearchParams` turns filters into query-string parameters.
7. Configure `useQuery` so its `queryKey` changes when filters change.
8. Explain why a `useDebounce` hook needs both `useEffect` and timer cleanup.
9. Extract related filter controls into a reusable `usePostFilters` custom hook.
10. Choose between **server-side** and **client-side** filtering and sorting.
11. Diagnose mismatched API contracts, CORS, requests that never refresh, and poor empty/error handling.
12. Reproduce the concepts in a small HTTP/React laboratory independent of MongoDB.

### The big picture

```text
User types "amina" / selects sorting
                 |
                 v
Controlled input -> useState (immediately updates UI)
                 |
                 v
useDebounce (350 ms lab / 3000 ms source default)
                 |
                 v
useQuery(['posts', {author, tags, sortBy, sortOrder}])
                 |
                 v
src/api/posts.js -> fetch(URLSearchParams(...))
                 |
                 v
GET /api/v1/posts?author=amina&sortOrder=descending
                 |
                 v
Backend filters + sorts -> JSON response
                 |
                 v
TanStack Query stores result -> <PostList> renders it
```

**Remember:** input state changes immediately; the debounced value changes later; the query's cache key represents the requested dataset.

## 2. Prerequisites and installation from zero (Windows)

You should already recognize the basics from Cheatsheets **03 (Vite), 14 (React basics), 15 (REST APIs), 16 (TanStack Query)** and 17 (CRUD). They are helpful but not required to follow the lab.

| Tool | Role | Verify in PowerShell |
|---|---|---|
| Current supported Node.js LTS compatible with Vite 7 | Run Vite and native Node.js tests | `node --version` |
| npm | Install JavaScript dependencies | `npm.cmd --version` |
| Git | Retrieve the real Chapter 4 | `git --version` |
| VS Code | Edit `.jsx` and `.js` files | `code --version` |
| Browser | Inspect UI and network requests | F12, Network tab |

Install tools through their official installers: [nodejs.org](https://nodejs.org/), [git-scm.com](https://git-scm.com/), [code.visualstudio.com](https://code.visualstudio.com/). Restart your terminal after installation.

### Open the real source project

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development.git
cd from-zero-to-fullstack-ch4-frontend-development
npm.cmd install
code .
```

The repo has **two package roots**: frontend scripts in the root `package.json`; backend scripts in `backend/package.json`. Use **separate terminals** to run them. The source frontend's `package.json` declares `@tanstack/react-query`, `react`, and `react-dom`, but imports `prop-types` without declaring it: if a missing-module error occurs, use `npm.cmd install prop-types` in the frontend root. This is a **local corrective step**; the source repository has not been changed by this guide.

One more source issue: `src/components/DeletePost.jsx` finishes with `DeletePostButton.propTypes = ...`, although the component is named `DeletePost`. Change that reference to `DeletePost.propTypes` **in your local working copy** to avoid a runtime `ReferenceError`. `PostSorting.jsx` declares `field` instead of its actual `fields` prop in `propTypes`; correct that if you want accurate prop checks.

### Configuring the real frontend and backend

The backend routes are under `/api/v1/posts`, so the base URL for `src/api/posts.js` needs the **`/api/v1` prefix**:

```dotenv
# Frontend .env.local, placed at repository root (example only)
VITE_BACKEND_URL=http://localhost:3000/api/v1
```

**Do not put credentials in `VITE_` variables**: Vite exposes them to browser code. Check whether environment files are already tracked before committing; do not commit secrets.

For the real backend, MongoDB must be available. The source `backend/src/db/init.js` defaults to `mongodb://localhost:27018/blog`; standard local installations commonly use port `27017`, so specify an environment variable that matches **your actual database**, or start MongoDB on the port expected by the project. Then run:

```powershell
# Terminal A, from the project root
cd backend
npm.cmd install
npm.cmd run dev

# Terminal B, from the project root
npm.cmd run dev
```

The source backend has CORS middleware (`cors()`); the frontend Vite server typically runs on a different port. These steps describe **how to start it**, not a claim that the original stack was successfully launched during this guide's preparation.

## 3. Find filtering, sorting, and hooks in the actual source

| Source file | Actual responsibility |
|---|---|
| `src/Blog.jsx` | Holds `author`, `tags`, `sortBy`, `sortOrder`; calls `useDebounce(author)` and `useQuery(...)`; renders controls and results. |
| `src/components/PostFilter.jsx` | Controlled text input: `value`, `onChange`, and accessible `htmlFor`. |
| `src/components/PostSorting.jsx` | Controlled sorting-field and sorting-direction dropdowns. |
| `src/hooks/useDebounce.js` | `useState` + `useEffect` + `setTimeout` + `clearTimeout`; default delay **3000 ms**. |
| `src/api/posts.js` | Forms the HTTP URL using `new URLSearchParams(queryParams)` and `VITE_BACKEND_URL`. |
| `backend/src/routes/posts.js` | Reads query params `author`, `tags`, `sortBy`, `sortOrder`, returning HTTP 400 if author and tags are both present. |
| `backend/src/services/posts.js` | Implements MongoDB author/tag filtering and Mongoose `.sort(...)`. |

**SOURCE PROJECT:** The author input is debounced, but **the tags input is not**. Both fields and both sorting values are included in the query key. This means tags edits can trigger more frequent refetches than author edits.

### Actual code shape in `Blog.jsx`

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
```

This excerpt preserves the source's behavior while simplifying spacing. It is **not** a new architecture being passed off as existing code.

## 4. Filtering explained: narrowing down a dataset

A **filter** keeps only items meeting conditions. If a list contains 100 posts and you filter by author, you might end up with 12 posts. Filtering **does not change the order by itself**.

For a tiny in-memory array:

```js
const posts = [
  { title: 'React', author: 'Amina', tags: ['frontend'] },
  { title: 'Node', author: 'Moussa', tags: ['backend'] },
]
const results = posts.filter(post => post.author === 'Amina')
// [{ title: 'React', author: 'Amina', tags: ['frontend'] }]
```

### Exact match versus substring match

```js
post.author === 'Amina'                  // exact and case-sensitive
post.author.toLowerCase().includes('am') // substring, case-insensitive
```

**SOURCE PROJECT:** MongoDB receives `{ author }`, so its author filter is an **exact equality query**, not the substring search shown in the lab. Do not promise a partial search on the original backend without implementing it.

### One tag versus multiple tags

**SOURCE PROJECT:** the backend constructs a MongoDB query equivalent to:

```js
{ tags: { $in: tags.split(',') } }
```

`$in` matches a post when one of its tags is among the requested values (**OR across tags**). This source code does **not** trim each comma-separated input before querying. A request with `tags=react, node` may therefore look for `' node'` (including a leading space), which may not match a stored `'node'` tag.

**IMPROVEMENT:** normalize commas and whitespace on both client and server before constructing the query.

## 5. Combining filter conditions: understand the API contract

There are several legitimate filter contracts. Never silently assume which one the backend implements.

| Request | Actual Chapter 4 API | Independent lab |
|---|---|---|
| `?author=Amina` | Exact author match | Case-insensitive substring author match |
| `?tags=react,node` | At least one matching tag | At least one matching tag |
| `?author=Amina&tags=react` | **400 Bad Request** | Both conditions applied (AND) |
| `?sortOrder=ascending` | Sort ascending (default field `createdAt`) | Sort ascending (default field `createdAt`) |

If the real UI leaves both input fields populated, its GET request can return **400**, which the current `getPosts` function incorrectly parses as JSON without checking `response.ok`.

### Fix option A — UI enforces a single active filter

**IMPROVEMENT; not in source.** When selecting a nonempty author, clear tags; when selecting tags, clear author. Example:

```jsx
<PostFilter field="author" value={author}
  onChange={value => { setAuthor(value); if (value) setTags('') }} />
<PostFilter field="tags" value={tags}
  onChange={value => { setTags(value); if (value) setAuthor('') }} />
```

**Alternative B:** change the real backend contract to support combinations and add tests. The standalone lab chooses this alternative explicitly; it must not be confused with the actual backend.

## 6. Sorting explained: choosing result order

**Sorting** rearranges items using a chosen property. Common fields:

- `createdAt`: when the post was first created.
- `updatedAt`: when the post was most recently modified.
- `ascending`: oldest first if the field is a timestamp.
- `descending`: newest first if the field is a timestamp.

In plain JavaScript, a safe numeric/date comparator can look like:

```js
const oldestFirst = [...posts].sort((a, b) =>
  new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime(),
)
```

The spread syntax `[...]` prevents mutating your original `posts` array with `.sort()`. For ISO 8601 UTC timestamps of consistent format, lexical comparison is an efficient alternative.

### Sorting happens on the backend in our real app

**SOURCE PROJECT** (`backend/src/services/posts.js`):

```js
const sortValue = sortOrder === 'ascending' ? 1 : -1
const sort = { [sortBy]: sortValue }
return await Post.find(query).sort(sort)
```

The default `sortBy` is `'createdAt'` and default direction is `'descending'`. The MongoDB driver interprets `1` as ascending, `-1` as descending. The frontend currently shows two choices: `createdAt` and `updatedAt`.

### Validate sort fields on the server

**IMPROVEMENT:** the source backend builds a dynamic field from user input without allowlisting; restrict allowed sorting fields server-side, not only in a dropdown:

```js
const SORT_FIELDS = new Set(['createdAt', 'updatedAt'])
const safeField = SORT_FIELDS.has(sortBy) ? sortBy : 'createdAt'
const direction = sortOrder === 'ascending' ? 1 : -1
return Post.find(query).sort({ [safeField]: direction, _id: direction })
```

The `_id` tie-breaker gives deterministic ordering when multiple posts have the same timestamp. It is a **recommendation**; the source has no such tie-breaker.

## 7. Controlled filter and sort components

A **controlled component** receives the current value through props and notifies a parent using a callback. The input is not a separate source of truth.

```jsx
export function TextFilter({ label, value, onChange }) {
  const id = `filter-${label.toLowerCase()}`
  return (
    <label htmlFor={id}>
      {label}
      <input
        id={id}
        value={value}
        onChange={event => onChange(event.target.value)}
      />
    </label>
  )
}
```

The component does **not** decide how to search MongoDB; it only reports user interactions. You can reuse it for author, title, tags, or another string field.

### Sorting controls

```jsx
const fields = ['createdAt', 'updatedAt']
<select value={sortBy} onChange={e => setSortBy(e.target.value)}>
  {fields.map(field => <option key={field} value={field}>{field}</option>)}
</select>
<select value={sortOrder} onChange={e => setSortOrder(e.target.value)}>
  <option value="descending">Newest first</option>
  <option value="ascending">Oldest first</option>
</select>
```

**Source note:** `PostSorting.jsx` already uses controlled selects, but declares `field` in its `propTypes` where it actually receives `fields`. A correct definition would use `fields: PropTypes.arrayOf(PropTypes.string).isRequired`. The source imports `React` but need not use that import explicitly with the modern JSX transform.

## 8. Custom hooks: extract behavior without extracting UI

A React **custom hook** is a reusable function beginning with `use` that may call other hooks. It can share **logic**, but each invocation maintains its own state unless deliberately connected to shared external state.

Examples: `useDebounce`, `usePostFilters`, `useOnlineStatus`. A `usePostFilters` hook can manage filter state while a `Filters` component renders the controls.

Do **not** call hooks inside loops, event handlers, nested functions, conditions, or after early returns. Always call them at the top level of a React component or another custom hook.

### A focused hook from our real Chapter 4

```jsx
import { useState, useEffect } from 'react'

export function useDebounce(value, delay = 3000) {
  const [debouncedValue, setDebouncedValue] = useState(value)
  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])
  return debouncedValue
}
```

**SOURCE PROJECT:** this is the actual hook logic. The `useEffect` cleanup clears the previous timer whenever `value` or `delay` changes, and on unmount. It avoids an obsolete scheduled update. It does **not** stop the `setAuthor` call or delay showing typed characters in the input: only the *derived debounced value* updates later.

### Timeline: why debounce helps

```text
Time (ms)       0      80     160                      510
Input value     'r'    're'   'rea'                     'rea'
New timer       start  reset  reset                    complete
Debounced value ''     ''     ''                       'rea'
API request     no     no     no                       one request
```

For the lab we use **350 ms** for faster interaction. The source uses **3000 ms**. Debouncing is useful for typing, but usually not needed when clicking a sort dropdown.

### Why the cleanup is necessary

Without `return () => clearTimeout(handler)`, old timers could still fire after new characters arrive. You would see multiple delayed updates. React's development Strict Mode also deliberately exercises effect cleanup, so your effect should tolerate setup/cleanup cycles.

**Important:** React's `useEffect` is for synchronizing with external systems. Avoid creating unnecessary effects for purely synchronous calculations (e.g., determining if filters are empty).

## 9. Extract a reusable `usePostFilters` custom hook

**INDEPENDENT LAB; not currently in the Chapter 4 repo.** This combines related input state and debounced query values:

```jsx
import { useState } from 'react'
import { useDebounce } from './useDebounce.js'

const INITIAL = {
  author: '', tags: '', sortBy: 'createdAt', sortOrder: 'descending',
}

export function usePostFilters() {
  const [filters, setFilters] = useState({ ...INITIAL })
  const author = useDebounce(filters.author, 350)
  const tags = useDebounce(filters.tags, 350)

  function change(field, value) {
    if (!['author', 'tags', 'sortBy', 'sortOrder'].includes(field)) return
    setFilters(prev => ({ ...prev, [field]: value }))
  }

  function reset() { setFilters({ ...INITIAL }) }

  return {
    filters,                         // immediate input values
    queryFilters: { ...filters, author, tags }, // delayed network values
    change,
    reset,
  }
}
```

Notice the two separate outputs: `filters` keeps inputs responsive; `queryFilters` is sent to the server. The lab's implementation uses a shared `INITIAL_FILTERS` from `src/lib/filters.mjs`.

> **Trade-off:** Debouncing both inputs lets the user briefly see newly typed text while the old query results are still displayed. Show a subtle refreshing indicator and, if needed, a message that the current results correspond to the previous search until the new query finishes.

## 10. Encode filters safely with URLSearchParams

Request query strings use encoded key/value pairs:

```text
GET /api/v1/posts?author=Amina&sortBy=createdAt&sortOrder=descending
GET /api/v1/posts?tags=react%2Capi&sortBy=updatedAt&sortOrder=ascending
```

Always encode user inputs; do not concatenate raw characters such as spaces, `&`, `#`, or `?` into a URL.

```js
const params = new URLSearchParams({
  author: 'Amina & friends',
  sortBy: 'createdAt',
  sortOrder: 'descending',
})
console.log(params.toString())
// author=Amina+%26+friends&sortBy=createdAt&sortOrder=descending
```

**SOURCE PROJECT:** `getPosts` already uses `new URLSearchParams(queryParams)`. It currently **does not check `res.ok`** before `res.json()`. That means an HTTP 400 response with JSON can be mistaken for successful query data.

### A safer API helper (improvement)

```js
const API_BASE = import.meta.env.VITE_BACKEND_URL

export async function getPosts(filters, { signal } = {}) {
  const params = new URLSearchParams(filters)
  const response = await fetch(`${API_BASE}/posts?${params}`, { signal })
  if (!response.ok) {
    throw new Error(`Unable to list posts: HTTP ${response.status}`)
  }
  return await response.json()
}
```

TanStack Query needs a rejected promise (or a thrown error) to set `isError`. Because native `fetch()` normally resolves for HTTP 4xx/5xx, checking `response.ok` is essential.

## 11. Connect debounced filters to a TanStack Query cache key

Each unique filter/sort combination represents a potentially different dataset. Put changing request parameters inside the query key:

```jsx
const { filters, queryFilters, change, reset } = usePostFilters()

const query = useQuery({
  queryKey: ['posts', queryFilters],
  queryFn: ({ signal }) => getPosts(queryFilters, { signal }),
  staleTime: 15_000,
})
```

**Why?** If you only used `queryKey: ['posts']`, the cache would not distinguish requests for different authors, tags, or sorting orders; changing filters might not fetch the dataset you expect.

TanStack Query computes query identity from the key contents. An object in a key can safely represent serializable filters. Avoid mutable, nonserializable values or secrets in keys.

### Proper loading, refreshing, error and empty states

```jsx
{query.isPending && <p>Loading posts...</p>}
{query.isFetching && <p>Updating results...</p>}
{query.isError && <p role="alert">{query.error.message}</p>}
{query.isSuccess && query.data.length === 0 && <p>No matches.</p>}
{query.isSuccess && query.data.map(post => (
  <article key={post._id}><h3>{post.title}</h3></article>
))}
```

`isPending` indicates the query has no successful data yet; `isFetching` indicates a request is in flight, including background refreshes. The same query may have stale data on screen while refreshing.

### Cancellation with AbortSignal

TanStack Query passes `signal` into the query function context. Forwarding it to `fetch` allows cancellation when the request becomes irrelevant, reducing wasted work. Cancellation behavior depends on consuming the signal and on the query lifecycle.

## 12. Source behavior versus the independent lab

This is especially important because the lab deliberately fixes some rough edges to make the examples clear.

| Feature | SOURCE PROJECT | INDEPENDENT LAB |
|---|---|---|
| Author filter | Exact equality in MongoDB | Case-insensitive substring |
| Multiple tags | `$in` / OR, no trimming in backend | OR with whitespace normalization |
| Author + tags simultaneously | HTTP **400** | Allowed; author AND any tag |
| Sorting fields | Dynamic key from request | Allowlist: `createdAt`, `updatedAt` |
| Equal timestamp ordering | No explicit tie-breaker | Numeric-aware `_id` tie-breaker |
| Debounce | Author only, **3000 ms** | Author and tags, **350 ms** |
| Database | MongoDB / Mongoose | In-memory seeded array |
| Query error handling | GET `res.json()` without `res.ok` check | Throws an error if `!response.ok` |
| HTTP API | Express v5, CORS middleware | Node.js native HTTP, simple read-only API |

These differences are **intentional educational choices**. None have been silently applied to the Chapter 4 GitHub project.

## 13. Independent lab: project map

Download the ZIP provided alongside this cheatsheet. It contains the PDF, this Markdown file, and the following independent working files:

```text
lab/
├── package.json
├── index.html
├── vite.config.js
├── .env.example
├── README.md
├── server/
│   ├── server.mjs
│   └── server.test.mjs
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── api.js
    ├── styles.css
    ├── components/
    │   └── Filters.jsx
    ├── hooks/
    │   ├── useDebounce.js
    │   └── usePostFilters.js
    └── lib/
        ├── filters.mjs
        └── filters.test.mjs
```

The API serves a set of **four preloaded posts**. There is no authentication, persistence, or MongoDB connection. This is a focused **read-only** exercise about lists and filtering, not a replacement for the CRUD lab from Cheatsheet 17.

## 14. Run the independent lab on Windows

In the `lab/` directory after extraction:

```powershell
# Verify tools
node --version
npm.cmd --version

# Install only the frontend React/Vite dependencies
npm.cmd install

# Check the helper logic and the HTTP API (does not require MongoDB)
npm.cmd test
```

The test suite has **14 tests**: eight pure JavaScript helper checks and six HTTP checks. If Node.js supports native test runner and built-in `fetch()`, these tests can run without downloading React dependencies; `npm.cmd install` is only needed for Vite and the React UI.

Run these in separate terminals **inside `lab/`**:

```powershell
# Terminal A: starts the stand-alone API on port 3001
npm.cmd run api

# Terminal B: starts Vite on port 5173
Copy-Item .env.example .env
npm.cmd run dev
```

Visit `http://127.0.0.1:5173` in your browser. The backend URL is `http://127.0.0.1:3001/api/v1/posts`. On Windows, prefer `npm.cmd` in PowerShell if the `npm.ps1` execution policy is restrictive; there is no need to weaken your PowerShell policy just to run npm commands.

### Expected manual exercises

1. Enter `Amina`. After approximately **350 ms**, two posts should match.
2. Type `react` under tags while `Amina` remains selected: both Amina/React posts should match.
3. Replace tags with `api`: only **React with REST APIs** should match.
4. Choose `updatedAt` descending: the post last modified most recently should appear first among matches.
5. Enter `nobody`: the list should say that there are no matches, not crash.
6. Click **Reset filters**: all four posts return.
7. Open F12 → Network: inspect the `/posts?...` URL and response status.
8. Stop terminal A and retry: React should show a request error, not an unhandled exception.
9. With dependencies installed, check `npm.cmd run build`.

### Why there are two terminals

**Vite** runs the frontend development server; the separate **Node.js API** answers HTTP requests. React cannot respond to `/api/v1/posts` without an API service. The `.env` example tells the frontend where to send the requests.

## 15. Understand the lab's test cases

| Verification | What should happen |
|---|---|
| Whitespace normalization | `" react, api "` becomes `"react,api"` |
| Unknown sorting field | Falls back to `createdAt` |
| Encoded author | `Amina & friends` round-trips safely through `URLSearchParams` |
| Case-insensitive author | `AMINA` matches `Amina` |
| Two tags | `api,node` means **either** tag |
| Combined conditions | Author AND (at least one tag) |
| Ascending timestamps | Earlier dates come first |
| Immutability | Input post array is not rearranged |
| Timestamp ties | `_id` resolves equal timestamps deterministically |
| HTTP filter | GET `/api/v1/posts?author=amina&tags=api` returns the expected post |
| HTTP sorting | GET with `updatedAt` changes first result |
| Empty result | HTTP 200 with `[]` |
| Missing route | HTTP 404 |
| CORS preflight | OPTIONS returns 204 with CORS headers |

**Test boundary:** these Node.js tests verify the pure helper functions and the lab API. They do **not** test actual React rendering, `useDebounce` timers inside a browser, the Vite build, or MongoDB. Test those separately on your machine.

### Optional: a stronger React test suite

For an advanced extension, install Vitest + React Testing Library + a browser-like environment such as jsdom (versions compatible with your stack). Use fake timers to verify `useDebounce` waits the specified delay and cancels earlier timers. This **is an improvement exercise**, not something already included in the source or proven by the 14 tests above.

## 16. Security, performance and accessibility

- **Server-side allowlist:** validate `sortBy` and direction again in the backend. A React `<select>` does not prevent malicious or malformed external HTTP requests.
- **Input validation:** reject or limit extremely long filters, normalize whitespace, and define whether search is case-sensitive.
- **Database safety:** index common filter/sort combinations for large datasets. Query plans and indexes are database concerns, not frontend concerns.
- **Pagination:** avoid transferring thousands of records merely to filter them in the browser. Page and limit belong in both URL/query key and backend validation.
- **Safe interpolation:** React escapes text content in JSX by default; never substitute unsafe HTML rendering for ordinary post strings.
- **CORS:** the lab uses `Access-Control-Allow-Origin: *` only because it exposes a **public read-only** demo API. Production authenticated APIs require an explicit origin/cookie policy.
- **Vite environment:** variables beginning with `VITE_` are publicly visible in the built frontend. Never place database passwords or secrets there.
- **Accessibility:** associate a `label` and input with `htmlFor` and `id`, use native `<select>`, and expose loading/error messages with `role="status"` / `role="alert"` as appropriate.
- **Network traffic:** debounce free-text filters; avoid debouncing dropdowns without a reason. Prefer cancellation signals for obsolete queries.
- **Deterministic UX:** provide a clear “Reset filters” button, show active filters, distinguish empty from error, and avoid flickering while new results load.

## 17. Troubleshooting: symptoms, causes, fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Input changes but list does not | `queryKey` omits changing filter | Include filters in `['posts', filters]`. |
| Search updates only after 3 s | Source hook uses default `delay = 3000` | Pass a smaller delay such as `useDebounce(author, 350)`. |
| Every character triggers a GET | Text input passed directly to query key | Debounce the relevant filter, not the visible input. |
| `TypeError: posts.map is not a function` | HTTP 400 JSON error parsed as successful list | Check `response.ok` and throw; resolve author+tags contract mismatch. |
| Author + tags returns 400 on real API | Backend explicitly rejects both | Clear one filter or implement combined queries on backend. |
| Tag search misses `node` | Request sends `' node'` with whitespace | Trim tags on server before `$in`; normalize client input. |
| Sorting is unexpected | Invalid sort field or two identical timestamps | Server allowlist, deterministic secondary field, inspect data. |
| `PropTypes` missing | Imported but undeclared package in root | `npm.cmd install prop-types` locally. |
| `DeletePostButton is not defined` | Typo in real `DeletePost.jsx` | Change `DeletePostButton.propTypes` to `DeletePost.propTypes`. |
| UI says CORS or failed fetch | API is stopped or origin not permitted | Start API separately; check ports and CORS response. |
| Browser URL is `undefined/posts` | Missing `VITE_BACKEND_URL` | Set URL with `/api/v1`, restart Vite. |
| Hooks error | Hook called conditionally or outside React | Move hook call to top level of component/custom hook. |
| Dev Effect seems to run twice | Strict Mode development lifecycle | Keep cleanup symmetrical; avoid removing Strict Mode as a workaround. |
| `npm.ps1 cannot be loaded` | PowerShell execution policy | Use `npm.cmd` rather than changing policy. |
| UI renders but shows zero results | Input matched no data, or tag contract differs | Inspect Network tab and query params, then check API response JSON. |

## 18. Quick reference: reusable patterns

| Task | Pattern |
|---|---|
| Controlled text filter | `<input value={author} onChange={e => setAuthor(e.target.value)} />` |
| Debounced value | `const debouncedAuthor = useDebounce(author, 350)` |
| Reusable state logic | `const { filters, change, reset } = usePostFilters()` |
| Sort direction | `'ascending'` / `'descending'` |
| Sort field allowlist | `['createdAt', 'updatedAt'].includes(sortBy)` |
| Build query params | `new URLSearchParams({ author, sortBy, sortOrder })` |
| Error-aware GET | `if (!response.ok) throw new Error(...)` |
| Query cache key | `['posts', { author, tags, sortBy, sortOrder }]` |
| Cancel obsolete requests | `queryFn: ({ signal }) => fetch(url, { signal })` |
| Empty result state | `query.isSuccess && query.data.length === 0` |
| Reset filters | `setFilters({ ...INITIAL_FILTERS })` |
| Test pure logic | `node --test src/lib/filters.test.mjs` |
| Test HTTP API | `node --test server/server.test.mjs` |

### Independent practice challenge

Implement one extra filter: `title`. Before changing code, write down the contract: exact match or substring? Case-sensitive or insensitive? May it be combined with author and tags? Then update the custom hook, query key, API, backend helper, and tests **together**. Next, add pagination and design how page number is reset when filters change.

### Reuse checklist

- [ ] Node.js, npm and Git work in PowerShell.
- [ ] The frontend and backend are started in distinct terminals.
- [ ] API base URL includes the correct `/api/v1` prefix.
- [ ] Controlled inputs reflect local state immediately.
- [ ] Free-text queries use a suitable debounce delay.
- [ ] Timers have cleanup functions.
- [ ] Custom hooks are only called at valid top-level locations.
- [ ] Query keys represent every changing server-side parameter.
- [ ] Query functions throw when HTTP responses are not successful.
- [ ] Combined filter behavior matches the backend contract.
- [ ] Sorting fields and directions are validated on the server.
- [ ] Lists distinguish loading, refreshing, empty and error states.
- [ ] URL parameters are encoded safely.
- [ ] I ran the lab tests and checked the browser Network tab.
- [ ] I did not commit secrets or expose private keys through Vite.
- [ ] Source features and optional improvements are clearly distinguished.

## 19. References, source notes, and what comes next

**Code inspected (GitHub, 20 October 2026):** `src/Blog.jsx`, `src/components/PostFilter.jsx`, `src/components/PostSorting.jsx`, `src/hooks/useDebounce.js`, `src/api/posts.js`, `backend/src/routes/posts.js`, `backend/src/services/posts.js`, and root `package.json` from [XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development).

**Official references:**

- [React — useState](https://react.dev/reference/react/useState)
- [React — useEffect and effect cleanup](https://react.dev/reference/react/useEffect)
- [React — Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
- [TanStack Query v5 — Query Keys](https://tanstack.com/query/latest/docs/framework/react/guides/query-keys)
- [TanStack Query v5 — Query Functions](https://tanstack.com/query/latest/docs/framework/react/guides/query-functions)
- [TanStack Query v5 — Query Cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation)
- [MDN — URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)
- [Mongoose — Queries](https://mongoosejs.com/docs/queries.html)
- [Vite — Environment Variables](https://vite.dev/guide/env-and-mode)

**Testing record:** 14 native Node.js tests passed for the independent lab (8 helper tests and 6 HTTP tests). React browser rendering, Vite compilation and live MongoDB integration were not executed by this guide's author; perform those checks locally before treating the project as production-ready.

**Chapter milestone:** Cheatsheets 14–18 complete the **Frontend Development** learning block. Next: **Cheatsheet 19 — Docker Fundamentals & Installation**, opening **Chapter 05: DevOps**.

---

*Learn → Practice → Document → Reuse.*