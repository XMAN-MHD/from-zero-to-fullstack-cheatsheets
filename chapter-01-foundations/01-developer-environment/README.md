# Cheatsheet 01 — Complete Developer Environment Setup

**From Zero to Fullstack · Chapter 1: Foundations**  
**Audience:** Absolute beginners and developers refreshing their setup  
**Platform:** Windows 10/11; PowerShell and Visual Studio Code  
**Documentation version:** 1.0 · 13 October 2026  
**Original project:** https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations

> **Learn → Practice → Document → Reuse.**  
> Set up a JavaScript development workstation from scratch, reproduce the real Chapter 1 React/Vite project, verify each tool, and troubleshoot common Windows problems.

## At a glance

| Tool | Purpose | Verification |
| --- | --- | --- |
| VS Code | Write and inspect code | `code --version` |
| Git for Windows | Track changes | `git --version` |
| GitHub | Host your repositories | Sign in using a browser |
| Node.js LTS | Run JavaScript outside browsers | `node --version` |
| npm | Install and run dependencies | `npm --version` |
| Browser | Check your running React application | Open Vite's local URL |

**Scope:** This first guide is about the complete development environment and reproducing the existing starter. Future guides explain Git, Vite/React, ESLint/Prettier, and Husky/Commitlint in greater depth.

## 1. Essential ideas

A **frontend** is the browser interface. A **backend** receives requests and applies application logic. A **database** stores persistent information. An **API** enables systems to exchange information, commonly through HTTP and JSON.

The first chapter prepares our workstation and a minimal Vite + React starter. It does **not** yet contain a complete full-stack MERN application.

- **Terminal:** an interface where you type commands (PowerShell, Git Bash).
- **Git:** local source code version control.
- **GitHub:** online hosting of repositories; it is not the same as Git.
- **Node.js:** a JavaScript runtime outside the browser.
- **npm:** a package manager and script runner distributed with Node.js.

The workflow is simple: Windows → VS Code + terminal → Git + Node/npm → React/Vite development server → browser. GitHub stores the project's remote source history.

## 2. Requirements and compatibility

**Required knowledge:** None. You should only know how to download an installer, open a folder, and enter a command.

**Recommended PC:** Windows 10 or 11, 8 GB RAM or better, a few GB of free disk space, and internet access. Lower-spec machines may work more slowly.

**Not required in Chapter 1:** Docker, WSL, MongoDB installation, or a cloud deployment account.

The published Chapter 1 source uses **React 19 and Vite 7**. Vite 7 needs **Node.js 20.19+ or 22.12+**. A supported **Node.js 22 LTS (22.12 or newer)** is a course-aligned option; a compatible newer LTS can also work. Check the versions installed on your computer instead of assuming every installer meets the project's minimum.

Only download tools from their official websites. Never commit API keys, passwords, or private environment variables.

## 3. Install Visual Studio Code

1. Open https://code.visualstudio.com/download.
2. Download the **Windows User Installer** (recommended for most users).
3. Run the installer and follow its steps.
4. Open Visual Studio Code from the Start menu.
5. Close and reopen PowerShell after installing so your updated `PATH` is recognized.
6. Run:

~~~powershell
code --version
~~~

**Expected:** VS Code prints its version and build information. If `code` is not recognized, restart the terminal and consult the official Windows setup instructions to add the VS Code command to `PATH`.

Useful shortcuts:

| Action | Shortcut |
| --- | --- |
| Open project folder | File → Open Folder; or `code .` |
| Integrated terminal | Ctrl + backtick |
| Command Palette | Ctrl + Shift + P |
| Open file quickly | Ctrl + P |
| Search all files | Ctrl + Shift + F |
| Extensions view | Ctrl + Shift + X |
| Save | Ctrl + S |

Useful extensions: **ESLint**, **Prettier – Code formatter**. **GitLens** is optional. Extensions help the editor; they are different from installing dependencies inside an npm project.

## 4. Install and configure Git

1. Visit https://git-scm.com/install/windows.
2. Download and run the official **Git for Windows** installer.
3. Prefer normal installer defaults unless you have special requirements. Git Bash and Git Credential Manager are helpful.
4. Restart the terminal.
5. Confirm the installation:

~~~powershell
git --version
~~~

**Expected:** A Git version string.

Configure your identity with your own details (a GitHub **noreply** address can help protect email privacy):

~~~powershell
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global --list
~~~

Git records the configured name and email on your commits. `init.defaultBranch main` makes `main` the initial branch in newly initialized repositories.

**Terminal rule:** This guide uses Windows PowerShell, not Bash. Similar-looking commands may behave differently between shells.

## 5. Set up a GitHub account securely

1. Sign up at https://github.com/signup.
2. Verify your email.
3. Enable **two-factor authentication (2FA)**.
4. Sign in and verify you can see your dashboard.

**Cloning** a public project typically does not require authentication. **Pushing** to a repository requires write permission and authentication. Prefer **Git Credential Manager**'s secure browser login. GitHub no longer accepts normal account passwords for HTTPS Git operations; use a credential manager, SSH, or an appropriately scoped token where required.

**Never paste tokens** in a code sample, terminal screenshot, or Git remote URL that you publish.

## 6. Install Node.js and npm

1. Open https://nodejs.org/en/download.
2. Select a compatible **LTS** Node.js release. For course alignment, choose **22 LTS (22.12+)** from the official version selector/archive if available. A supported newer LTS can also work.
3. Download the **Windows .msi installer** for your architecture (usually x64).
4. Complete installation with normal defaults; npm is typically included.
5. Restart PowerShell and verify:

~~~powershell
node --version
npm --version
node -e "console.log('Node works:', 2 + 2)"
~~~

**Expected:** Node and npm versions followed by `Node works: 4`.

### Windows PowerShell may block npm.ps1

If you see *“npm.ps1 cannot be loaded because running scripts is disabled”*, do not weaken machine-wide policies for this tutorial. Run the equivalent `.cmd` launcher:

~~~powershell
npm.cmd --version
npm.cmd ci
npm.cmd run dev
~~~

Command Prompt and Git Bash are other alternatives.

| Command | What it does |
| --- | --- |
| `npm install` | Install packages; may update lockfile |
| `npm ci` | Clean install using an existing `package-lock.json` |
| `npm run dev` | Run the `dev` script |
| `npm run lint` | Run the project's linter |
| `npm run build` | Compile the production build |
| `npm run preview` | Preview the built Vite site |

Do not normally commit `node_modules/`; do commit `package-lock.json` so dependencies are reproducible.

## 7. Create a clean local workspace

Open PowerShell and run:

~~~powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
Get-Location
~~~

**Expected:** Your current location ends in `Projects`. A local development directory can avoid synchronization conflicts caused by some cloud-backed folders.

| PowerShell command | Meaning |
| --- | --- |
| `pwd` | Show current location |
| `ls` | List current files |
| `cd directory` | Enter directory |
| `cd ..` | Go to parent |
| `code .` | Open current folder in VS Code |
| Ctrl + C | Stop a running terminal process |

Always use `pwd` before installing dependencies or running project scripts.

## 8. Reproduce our actual Chapter 1 project

This is the source project we actually worked with: https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.

### 8.1 Clone

~~~powershell
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
code .
~~~

**Expected:** A VS Code window displaying `package.json`, `src/`, `index.html`, and `vite.config.js`. Cloning lets you inspect and run the public project. It does not grant permission to push to its original repository.

### 8.2 Install exact locked dependencies

~~~powershell
npm ci
~~~

Use `npm.cmd ci` if PowerShell blocks `npm.ps1`.

**Expected:** Dependencies installed using the existing `package-lock.json`. Because the repository defines a `prepare` script, Husky may initialize Git hooks after install.

### 8.3 Run the frontend

~~~powershell
npm run dev
~~~

**Expected:** The terminal shows the Vite server URL, often `http://localhost:5173/`. Open the **exact address displayed**. If port 5173 is occupied, Vite may use another.

You should see the **Vite + React** starter. Click the counter button to check that React state updates. Stop the server with **Ctrl + C**.

### 8.4 Verify lint, build, and preview

~~~powershell
npm run lint
npm run build
npm run preview
~~~

Execute in this order. Inspect the URL printed by the preview command and stop it with **Ctrl + C** when finished.

**Expected:** Linting finds no blocking issue; building creates the `dist/` folder; preview serves the compiled application. The project's `.gitignore` excludes `dist/`.

### 8.5 Source-grounded project layout

~~~text
from-zero-to-fullstack-ch1-foundations/
├── .husky/
│   ├── pre-commit           # npm run lint
│   └── commit-msg           # commitlint
├── .prettierrc.json         # Formatting
├── .gitignore               # Ignore node_modules and dist
├── commitlint.config.js     # Conventional commits
├── eslint.config.js         # JavaScript/React linting
├── index.html               # Browser HTML entry
├── package.json             # Project scripts/dependencies
├── package-lock.json        # Locked dependency tree
├── src/
│   ├── App.jsx             # React starter and counter
│   └── main.jsx            # React entry point
└── vite.config.js           # React plugin for Vite
~~~

At the time of review, the repository declares **React `^19.1.1`, Vite `^7.1.7`, Husky `^9.1.7`** and defines:

~~~json
{
  "scripts": {
    "prepare": "husky",
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "preview": "vite preview"
  }
}
~~~

The `.husky/pre-commit` hook runs `npm run lint`; `.husky/commit-msg` runs `npx commitlint --edit "$1"`. These are explored in dedicated later guides.

**Accuracy note:** `mongodb` is listed as a Chapter 1 dependency, but the React starter is not a functioning MongoDB application. Dependency presence by itself is not proof of database use.

## 9. Optional: start your own new React application

**Additional exercise—not a reproduction of the original Chapter 1 history.**

~~~powershell
Set-Location "$HOME\Projects"
npm create vite@latest my-first-react-app -- --template react
Set-Location my-first-react-app
npm install
npm run dev
~~~

Open the URL displayed by Vite. This creates a **new template** and may use more recent dependencies than the original chapter. For an exact existing-project reproduction, clone it and run `npm ci`.

## 10. First Git inspection exercise

~~~powershell
Set-Location "$HOME\Projects\from-zero-to-fullstack-ch1-foundations"
git status
git log -1 --oneline
git remote -v
~~~

- **git status:** changes in your working tree.
- **git log:** latest recorded commit.
- **git remote -v:** configured remote origin.

Do not attempt to push to someone else's project without permission. To share your own work, create a repository under your GitHub account and connect that new repository as the remote. The next cheatsheet will explain init/clone/add/commit/branch/push/pull in detail.

## 11. Troubleshooting on Windows

- **`node` not recognized:** Install Node.js, restart the terminal, and run `where.exe node`.
- **`npm.ps1 ... disabled`:** Use the Windows `npm.cmd` launcher or Git Bash; do not weaken system security settings.
- **`git` not recognized:** Reinstall Git for Windows, restart, and run `where.exe git`.
- **`code` not recognized:** Restart the terminal or check the official VS Code `PATH` instructions.
- **`EBADENGINE`:** Your Node version is incompatible; verify Vite's version requirements.
- **`npm ci` lockfile error:** Inspect first error; restore committed `package.json`/lockfile if you modified the original.
- **Missing npm script:** Run `pwd`; check `package.json` from the correct folder.
- **Port occupied:** Stop the other server or use the alternate URL Vite displays.
- **Browser cannot reach localhost:** Keep the server active and open its exact local URL.
- **`git push` permission denied:** You may not own that repository or may need to authenticate.
- **GitHub HTTPS password prompt:** Prefer browser login with Git Credential Manager or an authorized alternative.
- **`node_modules` appears in Git status:** Inspect `.gitignore`, and never commit dependencies unnecessarily.

Useful diagnostics:

~~~powershell
pwd
where.exe node
where.exe npm
where.exe git
node --version
npm.cmd --version
git --version
code --version
~~~

Share error messages when seeking help, but **remove credentials, private links, tokens, and personal data** first.

## 12. Security and good habits

1. Keep real credentials out of repositories and screenshots.
2. For future apps, ignore `.env` and `.env.*` while allowing a sanitized `.env.example`.
3. Run `git status` and review file changes before committing.
4. Use supported Node LTS versions and check project requirements before upgrading.
5. Use `npm ci` for existing locked projects and `npm install` for a newly scaffolded project.
6. Do not run unknown installation scripts as Administrator.
7. Distinguish machine-installed apps, editor extensions, and npm packages.
8. Version the source, documentation, and lockfiles; usually ignore `node_modules` and `dist`.

**Original-project limitation:** The Chapter 1 `.gitignore` ignores `node_modules` and `dist`, but does not have a blanket `.env` rule. Adding an `.env` ignore pattern is a **recommendation for new work**, not a claim about the published Chapter 1 source.

## 13. Quick reference

~~~powershell
# Check prerequisites
node --version
npm.cmd --version
git --version
code --version

# Reproduce the existing project
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
npm.cmd ci
npm.cmd run dev

# Stop the dev server with Ctrl+C; then:
npm.cmd run lint
npm.cmd run build
git status
git remote -v
~~~

If you already cloned the repository, **enter the existing folder** instead of cloning again over it. Review `git status` before pulling updates.

## 14. Final verification checklist

- [ ] I have a clean local project folder.
- [ ] VS Code opens and `code --version` works.
- [ ] Git works and my commit author identity is configured.
- [ ] GitHub sign-in works and my account uses 2FA.
- [ ] Node.js satisfies Vite's requirements.
- [ ] npm works normally or through `npm.cmd`.
- [ ] I cloned the actual Chapter 1 source project.
- [ ] `npm ci` completed without errors.
- [ ] The browser displays the starter and its counter changes.
- [ ] `npm run lint` and `npm run build` work.
- [ ] I understand cloning versus creating a new Vite project.
- [ ] I can locate project configuration and troubleshooting logs.

**Outcome:** Your computer is ready for **Chapter 2 — Backend Fundamentals**.

## 15. Project evidence and official references

**Source-verified Chapter 1 files:**

- [Chapter 1 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)
- [package.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/package.json)
- [vite.config.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/vite.config.js)
- [eslint.config.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/eslint.config.js)
- [Husky hooks](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/tree/main/.husky)

**Official setup documentation:**

- [VS Code — Windows setup](https://code.visualstudio.com/docs/setup/windows)
- [Git for Windows](https://git-scm.com/install/windows)
- [Git — first-time configuration](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [Node.js downloads](https://nodejs.org/en/download)
- [Vite setup and compatibility](https://vite.dev/guide/)
- [Create a GitHub account](https://docs.github.com/en/account-and-profile/how-tos/account-management/creating-an-account-on-github)
- [GitHub authentication options](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)

**Verification boundary:** Source file names, scripts, and declared dependencies were reviewed against the published Chapter 1 repository on 13 October 2026. Installation guidance is based on official tool documentation. The full new-PC Windows walkthrough is provided as an actionable procedure, not claimed to have been physically executed on every supported machine.

---

**Next (planned):** Cheatsheet 02 — Git & GitHub from Scratch.  
[Back to Chapter 1](../) · [Full library roadmap](../../README.md)
