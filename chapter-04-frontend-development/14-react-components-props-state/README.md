# Cheatsheet 14 — React Components, JSX, Props & State

**From Zero to Fullstack · Chapter 4: Frontend Development**  
**Audience:** Absolute beginners, self-taught React learners and developers starting new frontend projects  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code · Node.js  
**Version:** 1.0 · 19 October 2026  
**Source project:** [Chapter 4 — Frontend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development)  
**Method:** **Learn → Practice → Document → Reuse**

> **Accuracy note.** **SOURCE PROJECT** means a file checked directly in the Chapter 4 GitHub repository. **INDEPENDENT LAB** means the new, standalone React example supplied inside this guide's ZIP. **IMPROVEMENT** means a best-practice recommendation that is not claimed to be implemented in the source. Five tests of the lab's plain-JavaScript helper functions passed in Node.js. The complete React/Vite browser application was **not compiled or run here** because the npm cache lacks its dependencies; the guide includes steps to verify it locally.

## 1. Learning goals and the big picture

After completing this cheatsheet, you will know how to:

1. Install the tools needed for a React/Vite project and open an existing repository.
2. Explain what a **component**, **JSX**, **prop**, **state**, **event handler** and **render** are.
3. Read the actual component structure in our Chapter 4 blog application.
4. Build reusable UI elements with function components and JSX.
5. Pass data **down** through props and pass event callbacks **up**.
6. Use `useState` to update a component after user interaction.
7. Build controlled inputs and forms.
8. Render lists safely using stable `key` values and render conditional content.
9. Update arrays and objects **immutably** rather than changing existing state.
10. Understand the difference between **local UI state** and **server state** (introduced in detail in Cheatsheet 16).
11. Verify behavior, troubleshoot common errors and reuse a small React starter in a new project.

### React in one mental model

```text
A component is a JavaScript function that describes UI using JSX.

                         <App />            owner of shared state
                            |
               +------------+--------------+
               |                           |
           <PostForm />                <PostList />
               |                           |
     onAdd(title, author)             <PostCard /> (repeated)
               |                           |
               +-----> App state <----- onRemove(id)
                          |
                  React renders again
```

**Props are inputs received from a parent. State is data a component remembers between renders.** When state changes using its setter, React renders again to update the screen.

| Concept | Think of it as | Example |
|:--|:--|:--|
| Component | Reusable UI building block | `PostList` or `Post` |
| JSX | Markup-like syntax inside JavaScript | `<h2>{title}</h2>` |
| Props | Read-only inputs from a parent | `<Post title="Hello" />` |
| State | Component memory | `const [count, setCount] = useState(0)` |
| Event handler | Function called after an interaction | `onClick={handleClick}` |
| Render | React calculates UI for current props/state | A new post appears |

## 2. Prepare your Windows environment from zero

Install these tools from their official download pages:

- **Node.js:** <https://nodejs.org/en/download> — choose a supported LTS release. Vite 7 needs **Node.js 20.19+ or 22.12+**.
- **Git:** <https://git-scm.com/downloads>
- **Visual Studio Code:** <https://code.visualstudio.com/>
- **Browser:** a current version of Chrome, Firefox or Edge.

Open the VS Code terminal (**Terminal → New Terminal**) and choose PowerShell:

```powershell
node --version
npm.cmd --version
git --version
code --version
```

Expected: a version number for each tool. If `code` is not available in your terminal, start VS Code from the Windows Start menu and use **File → Open Folder**. If PowerShell complains that `npm.ps1` is not digitally signed, use **`npm.cmd`**, as in all commands below, instead of globally weakening execution policies.

### Clone our Chapter 4 source repository

```powershell
New-Item -ItemType Directory -Force "$HOME\projects" | Out-Null
Set-Location "$HOME\projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development.git
Set-Location .\from-zero-to-fullstack-ch4-frontend-development
Get-ChildItem
npm.cmd ci
```

**SOURCE PROJECT:** Chapter 4 uses **two separate npm projects**, not a monorepo workspace: the React/Vite frontend is in the repository root and the Express/Mongoose backend is in `backend/`. To install backend dependencies independently:

```powershell
Set-Location .\backend
npm.cmd ci
Set-Location ..
```

For this cheatsheet we focus on **components and local state**. A running MongoDB database or backend is **not required for the separate lab**. To run the **full Chapter 4 application**, follow the setup and API URL instructions in [Cheatsheet 15 — Connecting React to a REST API](https://github.com/XMAN-MHD/from-zero-to-fullstack-cheatsheets/blob/main/chapter-04-frontend-development/15-react-rest-api/README.md).

### Important source-project startup caveats

The Chapter 4 frontend imports `PropTypes` from `prop-types` in several files, but its checked root `package.json` does **not** declare this dependency. If Vite reports that it cannot resolve `prop-types`, install it in the frontend root:

```powershell
npm.cmd install prop-types
```

Also, **SOURCE PROJECT** `src/components/DeletePost.jsx` ends with `DeletePostButton.propTypes`, but the actual function is `DeletePost`. The reference should be `DeletePost.propTypes`. Additionally, `src/components/PostSorting.jsx` declares `field` instead of `fields` in its PropTypes mapping. Correct these **in your local copy if needed**; this guide does **not** claim these fixes have been committed. Check `npm.cmd run lint` and `npm.cmd run build` before trusting a cloned project. Do not commit existing `.env` files or other credentials; review repository history if sensitive values have ever been tracked.

## 3. Understand the real React component structure

The following files were read directly from the **SOURCE PROJECT**:

```text
src/
├── main.jsx                    # React root; StrictMode
├── App.jsx                     # QueryClientProvider around Blog
├── Blog.jsx                    # page composition, local filters, useQuery
├── api/posts.js                # HTTP requests (Cheatsheet 15)
├── components/
│   ├── PostList.jsx            # maps posts into Post components
│   ├── Post.jsx                # post details, local edit toggle
│   ├── PostFilter.jsx          # controlled filter input
│   ├── PostSorting.jsx         # sorting selectors
│   ├── CreatePost.jsx          # form state and create mutation
│   ├── EditPost.jsx            # editable fields and update mutation
│   └── DeletePost.jsx          # confirmation state and delete mutation
└── hooks/useDebounce.js        # delayed local filter value
```

**Entry point — `src/main.jsx`** renders `<App />` inside `<StrictMode>`. **`src/App.jsx`** wraps `<Blog />` with a `QueryClientProvider`, which supplies TanStack Query. **`src/Blog.jsx`** composes the create form, filters, sorting selector and list. The source uses `useQuery` for server data; we will analyze that in **Cheatsheet 16**.

### SOURCE PROJECT: list rendering and props

The real `src/components/PostList.jsx` is conceptually:

```jsx
export function PostList({ posts = [] }) {
  return (
    <div>
      {posts.map((post) => (
        <Fragment key={post._id}>
          <Post {...post} />
        </Fragment>
      ))}
    </div>
  )
}
```

Here, `posts` arrives as a prop, `.map()` creates one UI element for each post, `key={post._id}` gives each list entry a stable identity, and `{...post}` spreads the object's properties into the child `Post` props. A short form is `<Post key={post._id} {...post} />`; that is an **explanatory alternative**, not the exact source markup.

**SOURCE PROJECT** `Post.jsx` destructures its props as `({ _id, title, contents, author, tags })`, shows title/author/tags and keeps its own `isEditing` state. `CreatePost.jsx` holds each form field in local state. `Blog.jsx` owns the filter states and passes setters to `PostFilter`.

## 4. JSX: how JavaScript describes the interface

JSX looks similar to HTML but lives inside JavaScript. For a beginner, remember these rules:

```jsx
function Welcome({ name }) {
  const today = new Date().toLocaleDateString()
  return (
    <section className="welcome">
      <h2>Hello, {name}!</h2>
      <p>Today is {today}.</p>
      <label htmlFor="username">Your name</label>
      <input id="username" />
    </section>
  )
}
```

- Use **`className`** instead of HTML's `class` and **`htmlFor`** instead of `for`.
- Put JavaScript **expressions** inside `{ ... }`: `name`, `items.length`, `isReady ? 'Yes' : 'No'`. Statements such as a full `if` block cannot go directly inside JSX braces.
- A component must return one root tree; wrap sibling nodes in `<div>...</div>` or an empty fragment `<>...</>`.
- Self-close empty elements: `<input />`, `<img alt="..." src="..." />`.
- Component names start with a **capital letter**; lowercase JSX tags represent HTML elements.
- An event prop receives a **function**: `onClick={handleClick}`. Writing `onClick={handleClick()}` calls it immediately during render.
- Most browser DOM events use camelCase names such as `onChange`, `onClick` and `onSubmit`.

### Reusable component example

```jsx
function Badge({ children, variant = 'normal' }) {
  return <span className={`badge badge--${variant}`}>{children}</span>
}

function Profile() {
  return (
    <div>
      <Badge variant="success">Available</Badge>
      <Badge>Learning React</Badge>
    </div>
  )
}
```

`children` is an ordinary prop filled by whatever appears between opening and closing component tags. `variant` has a default value. A component may return different JSX depending on its props.

## 5. Props: passing information from parent to child

**Props are read-only.** A parent creates a child with a value, and the child receives it. A component should not directly change the prop or any nested object passed to it.

```jsx
function PostSummary({ title, author = 'Anonymous' }) {
  return (
    <article>
      <h3>{title}</h3>
      <p>By {author}</p>
    </article>
  )
}

function App() {
  return <PostSummary title="Learning JSX" author="Awa" />
}
```

This example uses default parameters, destructuring and readable markup. **Destructuring** means extracting values from the props object directly in the function signature.

### Passing callbacks upward

The data direction is **parent → child via props**. To let a child ask its parent to change something, the parent passes a function as a prop:

```jsx
function FilterInput({ value, onChange }) {
  return (
    <label>
      Search
      <input value={value} onChange={(event) => onChange(event.target.value)} />
    </label>
  )
}
```

**SOURCE PROJECT `PostFilter.jsx`** uses precisely this interaction pattern: `Blog.jsx` passes `value={author}` and `onChange={setAuthor}`; the child calls the callback with the new text. It **does not modify props**.

## 6. State: a component's memory with `useState`

A normal local variable resets when a function renders again and changing it does **not** ask React to update the UI. React state persists between renders and its setter schedules an update.

```jsx
import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>Count: {count}</p>
      <button type="button" onClick={() => setCount((current) => current + 1)}>
        Increment
      </button>
    </div>
  )
}
```

`count` is the current state snapshot; `setCount` requests the next state. The **functional update** `setCount((current) => current + 1)` is best when the new value depends on the previous value; React may batch multiple updates. **Do not expect a log immediately after `setCount()` in the same event handler to show the new value.**

### SOURCE PROJECT: editing a post

```jsx
const [isEditing, setIsEditing] = useState(false)

if (isEditing) {
  return <EditPost post={post} onClose={() => setIsEditing(false)} />
}

return <button onClick={() => setIsEditing(true)}>Edit</button>
```

This is a **shortened explanation** of the real `Post.jsx` code. The actual component passes an object with `_id`, `title`, `contents`, `author` and `tags` and also displays the post, author and controls. The important idea is that one state boolean changes which interface gets rendered.

### Hooks: rules you must follow

- Call `useState`, `useEffect` and other hooks **only at the top level** of a React function component or custom hook.
- Never call hooks inside loops, conditions or ordinary event callbacks.
- Component functions must be used as JSX (`<Counter />`) rather than called directly as `Counter()`.
- In development, React **StrictMode** may render components or run effect setups more than once to help expose bugs. Do not confuse this with a production double-submit; move side effects out of rendering and make effects safe.

## 7. Forms and events: controlled inputs

A **controlled input** gets its displayed value from React state and informs React about edits through an event handler:

```jsx
function PostTitleForm({ onAdd }) {
  const [title, setTitle] = useState('')

  function handleSubmit(event) {
    event.preventDefault() // stop the browser from reloading the page
    const cleanTitle = title.trim()
    if (!cleanTitle) return
    onAdd(cleanTitle)
    setTitle('')
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="post-title">Title</label>
      <input
        id="post-title"
        value={title}
        onChange={(event) => setTitle(event.target.value)}
        required
      />
      <button type="submit" disabled={!title.trim()}>Add post</button>
    </form>
  )
}
```

**SOURCE PROJECT `CreatePost.jsx`** uses local `title`, `author`, `tags` and `contents` states with `value` and `onChange`. Unlike this small example, the real project sends the form to the backend through TanStack Query's `useMutation` and invalidates cached post queries on success.

**Practical form rules:** keep related labels and input IDs aligned, disable invalid submissions, show pending/error states for asynchronous operations, and never trust browser-only validation for security (validate again in the backend).

## 8. Lists, keys and conditional rendering

**List:**

```jsx
function Names({ names }) {
  return (
    <ul>
      {names.map((person) => <li key={person.id}>{person.name}</li>)}
    </ul>
  )
}
```

Use **stable unique IDs** for keys when list items can be inserted, deleted or reordered. Array index keys can produce confusing UI behavior when order changes. `key` is used by React for reconciliation; a child component **does not receive it as a normal prop**. Pass the ID separately when needed.

**Conditions:**

```jsx
if (loading) return <p role="status">Loading...</p>
if (error) return <p role="alert">Something went wrong.</p>
if (posts.length === 0) return <p>No posts available.</p>
return <PostList posts={posts} />
```

For small conditions, use ternary (`isOpen ? <Menu /> : null`) or `&&` (`isOpen && <Menu />`). Beware the `0 && <div />` trap, which renders `0` in React; write `count > 0 && <div />` instead.

## 9. Immutable state updates and lifting state up

Never mutate state arrays or objects in place:

```jsx
// Avoid: mutates the existing array and may not trigger the expected update.
posts.push(newPost)
setPosts(posts)

// Prefer: create a new array.
setPosts((current) => [newPost, ...current])

// Delete one item without mutating the old array.
setPosts((current) => current.filter((post) => post._id !== id))

// Update one record while preserving others.
setPosts((current) => current.map((post) =>
  post._id === id ? { ...post, title: 'Updated title' } : post
))
```

**Lifting state up** means placing shared state in the **closest common parent** of two or more components that need it. In our **INDEPENDENT LAB**, `App` owns `posts` and `authorQuery`, sends the filtered data to `PostList`, and passes `onAdd`/`onRemove` callbacks to its children. This keeps a single source of truth.

### Local UI state vs server state

| Question | Local state (`useState`) | Server state (TanStack Query) |
|:--|:--|:--|
| Where does the data originate? | Browser component | Backend/API |
| Typical example | Input value, edit modal open/closed | Posts loaded from `/api/v1/posts` |
| Who manages updates? | Component setter | Query cache, fetch and mutations |
| Does it survive a reload by itself? | No | Not reliably; cache behavior depends on setup |
| Deep dive | **This cheatsheet (14)** | **Cheatsheet 16** |

**SOURCE PROJECT** `Blog.jsx` combines both: `useState` for filters and sorting settings, and `useQuery` for data from the backend. These tools solve **different problems** rather than competing for one job.

## 10. Independent lab: build and practice without MongoDB

The ZIP includes a complete **INDEPENDENT LAB** in the `lab/` folder. It is intentionally separate from Chapter 4 source. This app needs React/Vite but no Express, TanStack Query, MongoDB or `.env` file.

### Start the lab on Windows

Extract the ZIP, then open PowerShell in the extracted `lab` directory:

```powershell
npm.cmd install
npm.cmd test
npm.cmd run dev
```

Open the local address printed by Vite (usually `http://localhost:5173`). After practising, check the production build:

```powershell
npm.cmd run build
npm.cmd run preview
```

The lab uses a standard root `index.html`, `src/main.jsx`, `src/App.jsx`, `src/components/` folder, `src/lib/posts.js` and a `vite.config.js` file. It uses versions compatible with the Chapter 4 React/Vite setup, but installs only the dependencies needed for this small example.

### Lab behavior and component tree

```text
App.jsx
├── PostForm.jsx
│   ├── title state
│   └── author state
├── author filter state
├── help-toggle state
└── PostList.jsx
    └── PostCard.jsx  (one component for every displayed post)
```

- Add a post; notice how the input fields clear on success.
- Type `AWA` in the author filter; only posts by Awa remain, case-insensitively.
- Remove a post; notice the list and post count update immediately.
- Toggle the help text; notice a conditional section appears/disappears.
- Reload the page; all changes disappear because **this example stores data only in React state**.

### Four important source snippets from the lab

**A. State owned by the parent (`src/App.jsx`):**

```jsx
const [posts, setPosts] = useState(starterPosts)
const [authorQuery, setAuthorQuery] = useState('')
const visiblePosts = filterPosts(posts, authorQuery)
```

**B. Passing callbacks to children:**

```jsx
<PostForm onAdd={addPost} />
<PostList posts={visiblePosts} onRemove={removePost} />
```

**C. A stable list key (`src/components/PostList.jsx`):**

```jsx
{posts.map((post) => (
  <PostCard key={post._id} post={post} onRemove={onRemove} />
))}
```

**D. Immutable update:**

```jsx
function removePost(postId) {
  setPosts((currentPosts) => currentPosts.filter((post) => post._id !== postId))
}
```

The full, runnable JSX is in the ZIP; these short excerpts focus on the concepts.

## 11. Tests and verification matrix

The lab's `src/lib/posts.test.js` uses **Node.js's built-in test runner**, requiring no Jest, browser or npm dependencies to check the pure-JavaScript helpers. Run in `lab/`:

```powershell
node --test .\src\lib\posts.test.js
# or: npm.cmd test
```

| Check | How | Expected result | Authoring status |
|:--|:--|:--|:--|
| Normalize whitespace in title | Node test | One space between words | **Passed** |
| Author search ignores case | Node test | 2 posts for `AWA` | **Passed** |
| Empty filter returns all posts | Node test | No filtering applied | **Passed** |
| Draft post creation | Node test | Trimmed title, default author | **Passed** |
| Blank title is rejected | Node test | Throws explanatory error | **Passed** |
| React dev server opens | `npm.cmd run dev` | Vite URL opens in browser | **Not run here** |
| React create/filter/remove UI | Browser actions | Screen updates correctly | **Not run here** |
| React production build | `npm.cmd run build` | `dist/` is created | **Not run here** |

**Test boundary:** these five Node tests verify **plain JavaScript helper functions only**, not React rendering, DOM interactions or React state. Local browser testing remains necessary. Automated UI/component tests using React Testing Library can be introduced later; they are **not included** or claimed as passed.

### Practice checklist

- [ ] I can explain the difference between components, JSX, props and state.
- [ ] I can create a component with destructured props.
- [ ] I can render an array with unique keys.
- [ ] I can build a controlled input and stop a form reload.
- [ ] I can add and delete list entries immutably.
- [ ] I know which component owns shared state in the lab.
- [ ] I can run five helper tests and inspect the results.
- [ ] I can open the React app, perform the UI checks and create a production build.

## 12. Troubleshooting (Windows, Vite and React)

| Symptom | Likely cause | What to check |
|:--|:--|:--|
| `npm.ps1 cannot be loaded` | PowerShell execution policy | Use `npm.cmd` |
| Vite says Node version unsupported | Node.js is too old | Install Node 20.19+ or 22.12+ |
| `Cannot find module 'react'` | Dependencies not installed | Run `npm.cmd install` in the **correct folder** |
| `Failed to resolve import 'prop-types'` | Missing package in source project | In source root: `npm.cmd install prop-types` |
| `DeletePostButton is not defined` | Real `DeletePost.jsx` typo | Change reference to `DeletePost.propTypes` locally |
| Blank screen / JSX compile error | Invalid JSX / module imports | Check Vite terminal and browser console |
| Input does not update | `value` given without correct handler | Pair `value` with `onChange` that calls setter |
| Clicking button does nothing | Handler not wired | Use `onClick={handleClick}`, not a string |
| React warns about missing keys | Unstable mapped list | Add `key={post._id}` on returned element |
| UI does not update after `push()` | State array mutated in place | Use spread, `.filter()` or `.map()` to return a new array |
| Unexpected repeated render logs | StrictMode development checks | Do not perform side effects during render |
| Original Chapter 4 API returns an error | Backend/URL/DB not ready | Consult Cheatsheet 15; use the independent lab for this lesson |

**Security reminders:** avoid rendering untrusted HTML via `dangerouslySetInnerHTML`; React escapes interpolated text by default. Do not expose secrets in `VITE_` variables (they are bundled into client code). Do not commit `.env` files. Never treat frontend validation as a substitute for backend validation. Keep package versions and browser dependencies updated.

## 13. Quick reference: copy this when starting a new app

```powershell
# New React project with JavaScript + Vite
npm.cmd create vite@latest my-react-app -- --template react
Set-Location .\my-react-app
npm.cmd install
npm.cmd run dev
npm.cmd run build
```

```jsx
// Component + props
function Greeting({ name = 'friend' }) {
  return <h1>Hello {name}</h1>
}

// State
const [value, setValue] = useState('')

// Controlled input
<input value={value} onChange={(e) => setValue(e.target.value)} />

// Stable list keys
{items.map((item) => <Item key={item.id} item={item} />)}

// Immutable append
setItems((previous) => [...previous, newItem])

// Immutable remove
setItems((previous) => previous.filter((item) => item.id !== removeId))
```

**Rules:** one-way data flow; call hooks at the top level; components start with capitals; props are read-only; update state with new references; use stable keys; handle keyboard access and meaningful labels; test UI behavior, not just helper functions.

## 14. Reuse checklist for a professional frontend project

- [ ] Create the project with supported Node.js and Vite versions.
- [ ] Separate reusable components from page-level orchestration.
- [ ] Keep local state close to the components that own it; lift only when sharing is needed.
- [ ] Give components descriptive names (`PostCard`, `PostForm`, `PostList`).
- [ ] Define clear and narrow props; use `children` for composition.
- [ ] Use stable IDs and immutable state updates.
- [ ] Add appropriate empty, loading and error states when data is asynchronous.
- [ ] Test keyboard navigation, labels, visible focus and responsive layout.
- [ ] Check ESLint, format with Prettier, and run the production build.
- [ ] Never commit credentials or expose secrets in browser-side variables.
- [ ] Keep business/server data requests outside purely presentational components.
- [ ] Document what is implemented vs proposed, and rerun tests after updates.

## 15. Source notes and trustworthy references

**Primary source (verified):** [Chapter 4 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development), specifically `src/main.jsx`, `src/App.jsx`, `src/Blog.jsx`, `src/components/Post.jsx`, `src/components/PostList.jsx`, `src/components/PostFilter.jsx`, `src/components/CreatePost.jsx`, `src/components/PostSorting.jsx`, and root `package.json` as checked for this guide.

**Separate practice code:** `lab/` delivered in this ZIP; it does not replace or modify the Chapter 4 source repository. The tests were run with Node.js 22.16.0; React and Vite could not be installed from the local npm cache for this authoring environment, so the lab's UI/build remain **user verification steps**, not verified results.

**Official documentation:**

- React — Quick Start: <https://react.dev/learn>
- React — Your First Component: <https://react.dev/learn/your-first-component>
- React — Passing Props: <https://react.dev/learn/passing-props-to-a-component>
- React — State: <https://react.dev/learn/state-a-components-memory>
- React — Updating Arrays in State: <https://react.dev/learn/updating-arrays-in-state>
- React — Render Lists: <https://react.dev/learn/rendering-lists>
- Vite — Getting Started: <https://vite.dev/guide/>
- Node.js — Test Runner: <https://nodejs.org/api/test.html>

---

*Learn → Practice → Document → Reuse.*  
**Next:** Cheatsheet 16 — TanStack Query from Scratch (server state, caching and mutations).  
**Publication note:** Cheatsheet 15 was published before this guide; Cheatsheet 14 closes that numbering gap.