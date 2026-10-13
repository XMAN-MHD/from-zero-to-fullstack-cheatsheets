# Cheatsheet 03 — React Project Setup with Vite

**From Zero to Fullstack · Chapter 1: Foundations**  
**Audience:** Absolute beginners + developers starting a reusable frontend project  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Version:** 1.0 · 13 October 2026  
**Source:** [Chapter 1 — Foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install prerequisites, create a React project using Vite, reproduce our actual Chapter 1 starter, understand every essential project file, run the development server, test a React state update, and produce a build suitable for deployment.

## Quick reference — which path should you choose?

| Goal | Action | Why |
| --- | --- | --- |
| **Reproduce our Chapter 1 exactly** | Clone the course repository and run `npm ci` | Uses the existing source and lockfile |
| **Start your own modern React app** | Run `npm create vite@latest my-react-app -- --template react` | Generates a new starter using current defaults |
| **Work on a downloaded project** | `npm ci`, then `npm run dev` | Does not unnecessarily rewrite the dependency lockfile |
| **Check source quality** | `npm run lint` | Runs the lint script defined by the project |
| **Create a production bundle** | `npm run build` | Generates static assets in `dist/` |
| **Preview the built output** | `npm run preview` | Serves the production build locally |

**Crucial distinction:** This chapter is a **React frontend starter**, not yet a finished full-stack MERN application. Express, MongoDB integration, API requests and Docker are covered in later chapters. The original Chapter 1 `package.json` lists `mongodb`, but its starter does not actually connect to a database.

---

## 1. Understand React, Vite, Node.js and npm

A **web browser** displays HTML/CSS, runs client-side JavaScript and renders our user interface.

- **React** is the UI library used to build reusable components that update when data changes.
- **JSX** lets us describe a React interface using HTML-like syntax inside JavaScript.
- **Vite** is our development server and build tool. It starts a local server, transforms source files and produces a production bundle.
- **Node.js** runs build and development tools on our computer; it does **not** mean that this React page requires a Node.js application server in production.
- **npm** installs dependencies and executes commands listed in `package.json`.
- **VS Code** is our editor; **Git** tracks our changes.

```text
YOUR COMPUTER                                   WEB BROWSER
VS Code → React source (JSX/CSS) → Vite dev server → http://localhost:5173/
                    |                                 ↑
                    +--- npm run build → dist/ --------+ (serve static files)
```

**Development vs production:** `npm run dev` is for local development. `npm run build` generates deployable static assets. `npm run preview` is a convenient local preview, **not** a recommended production web server by itself.

## 2. Install everything from scratch

### 2.1 Requirements

- Windows 10/11, a stable internet connection, enough space to install Node.js and npm packages, and a modern browser.
- Permission to install software; no React or programming experience is assumed.
- A compatible Node.js version. **The original Chapter 1 uses Vite `^7.1.7`, which requires Node.js `20.19+` or `22.12+`.** For new projects, confirm the requirements of the selected Vite/template version; newer templates may need a newer Node release.

### 2.2 Visual Studio Code

1. Download from [code.visualstudio.com/download](https://code.visualstudio.com/download).
2. Install the Windows User Installer using the normal setup procedure.
3. Open VS Code and its integrated terminal (**Terminal → New Terminal**).
4. Verify in a newly opened PowerShell:

```powershell
code --version
```

**Expected:** A version is printed. If `code` is not recognized, reopen PowerShell or use **File → Open Folder** in VS Code.

**Optional editor extensions:** ESLint, Prettier — Code formatter. These extensions are not substitutes for the project's npm dependencies.

### 2.3 Node.js and npm

1. Open [nodejs.org/en/download](https://nodejs.org/en/download).
2. Choose an **LTS release compatible with the Vite version you plan to run**; check the official compatibility note and the project’s `package.json` before choosing.
3. Install the Windows `.msi` package. npm is normally bundled with the Node installer.
4. Restart PowerShell and verify:

```powershell
node --version
npm --version
node -e "console.log('Ready:', 2 + 2)"
```

**Expected:** Both versions and `Ready: 4`.

**PowerShell warning:** If Windows blocks `npm.ps1` with an execution-policy error, you can run `npm.cmd` without changing the machine-wide execution policy:

```powershell
npm.cmd --version
npm.cmd run dev
```

In this guide, use `npm.cmd` instead of `npm` for **every npm command** if your PowerShell configuration requires it.

### 2.4 Git

Download Git for Windows from [git-scm.com/install/windows](https://git-scm.com/install/windows), install it and restart the terminal:

```powershell
git --version
```

GitHub authentication is **not needed** for creating and running a local Vite project. It becomes relevant when pushing your work to GitHub. See [Cheatsheet 02 — Git & GitHub](../02-git-github/README.md).

---

## 3. Path A — Create a brand-new React + Vite application

This section is a **new-project tutorial**, not a claim that the original Chapter 1 repository was produced by these exact commands.

### Step 1 — Prepare a workspace

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
Get-Location
```

**Expected:** PowerShell displays your `Projects` directory. Avoid accidentally running setup commands inside an unrelated repository.

### Step 2 — Scaffold the project

```powershell
npm create vite@latest my-react-app -- --template react
```

**Meaning:**

- `npm create vite@latest` obtains the current project scaffolding tool.
- `my-react-app` is the new folder name; choose a simple lowercase name.
- `-- --template react` passes the React JavaScript template option to the generator.
- The `react` template uses **JavaScript/JSX**. `react-ts` is a separate TypeScript option.

The interactive questions and exact generated dependency versions **may change over time**. Follow the displayed prompts; check Node compatibility if warned.

### Step 3 — Install and start

```powershell
Set-Location my-react-app
npm install
npm run dev
```

**Expected:** Vite prints a local URL, often `http://localhost:5173/`. Open the **exact URL in your terminal**; Vite may use a different port if that one is busy.

You should see a React/Vite starter. Click its counter button and verify the number changes. Stop the server with **Ctrl + C**.

### Step 4 — Open with VS Code

```powershell
code .
```

The `.` means the **current directory**. Open the generated `src/App.jsx` to see the component shown in your browser.

---

## 4. Path B — Reproduce our real Chapter 1 source

**Source-backed reconstruction:** Our published Chapter 1 is [XMAN-MHD/from-zero-to-fullstack-ch1-foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations).

```powershell
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
code .
npm ci
npm run dev
```

**Why `npm ci`?** The source project **already includes a `package-lock.json`**. `npm ci` installs dependencies using that lockfile, providing a reproducible existing-project setup. It does not attempt to solve the project's version range again as `npm install` might.

**Expected:** Vite starts; your browser shows the Chapter 1 `Vite + React` interface. A counter starts at zero and increases when clicked. In the terminal, **Ctrl + C** stops the server.

**Note:** Our Chapter 1 also includes a `prepare` script (`husky`). During installation, npm may initialize repository Git hooks. These hooks are project-specific additions, not requirements of React or Vite.

**Do not confuse the two paths:** `npm create vite@latest` makes a new project with up-to-date defaults; cloning and running `npm ci` reproduces our existing project and its locked dependencies.

---

## 5. Understand the Chapter 1 folder structure

This is a shortened map of **real paths** in the source repository:

```text
from-zero-to-fullstack-ch1-foundations/
├── .husky/
│   ├── pre-commit              # Runs npm run lint
│   └── commit-msg              # Runs commitlint
├── .gitignore                  # Ignores node_modules, dist, logs, etc.
├── .prettierrc.json            # Source formatting rules
├── commitlint.config.js        # Conventional commits
├── eslint.config.js            # Lint configuration
├── index.html                  # HTML shell; contains #root
├── package.json                # Scripts and package declarations
├── package-lock.json           # Locked dependency versions
├── public/
│   └── vite.svg                # Public static asset
├── src/
│   ├── App.jsx                 # Root React UI + counter
│   ├── App.css                 # Component styles
│   ├── index.css               # Global styles
│   ├── main.jsx                # React entry point
│   └── assets/react.svg        # Source-controlled asset
└── vite.config.js              # React plugin configuration
```

| Location | Purpose | When to change it |
| --- | --- | --- |
| `index.html` | Browser document, title, root element | Change page title and metadata |
| `src/main.jsx` | Mount React into the document | Add global providers and imports |
| `src/App.jsx` | Root component of the initial UI | Replace the starter with your app |
| `src/App.css` | Styles for root component | Adjust component appearance |
| `src/index.css` | Global CSS | Add a reset/theme/global rules |
| `public/` | Files served at absolute public paths | Favicons and unchanged static assets |
| `src/assets/` | Assets imported from source code | Component images and icons |
| `vite.config.js` | Vite build/dev tool configuration | Configure plugins or supported options |
| `package.json` | Scripts/dependencies | Install packages or define commands |
| `package-lock.json` | Exact dependency tree | Commit changes produced by npm |

`node_modules/` appears after dependency installation. `dist/` appears after a production build. Both are ignored in our Chapter 1's `.gitignore` and are **not source code to commit**.

## 6. Follow the browser-to-React execution path

### 6.1 `index.html` — the browser's entry document

**Actual Chapter 1 excerpt:**

```html
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.jsx"></script>
</body>
```

- `#root` is an initially empty container.
- The `type="module"` script loads `src/main.jsx` using modern JavaScript modules.
- React creates the interface inside the root container.

The original HTML title is `ch1` and its favicon is `/vite.svg`.

### 6.2 `src/main.jsx` — mount the application

**Actual Chapter 1 code:**

```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

- `createRoot(...)` connects React to the browser DOM element.
- `<App />` renders our root component.
- `<StrictMode>` enables extra **development-only checks** and may cause extra development renders. It is **not** a production performance problem by itself.
- Importing `index.css` activates the global stylesheet.

### 6.3 `src/App.jsx` — the first interactive component

**Selected source-backed behavior:** The real `App.jsx` imports `useState`, shows the Vite and React logos, renders `Vite + React`, and includes:

```jsx
const [count, setCount] = useState(0)

<button onClick={() => setCount((count) => count + 1)}>
  count is {count}
</button>
```

This introduces **state**: when the user clicks, `setCount` changes the state and React updates the rendered button label. The full file is available in the source repository.

### 6.4 `vite.config.js` — enable the React integration

**Actual Chapter 1 code:**

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

`@vitejs/plugin-react` integrates React into Vite. Do not confuse a Vite plugin with a React component.

---

## 7. Practical exercise — replace the starter with a clean component

**Guided improvement:** This section is an educational modification. The original GitHub source still contains the default logo-and-counter starter unless you personally modify your copy.

1. Open `src/App.jsx`.
2. Replace the file with:

```jsx
import { useState } from 'react'

export default function App() {
  const [count, setCount] = useState(0)

  function handleClick() {
    setCount((previous) => previous + 1)
  }

  return (
    <main>
      <h1>My First React App</h1>
      <p>React is running through Vite.</p>
      <button type="button" onClick={handleClick}>
        Clicked {count} times
      </button>
    </main>
  )
}
```

3. Save the file. Vite's **Hot Module Replacement (HMR)** usually shows the change in your browser without a full manual reload.
4. Click the button three times. **Expected:** `Clicked 3 times`.
5. Run `npm run lint`, then `npm run build` after stopping the development server.

**Explain the code:**

- `App` is a function component returning JSX.
- `useState(0)` stores a number for the current component.
- The `onClick` handler runs `handleClick` when the button is clicked.
- `setCount((previous) => previous + 1)` is the **functional update form**, useful when the next state depends on the previous one.
- `<main>` is a semantic HTML element; `type="button"` avoids accidental form submission in later projects.

**Common beginner mistake:** Do not use `count++` to update React state. Changing a normal variable does not notify React to update its UI.

### Restore original source if required

Before discarding work, review the diff:

```powershell
git diff -- src/App.jsx
```

If you are in the cloned source and **intentionally want to discard uncommitted changes** to this file:

```powershell
git restore src/App.jsx
```

**Warning:** `git restore` discards local uncommitted modifications to that file. Do not use it on work you need to keep.

---

## 8. Master the npm scripts and the full build cycle

These scripts come from our **actual** Chapter 1 `package.json`:

| Script | Defined command | Expected effect |
| --- | --- | --- |
| `npm run dev` | `vite` | Starts the development server |
| `npm run build` | `vite build` | Generates `dist/` |
| `npm run preview` | `vite preview` | Previews generated output |
| `npm run lint` | `eslint .` | Checks source for lint violations |
| `npm run lint:fix` | `eslint . --fix` | Applies supported automated lint fixes |
| `npm run prepare` | `husky` | Prepares the project's Git hooks |

```powershell
# From the project root
npm run lint
npm run build
npm run preview
```

**Verification:** Open the preview URL, confirm the UI loads, then stop it with **Ctrl + C**.

- `npm run dev` **does not** make a deployable build.
- `npm run build` compiles optimized assets to `dist/`.
- `npm run preview` is for local inspection of the build, not a production web server.
- When adding or upgrading packages, review resulting `package-lock.json` changes.

### Dependency types in `package.json`

**Actual Chapter 1 examples:**

- `dependencies`: `react`, `react-dom`, **and `mongodb`** (the latter is declared but is not used for browser-side database access in the starter).
- `devDependencies`: `vite`, `@vitejs/plugin-react`, ESLint, Prettier, Husky, Commitlint and related packages.

A `^` version range permits compatible updates according to npm semver rules. The lockfile records a resolved dependency tree. **Read both** before attempting exact reproduction.

### Check the environment

```powershell
node --version
npm --version
npm ls vite react react-dom
npm run lint
npm run build
```

**Expected:** Installed versions are listed; lint/build finish successfully; `dist/` exists. If npm lists missing packages, make sure you are in the folder containing `package.json` and install dependencies first.

---

## 9. Environment configuration and safety

**Important scope note:** The original Chapter 1 starter does **not** need a backend URL or project-specific `.env` file. The following is a recommended **new-project pattern** for later chapters.

### 9.1 A safe public frontend setting

Create a project-root file named `.env.example`:

```dotenv
# Non-secret frontend configuration only
VITE_APP_TITLE=My React App
```

For local testing, copy it into `.env.local` and update `src/App.jsx` as needed:

```jsx
const appTitle = import.meta.env.VITE_APP_TITLE ?? 'My React App'
```

- In Vite, variables with the `VITE_` prefix are exposed to client-side code.
- **Never put passwords, database credentials, JWT secrets or private API keys in `VITE_*` variables.** Browser visitors can inspect client bundle content.
- Restart the dev server after changing `.env` files.
- As a recommendation for **future projects**, add `.env`, `.env.*` and the relevant local variants to `.gitignore` while keeping `.env.example` tracked, according to your project policy. Do not claim this is how our Chapter 1 is configured: the source `.gitignore` currently excludes `*.local`, but it does not blanket-ignore all `.env` files.

### 9.2 Frontend versus backend variables

A Vite app runs in the browser after being built. It should call a backend API for protected operations. The database connection belongs on the **backend**; putting database credentials in a React client exposes them. Later chapters introduce the actual frontend/backend separation.

---

## 10. Debugging and common mistakes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `node` is not recognized | Node.js not installed or terminal PATH is stale | Install Node; reopen PowerShell; run `where.exe node` |
| `npm.ps1 cannot be loaded` | PowerShell execution policy | Use `npm.cmd`; avoid changing system-wide policy |
| `npm ERR! EBADENGINE` | Node version does not meet dependencies' requirements | Install compatible Node LTS; verify Vite's requirements |
| `npm: Missing script: dev` | Wrong working directory or missing script | Run `Get-Location`; inspect project `package.json` |
| Browser says connection refused | Dev server stopped or wrong URL | Run `npm run dev`; open printed address |
| Port 5173 unavailable | Another application uses it | Open the alternate URL printed by Vite or stop conflicting app |
| Cannot find `react` or `vite` | Dependencies not installed | Run `npm ci` for cloned project; `npm install` for new scaffold |
| `npm ci` says lockfile is not synchronized | `package.json` and lockfile disagree | Inspect Git changes; restore a known good pair or update intentionally |
| React screen is blank | Syntax/runtime error in component | Inspect terminal and browser developer console |
| Counter does not update | State mutated directly or handler not invoked | Use `useState` and its setter, wire `onClick` correctly |
| Build fails on import | Incorrect case, path or missing file | Check filename capitalization and paths |
| Modified `.env` value not shown | Server not restarted or missing `VITE_` prefix | Restart Vite; verify public variable naming |
| Unexpected duplicate logs or effects | React Strict Mode development checks | Consult React StrictMode docs before removing protections |
| Pushing `dist` / `node_modules` | Ignore rules missing or Git already tracks them | Inspect `.gitignore` and `git status` before committing |

**Useful diagnostic commands:**

```powershell
Get-Location
Get-ChildItem
where.exe node
where.exe npm
node --version
npm.cmd --version
npm ls vite react react-dom
git status
```

**Security reminder:** Remove secrets and private environment settings before sharing console output or screenshots.

---

## 11. Good practices for future projects

1. **Choose between clone and scaffold deliberately:** `npm ci` for an existing locked repository; `npm install` for a freshly generated project.
2. **Use a supported Node version:** check the `engines` field, dependency warnings and the official Vite documentation.
3. **Maintain predictable files:** name components clearly and keep entry files simple.
4. **Keep business secrets server-side:** never use `VITE_*` for passwords or tokens you need to protect.
5. **Review changes:** inspect `git status`, relevant diffs and dependency lockfile updates.
6. **Lint and build before publishing:** `npm run lint` and `npm run build` should pass in a healthy project.
7. **Do not commit generated folders:** ignore `node_modules` and `dist` unless your particular publishing process explicitly requires something different.
8. **Prefer the functional state setter** when updating based on earlier state.
9. **Avoid unrelated dependencies:** only add packages you actually use; `mongodb` in the Chapter 1 frontend package is not required for the basic React/Vite starter.
10. **Use environment samples:** commit a safe `.env.example` for future API configuration; not real secrets.

### Reusable project setup workflow

```text
Choose compatible Node LTS
         ↓
Open terminal in a clean workspace
         ↓
Scaffold Vite React (or clone an existing locked repository)
         ↓
Install dependencies
         ↓
Run dev server → verify browser counter
         ↓
Read index.html → main.jsx → App.jsx → vite.config.js
         ↓
Run lint and build
         ↓
Review git status and commit safe source files
```

## 12. Copy-and-use commands

**New React JavaScript application:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
node --version
npm.cmd --version
npm.cmd create vite@latest my-react-app -- --template react
Set-Location my-react-app
npm.cmd install
npm.cmd run dev
# Stop with Ctrl+C when finished
npm.cmd run lint
npm.cmd run build
npm.cmd run preview
```

**Reproduce our Chapter 1 project:**

```powershell
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
npm.cmd ci
npm.cmd run dev
# Stop with Ctrl+C when finished
npm.cmd run lint
npm.cmd run build
git status
```

**Do not clone twice into the same directory.** If you already have the project, navigate into it and inspect `git status` before pulling changes.

## 13. Beginner verification checklist

- [ ] I understand what React, JSX, Vite, Node.js and npm each do.
- [ ] I installed and verified VS Code, Git, Node.js and npm.
- [ ] My Node version meets the selected project's requirements.
- [ ] I successfully created a **new** React/Vite project.
- [ ] I successfully cloned and installed the **original Chapter 1** repository.
- [ ] I can explain `index.html` → `src/main.jsx` → `src/App.jsx`.
- [ ] I can run the development server and open the exact displayed URL.
- [ ] I clicked the counter and saw React update the UI.
- [ ] I understand the difference between `npm install` and `npm ci`.
- [ ] I ran `npm run lint` and `npm run build` successfully.
- [ ] I know what `package.json`, `package-lock.json`, `node_modules` and `dist` mean.
- [ ] I understand why `VITE_*` variables must not contain secrets.
- [ ] I can find useful errors in the terminal and browser console.

**Outcome:** You can bootstrap a modern React frontend and confidently reproduce the project that our Chapter 1 used.

---

## 14. Sources and verification boundary

### Source-verified project files (reviewed 13 October 2026)

- [Chapter 1 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)
- [`package.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/package.json)
- [`index.html`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/index.html)
- [`src/main.jsx`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/src/main.jsx)
- [`src/App.jsx`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/src/App.jsx)
- [`vite.config.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/vite.config.js)
- [`.gitignore`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.gitignore)

### Official reference documentation

- [Node.js — official downloads](https://nodejs.org/en/download)
- [VS Code — Windows installation](https://code.visualstudio.com/docs/setup/windows)
- [Git for Windows](https://git-scm.com/install/windows)
- [Vite — Getting Started and Node compatibility](https://vite.dev/guide/)
- [Vite — Environment variables and modes](https://vite.dev/guide/env-and-mode)
- [React — Strict Mode](https://react.dev/reference/react/StrictMode)
- [React — State: `useState`](https://react.dev/reference/react/useState)

**Verification boundary:** GitHub project code, file locations and declared versions were reviewed directly. The *create-new-app* tutorial is a documented, officially supported workflow; generated scaffolding and prompts may change with future Vite versions. No claim is made that the complete installation was physically repeated on all Windows configurations.

**Next:** Cheatsheet 04 — ESLint & Prettier Configuration.  
[← Chapter 1 index](../README.md) · [Full documentation library](../../README.md)
