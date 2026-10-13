# Cheatsheet 05 — Git Hooks with Husky & Commitlint

**From Zero to Fullstack · Chapter 1: Foundations**  
**Audience:** Absolute beginners and developers building reliable Git workflows  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Version:** 1.0 · 13 October 2026  
**Project source:** [Chapter 1 — Foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Understand native Git hooks, reproduce the precise Husky and Commitlint setup from our Chapter 1 project, install the same workflow in a new JavaScript project, and prove that both code-quality and commit-message checks work.

## Quick reference — what runs when you commit?

```text
Developer types: git commit -m "feat: add welcome message"
                     |
                     v
               Git pre-commit
               .husky/pre-commit
                     |
                     +-- npm run lint
                     |     |
                     |     +-- FAIL -> commit stops
                     |     +-- PASS -> continue
                     v
                Git commit-msg
                .husky/commit-msg
                     |
                     +-- npx commitlint --edit "$1"
                     |     |
                     |     +-- FAIL -> commit stops
                     |     +-- PASS -> commit saved locally
                     v
               git push (separate step)
```

| Component | Role and Chapter 1 implementation |
| --- | --- |
| **Git** | Runs native hooks before a commit is saved |
| **Husky 9** | Installs versioned hooks through `prepare: husky` |
| **`pre-commit`** | Runs `npm run lint` before committing |
| **`commit-msg`** | Runs Commitlint against the proposed message |
| **Commitlint** | Enforces conventional commit-message rules |
| **ESLint** | Runs `eslint .` to check JavaScript/React code |

**Important:** Hooks are **local checks**, not automatic GitHub Actions checks. Someone may bypass local hooks with `--no-verify`, and a GitHub web edit does not execute your local hooks. For team-wide enforcement, use server-side CI and branch protection separately.

---

## 1. First understand Git hooks, Husky, and Commitlint

A **Git hook** is a script Git can execute at a predefined point in its workflow. Some hooks are invoked before committing, some after, and others during other Git operations. A hook that exits with a **nonzero exit code** can prevent a commit when the hook supports that behavior.

**Husky** helps manage those scripts inside the repository, so teammates can share the hook definitions. Git itself still invokes the hooks; Husky makes their configuration practical in modern JavaScript projects. In Husky 9, installation uses Git's `core.hooksPath` and a `prepare` script.

**Commitlint** reads a proposed commit message, applies a rule set, and reports violations. It is a different tool from ESLint: ESLint checks **source code**; Commitlint checks **commit messages**.

### Native hooks we use

- **`pre-commit`**: runs before a commit is created; typically runs linters or checks.
- **`commit-msg`**: receives the path to Git's temporary commit-message file as its first argument. Commitlint checks the message stored in that file.
- **`post-commit`**: runs after a commit, but is **not** used in our Chapter 1 repository.

The literal **`"$1"` belongs to the hook's POSIX shell**, not to PowerShell: it means “the first argument Git passes to this hook.” Never substitute a personal filename or a GitHub URL for it.

## 2. Prerequisites — begin from a fresh Windows PC

You need internet access and permission to install software. Basic terminal familiarity helps, but no previous hook experience is required.

### 2.1 Install VS Code

1. Download the official Windows installer: <https://code.visualstudio.com/download>.
2. Install it and open **Terminal → New Terminal**.
3. Restart PowerShell and verify:

```powershell
code --version
```

**Expected:** VS Code reports its version. If `code` is not recognized, use VS Code's **File → Open Folder** menu and check that its command is in `PATH`.

### 2.2 Install Git for Windows

1. Download: <https://git-scm.com/install/windows>.
2. Use the normal installation options, including Git Bash where offered.
3. Close/reopen PowerShell and verify:

```powershell
git --version
where.exe git
```

**Expected:** A Git version and an installed `git.exe` location. Git is mandatory: Husky hooks do not make sense outside a Git repository.

### 2.3 Install Node.js and npm

1. Download a compatible **Node.js LTS** Windows installer: <https://nodejs.org/en/download>.
2. Install it, restart the terminal, and verify:

```powershell
node --version
npm --version
npx --version
```

The Chapter 1 React/Vite source uses Vite 7, whose compatible Node.js minimums include **20.19+ or 22.12+**. Use a supported LTS that meets the requirements of the project you are building.

**PowerShell execution-policy warning:** If `npm.ps1` or `npx.ps1` is blocked, use the existing Windows command launchers instead:

```powershell
npm.cmd --version
npx.cmd --version
```

Throughout this guide, `npm` can be replaced with `npm.cmd`, and `npx` with `npx.cmd`. There is no need to disable PowerShell security protections globally.

### 2.4 Configure your local Git identity

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global --list
```

Use your own identity or a privacy-preserving GitHub noreply email. Configuring an author identity does **not** authenticate you to GitHub.

---

## 3. Path A — inspect and reproduce OUR real Chapter 1 project

This section is **source-grounded**: the paths and configurations below were checked against the published GitHub repository on 13 October 2026.

### Step 1 — Clone the source

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
code .
```

If the repository already exists locally, enter it instead of cloning again.

### Step 2 — Install the locked dependencies

```powershell
npm ci
```

**Expected:** Dependencies are installed from `package-lock.json`. The project's `prepare` script (`husky`) normally runs during installation and sets up the Git hooks. A script-disabled npm installation or disabled Husky environment may prevent this setup; see Troubleshooting.

### Step 3 — Examine the actual file structure

```text
from-zero-to-fullstack-ch1-foundations/
├── .git/                  # Git's internal state (created by clone)
├── .husky/
│   ├── pre-commit         # npm run lint
│   └── commit-msg         # npx commitlint --edit "$1"
├── commitlint.config.js   # Conventional Commits preset
├── eslint.config.js       # Source lint rules
├── package.json           # prepare, lint and dependencies
└── package-lock.json      # Locked dependency versions
```

> The `.git/` directory is local Git metadata; it is not tracked as a folder inside the GitHub repository. The `.husky/` hook files, in contrast, are versioned project files.

### Step 4 — Read the actual configurations

**`package.json` — actual relevant scripts:**

```json
{
  "type": "module",
  "scripts": {
    "prepare": "husky",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

**Actual Chapter 1 development dependencies include:** `husky: ^9.1.7`, `@commitlint/cli: ^20.1.0`, and `@commitlint/config-conventional: ^20.0.0` (declared ranges, **not** a statement about exact currently resolved versions).

**`.husky/pre-commit` — actual contents:**

```sh
npm run lint
```

**`.husky/commit-msg` — actual contents:**

```sh
npx commitlint --edit "$1"
```

**`commitlint.config.js` — actual contents:**

```js
export default {
  extends: ['@commitlint/config-conventional'],
}
```

The project uses **ES modules** (`"type": "module"`), which is why this configuration uses `export default` rather than CommonJS `module.exports`.

### Step 5 — Verify installation and checks

```powershell
npm run prepare
git config --get core.hooksPath
npm run lint
npx commitlint --version
```

**Expected:** `core.hooksPath` usually prints `.husky/_` for Husky 9, linting succeeds if the checked-in source has no lint errors, and Commitlint reports a version. If `core.hooksPath` is empty, inspect the Husky initialization and the Git root.

**Do not modify or test commits directly on the course's `main` branch.** Use the disposable exercise in Section 6 for real pass/fail commit tests.

---

## 4. Path B — add both tools to YOUR new project

This is a **reusable recommended setup**, not a claim that these installation steps were recorded in the history of the Chapter 1 source.

### Step 1 — Create a safe practice project

```powershell
Set-Location "$HOME\Projects"
New-Item -ItemType Directory -Force -Path "git-hooks-lab"
Set-Location "git-hooks-lab"
git init -b main
npm init -y
```

**Expected:** A new local Git repository and `package.json` in an empty practice folder. If `git init -b main` is unsupported by an old Git installation, update Git.

### Step 2 — Add the tool dependencies

```powershell
npm install -D husky @commitlint/cli @commitlint/config-conventional eslint
```

> `eslint` is included so our pre-commit hook can run `npm run lint`. A real React/Vite project also needs an appropriate `eslint.config.js`. We create a minimal JavaScript example in the next step.

### Step 3 — Make a minimal ESLint configuration

Create **`eslint.config.js`**:

```js
export default [
  {
    files: ['**/*.js'],
    ignores: ['node_modules/**'],
    rules: {
      'no-unused-vars': 'error',
    },
  },
]
```

This modern **flat configuration** is intentionally minimal for the lab. Your React app should use the fuller Chapter 1 ESLint configuration and its React plugins instead.

Make sure the project uses ES modules. Run:

```powershell
npm pkg set "type=module"
npm pkg set "scripts.lint=eslint ."
```

Create **`app.js`**:

```js
export function greet(name) {
  return `Hello, ${name}!`
}
```

The `greet` function is exported; no unused-variable rule violation should occur in this first example.

### Step 4 — Initialize Husky 9

```powershell
npx husky init
```

**Expected:** Husky creates `.husky/pre-commit` and configures the `prepare` script automatically. Inspect the generated file instead of assuming it already runs ESLint.

Change **`.husky/pre-commit`** to precisely:

```sh
npm run lint
```

On Windows, edit the file in VS Code and ensure it uses **LF** line endings if a shell execution error appears.

### Step 5 — Configure Commitlint

Create **`commitlint.config.js`**:

```js
export default {
  extends: ['@commitlint/config-conventional'],
}
```

Create **`.husky/commit-msg`** with this exact content:

```sh
npx commitlint --edit "$1"
```

You may edit the file with VS Code, or create it from PowerShell while keeping `$1` **literal**:

```powershell
Set-Content -Path .husky/commit-msg -Value 'npx commitlint --edit "$1"'
```

The **single-quoted PowerShell string** prevents PowerShell from expanding `$1` before Git's shell receives it. Verify the file by opening it in your editor.

### Step 6 — Activate hooks and verify

```powershell
npm run prepare
git config --get core.hooksPath
npm run lint
```

**Expected:** Husky sets `core.hooksPath`; the simple JavaScript example passes ESLint.

**Check before making any commit:** Run `git status`, confirm that you are inside `git-hooks-lab`, and ensure `.gitignore` excludes `node_modules/` and `dist/`. For example, create `.gitignore` containing:

```gitignore
node_modules/
dist/
.env
.env.*
!.env.example
```

Do not commit real credentials.

---

## 5. Conventional Commits and Commitlint

The **Conventional Commits** format makes the intent of a change readable to humans and tooling:

```text
type(optional-scope): short imperative description
```

| Type | Typical meaning | Example |
| --- | --- | --- |
| `feat` | Introduce a feature | `feat(auth): add login endpoint` |
| `fix` | Correct a bug | `fix(api): handle missing post` |
| `docs` | Change documentation | `docs: explain environment setup` |
| `refactor` | Reorganize code without feature change | `refactor: extract post service` |
| `test` | Add or adjust automated tests | `test: cover duplicate username` |
| `chore` | Maintenance work | `chore: update dependencies` |
| `ci` | Change CI automation | `ci: add backend checks` |
| `build` | Change build tooling | `build: configure vite` |
| `style` | Formatting without behavior change | `style: format source files` |
| `perf` | Performance improvement | `perf: avoid repeated lookups` |

A **scope** is an optional qualifier such as `frontend`, `backend`, or `auth`. Avoid writing sentences such as `update stuff` or `final version` as commit subjects.

**Breaking changes:** The standard supports forms such as `feat(api)!: change response format` or a `BREAKING CHANGE:` footer. This is not required for routine beginner commits, but it becomes important when your API consumers rely on a stable contract.

### Test Commitlint without creating any commit

Use Git Bash or PowerShell. In PowerShell:

```powershell
"feat: add greeting" | npx commitlint
"update stuff" | npx commitlint
```

**Expected:** The first message passes (exit code 0); the second normally fails with Commitlint errors. To inspect the command result in PowerShell:

```powershell
$LASTEXITCODE
```

A zero value indicates success; a nonzero value indicates failure. Test each message independently and check the exit code right after it.

**One subtle point:** Running Commitlint through a pipe tests the rules themselves. It does **not** prove that Git invokes your `commit-msg` hook. Section 6 tests the full commit workflow.

---

## 6. Hands-on lab — demonstrate that each hook really works

Do these exercises in **your disposable `git-hooks-lab` project**, never directly on your course's `main` branch.

### Test A — a valid commit succeeds

```powershell
Set-Location "$HOME\Projects\git-hooks-lab"
git status
git add .gitignore package.json package-lock.json eslint.config.js app.js commitlint.config.js .husky/
git commit -m "chore: initialize git hooks lab"
```

**Expected:** The `pre-commit` hook runs ESLint and the `commit-msg` hook checks the message; both pass and Git creates a local commit. If your Git configuration requires additional identity information, set `user.name` and `user.email` from Section 2.

### Test B — an invalid commit message is rejected

```powershell
Add-Content -Path README.md -Value "Hooks prevent bad commits."
git add README.md
git commit -m "updated stuff"
```

**Expected:** ESLint passes, but Commitlint rejects the message; **no new commit is created**. The staged changes remain available.

Then use a valid message:

```powershell
git commit -m "docs: describe git hooks"
```

**Expected:** Commit succeeds and the history now includes the documentation change.

**PowerShell note:** If `README.md` does not yet exist, `Add-Content` creates it.

### Test C — ESLint stops a commit

Intentionally introduce a violation by adding this line to `app.js`:

```js
const unusedValue = 123
```

Then:

```powershell
git add app.js
git commit -m "test: verify lint hook"
```

**Expected:** The `pre-commit` hook runs `npm run lint`, detects the unused variable, and aborts the commit **before** Commitlint checks the message.

Remove the intentionally unused line, then run:

```powershell
npm run lint
git add app.js
git status
```

You may stop there if no files changed after cleanup. Do **not** generate empty test commits unless that is your deliberate exercise.

### Confirm the saved history

```powershell
git log --oneline --max-count=5
git status
```

**Expected:** Only the valid commits exist. None of these steps push anything to GitHub: `git push` is always a separate operation.

---

## 7. How hook files are discovered and executed

Native Git can run hooks from its default hooks directory. **Husky 9 changes `core.hooksPath`** so Git finds Husky's managed hooks in `.husky/_`.

```powershell
git config --get core.hooksPath
```

**Typical result:** `.husky/_`. The versioned files you should edit live in `.husky/pre-commit` and `.husky/commit-msg`; do **not** edit Husky's generated `.husky/_` support files.

The sequence for `git commit -m "..."` is:

1. Git prepares the staged snapshot.
2. Git invokes **`pre-commit`**; a nonzero exit aborts.
3. Git prepares the message and invokes **`commit-msg`**; a nonzero exit aborts.
4. Only then does Git write the commit to its **local history**.

**Performance trade-off:** Our Chapter 1 pre-commit runs `eslint .`, linting the entire project. For a larger application, a focused staged-file check may be faster; introducing **lint-staged** would be a **future improvement**, not something already present in Chapter 1.

## 8. Windows, PowerShell, and cross-platform behavior

Git for Windows provides the shell tools normally used to execute Husky hooks. **The hook files contain shell commands**, even when you start `git commit` from PowerShell. It is okay for your PowerShell installation commands and the hook's shell syntax to look different.

- Use **PowerShell** for the terminal steps in this cheatsheet.
- Keep `"$1"` literal inside the `commit-msg` hook.
- Prefer **LF** line endings for portable hook scripts.
- Restart your terminal after installing Git or Node.js so `PATH` updates are applied.
- If a GUI Git client says `node: command not found`, check its environment and Node installation; compare with running the same commit from a terminal.
- For Linux/macOS, use the matching package managers and terminal syntax; on those systems you may need executable file permissions for hook scripts.

## 9. Troubleshooting — symptom, cause, fix

| Symptom | What to check or fix |
| --- | --- |
| `git` is not recognized | Install Git for Windows, reopen PowerShell, run `git --version` |
| `npm.ps1` is blocked | Use `npm.cmd` and `npx.cmd` in PowerShell |
| `husky: command not found` | Install dependencies (including dev dependencies) |
| Empty `core.hooksPath` | Run `npm run prepare` inside the Git repository |
| Hooks never run | Inspect Git root, hook path, and whether Husky is disabled |
| `Missing script: lint` | Add or correct the `lint` script in `package.json` |
| `pre-commit` fails | Run `npm run lint`, fix errors, and retry |
| `commit-msg` fails | Use a conventional message such as `docs: explain hooks` |
| Commitlint preset not found | Install `@commitlint/cli` and the conventional preset |
| `$1` is empty in the hook | Keep literal `"$1"` using single-quoted PowerShell text |
| `env: sh\r: ...` | Save shell hook files with LF line endings |
| GUI commit fails, terminal works | Investigate Node/`PATH` differences in the GUI environment |
| CI/production install fails | Review Husky's official CI/production guidance |

Useful diagnostics:

```powershell
pwd
git status
git rev-parse --show-toplevel
git config --show-origin --get core.hooksPath
npm run prepare
npm run lint
npx commitlint --version
```

When sharing logs, remove confidential data and tokens.

## 10. Security, limits, and good practices

1. **Do not trust local hooks as the only quality gate.** They run on each contributor's machine and can be bypassed. Use GitHub Actions/branch protection for team-wide checks.
2. **Avoid auto-changing staged files unexpectedly.** Our Chapter 1 hook checks linting without silently reformatting code.
3. **Keep hooks focused and fast.** Slow checks encourage developers to bypass them; run comprehensive checks in CI when appropriate.
4. **Commit `.husky/pre-commit`, `.husky/commit-msg`, and `commitlint.config.js`**, but do not commit `node_modules` or private `.env` files.
5. **Do not use `--no-verify` to hide errors.** Git supports it for emergencies, but it bypasses important local checks; treat it as an exceptional, reviewed decision.
6. **Keep npm packages and lockfiles under version control.** The declared versions in `package.json` do not necessarily equal the exact resolved versions in `package-lock.json`.
7. **Use the correct hook for the correct job.** Code linting belongs in `pre-commit`; commit-message validation belongs in `commit-msg`.
8. **When cloning projects**, install dependencies and rerun `npm run prepare` if hooks are not active.
9. **Do not run untrusted scripts** from arbitrary repositories without reviewing them; installing a repository can execute lifecycle scripts.

### GitHub Actions vs Husky

| Aspect | Husky | GitHub Actions |
| --- | --- | --- |
| Location | Developer's machine | CI runner |
| Trigger | Local Git hook | Push or PR |
| Feedback | Before commit | After push / PR |
| Can it be bypassed? | Locally, yes | Required CI checks can be enforced |
| Course chapter | Chapter 1 | Chapter 5 |

**Optional advanced note:** Git's `--no-verify` can skip the `pre-commit` and `commit-msg` hooks for a given `git commit`. This is a diagnostic escape hatch, **not** a normal workflow recommendation.

## 11. Quick reference for future projects

### Existing Chapter 1 repository

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
cd from-zero-to-fullstack-ch1-foundations
npm ci
npm run prepare
git config --get core.hooksPath
npm run lint
```

### New project setup (after Git and Node are installed)

```powershell
git init -b main
npm init -y
npm install -D husky @commitlint/cli @commitlint/config-conventional eslint
npm pkg set "type=module"
npm pkg set "scripts.lint=eslint ."
npx husky init
# Create eslint.config.js and edit .husky/pre-commit in VS Code.
# Create commitlint.config.js and .husky/commit-msg in VS Code.
npm run prepare
```

**Reusable files:**

**`.husky/pre-commit`**

```sh
npm run lint
```

**`.husky/commit-msg`**

```sh
npx commitlint --edit "$1"
```

**`commitlint.config.js`** (ESM project)

```js
export default {
  extends: ['@commitlint/config-conventional'],
}
```

Commit format: `type(scope): subject` — e.g., `docs: add setup instructions`.

## 12. Reuse checklist

- [ ] Git, Node.js, and npm are installed and verified.
- [ ] The current folder is the correct Git repository root.
- [ ] `package.json` defines `lint` and `prepare` scripts.
- [ ] ESLint is installed and its configuration works.
- [ ] Husky and both Commitlint packages are installed as development dependencies.
- [ ] `.husky/pre-commit` runs the appropriate quality check.
- [ ] `.husky/commit-msg` reads Git's proposed message via literal `"$1"`.
- [ ] `commitlint.config.js` loads the conventional preset.
- [ ] `git config --get core.hooksPath` confirms hook initialization.
- [ ] A bad commit message is rejected in a safe practice branch/repository.
- [ ] A deliberate lint error is rejected in a safe practice branch/repository.
- [ ] A good commit passes, and no secrets enter Git history.
- [ ] CI remains the separate enforcement layer for protected branches.

**Outcome:** You can replicate a professional pre-commit code-quality check and conventional commit-message validation in any future JavaScript/React project.

## 13. Source evidence and official references

**Files reviewed in the actual Chapter 1 repository:**

- [Chapter 1 source](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)
- [package.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/package.json)
- [.husky/pre-commit](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.husky/pre-commit)
- [.husky/commit-msg](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.husky/commit-msg)
- [commitlint.config.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/commitlint.config.js)
- [eslint.config.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/eslint.config.js)

**Official external documentation:**

- [Git hooks reference](https://git-scm.com/docs/githooks)
- [Husky — Get Started](https://typicode.github.io/husky/get-started.html)
- [Husky — How To / Windows / CI](https://typicode.github.io/husky/how-to.html)
- [Commitlint — Getting Started](https://commitlint.js.org/guides/getting-started.html)
- [Commitlint — Local Setup](https://commitlint.js.org/guides/local-setup.html)
- [Conventional Commits specification](https://www.conventionalcommits.org/en/v1.0.0/)

**Verification boundary:** File contents, scripts, and declared package ranges were reviewed against the GitHub source on 13 October 2026. The new-project practice lab is an instructional implementation drawn from the official tool documentation. Passing results listed above are **expected outcomes**; they are not a claim of testing every Windows installation or GUI configuration.

---

**Chapter 1 is now documented through Cheatsheets 01–05.**  
**Next:** Cheatsheet 06 — Node.js Fundamentals & Native HTTP Server (Chapter 2).  
[Back to Chapter 1](../README.md) · [Full cheatsheet roadmap](../../README.md)