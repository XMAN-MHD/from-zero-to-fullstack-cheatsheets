# Cheatsheet 21 — GitHub Actions CI from Scratch

**From Zero to Fullstack · Chapter 05: DevOps**  
**Audience:** Absolute beginners building their first automated quality checks, and developers reusing CI in Node.js/React projects  
**Primary platform:** Windows 10/11 · PowerShell · Git · GitHub · Node.js  
**Version:** 1.0 · 22 October 2026  
**Source project:** [Chapter 05 — DevOps](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops)  
**Method:** **Learn → Practice → Document → Reuse**

> **Accuracy and testing scope.** **SOURCE PROJECT** refers to files inspected in the actual Chapter 05 GitHub repository, especially `.github/workflows/ci-cd.yaml`. **LAB** refers to the separately created zero-dependency Node.js teaching project included in the ZIP; it is **not** the source repository. **IMPROVEMENT** is a recommended change, not an implemented commit. In this preparation environment, **three backend HTTP tests passed**, frontend and backend lint commands passed, and the frontend's small build succeeded. The lab's YAML was parsed and structurally checked. **No GitHub-hosted runner execution was performed**, and the source repository's real Jest/MongoDB, Vite build, and Render deployment were not exercised here.

## 1. What you will learn

After this cheatsheet, you should be able to:

1. Distinguish **CI** (Continuous Integration) from **CD** (Continuous Delivery/Deployment).
2. Explain GitHub Actions **events**, **workflows**, **jobs**, **steps**, **runners**, **actions**, and **logs**.
3. Verify Git, Node.js, npm and the repository setup on Windows.
4. Find and read `.github/workflows/ci-cd.yaml` in the real Chapter 05 project.
5. Explain two parallel CI jobs for a **root-level frontend** and a **`backend/` directory**.
6. Use `actions/checkout`, `actions/setup-node`, `npm ci`, lint, build, and Jest.
7. Understand npm caching and the correct path to the backend lockfile.
8. Create a read-only CI workflow for pushes and pull requests.
9. Validate YAML syntax and run the equivalent commands locally.
10. Debug failing jobs using the Actions tab and individual step logs.
11. Apply least-privilege workflow permissions and avoid secret exposure.
12. Identify which parts of the source workflow already perform **deployment**, to be studied in Cheatsheet 22.

### Mental model

```text
Developer -> commit -> push or pull request
                          |
                          v
             .github/workflows/ci-cd.yaml
                          |
                  event matches?
                          |
                +---------+---------+
                |                   |
                v                   v
            Frontend CI         Backend CI
            npm ci              npm ci
            npm run lint        npm run lint
            npm run build       npm test
                |                   |
                +------ status -----+
                          |
                     PASS / FAIL
                          |
        Safe to review/merge? (policy-dependent)
                          |
           Source project also has a deploy job
             (covered separately in #22)
```

**Important:** In the source workflow, `frontend` and `backend` jobs have no dependency on each other. They **may run in parallel**. Only `deploy` has `needs: [frontend, backend]` and waits for both to finish successfully. The source Chapter 05 README depicts them sequentially for simplicity; the actual workflow YAML is authoritative.

## 2. CI, CD, and what GitHub Actions does

**Continuous Integration** means validating code changes regularly and automatically: install exact dependencies, lint code, run tests, and produce a build. A failed check is actionable information, not a guarantee that production is broken. Conversely, a green run does **not** prove absence of bugs or security vulnerabilities.

**Continuous Deployment** is different: an approved, validated change is automatically released to an environment. **Continuous Delivery** can stop at a deployable artifact awaiting approval. The source Chapter 05 already includes two Render deploy-hook calls; this guide will document their existence but focus on the **CI jobs**, not production release setup.

| Concept | Meaning | Real Chapter 05 example |
| --- | --- | --- |
| Repository | Source code and Git history | `from-zero-to-fullstack-ch5-devops` |
| Workflow | One YAML automation definition | `.github/workflows/ci-cd.yaml` |
| Event | What triggers a run | `push` and `pull_request` to `main` |
| Job | Steps executing on a runner | `frontend`, `backend`, `deploy` |
| Step | A shell command or reusable action | `npm ci`, `npm run lint` |
| Action | Prebuilt automation | `actions/checkout@v4` |
| Runner | Machine executing a job | Ubuntu runners |
| Artifact | File saved from a run | Optional frontend build folder `dist/` |
| Cache | Reused dependencies or package data | npm download cache via `setup-node` |
| Secret | Protected external value | Optional Render deploy hook in source workflow |

## 3. Setup from zero on Windows and PowerShell

**Prerequisites:** a GitHub account; permission to push a GitHub repository; Git for Windows; a compatible Node.js version with npm; VS Code or another text editor; and an internet connection to access GitHub. GitHub-hosted Actions itself is **not installed on your PC**.

Install Git from <https://git-scm.com/downloads/win> and Node.js from <https://nodejs.org/en/download>. Pick an actively supported Node.js version appropriate for your project. The real Chapter 05 workflow specifies **Node.js 22.x**; matching that major locally reduces surprises but does not imply that it remains the newest LTS forever.

Open **PowerShell** and verify:

```powershell
git --version
node --version
npm.cmd --version
```

If PowerShell blocks `npm.ps1`, use `npm.cmd` instead of globally weakening your script execution policy. Verify Git identity before committing:

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Clone the *actual* Chapter 05 repository:

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops.git
cd from-zero-to-fullstack-ch5-devops
git status
Get-ChildItem .github/workflows
Get-Content .github/workflows/ci-cd.yaml
```

**Repository structure — do not accidentally invent `frontend/`:**

```text
from-zero-to-fullstack-ch5-devops/
|-- package.json                 <-- FRONTEND at repository root
|-- package-lock.json
|-- src/                         <-- React/Vite source
|-- .github/
|   `-- workflows/
|       `-- ci-cd.yaml
|-- backend/
|   |-- package.json
|   |-- package-lock.json
|   |-- jest.config.json
|   `-- src/
|       |-- __tests__/posts.test.js
|       `-- test/globalSetup.js
`-- compose.yaml
```

The source project already has the workflow. To practice safely, use a **new branch or separate test repository** rather than immediately modifying or enabling a real deployment.

## 4. Inspect the actual workflow — SOURCE PROJECT

The following is the important behavior of `.github/workflows/ci-cd.yaml` as observed in the repository:

```yaml
name: Blog CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  frontend:
    name: Frontend CI
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.x
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Run ESLint
        run: npm run lint
      - name: Build frontend
        run: npm run build

  backend:
    name: Backend CI
    runs-on: ubuntu-22.04
    defaults:
      run:
        working-directory: ./backend
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.x
          cache: npm
          cache-dependency-path: backend/package-lock.json
      - name: Install dependencies
        run: npm ci
      - name: Run ESLint
        run: npm run lint
      - name: Run Jest tests
        run: npm test
```

**This is a focused excerpt of the source workflow, not the entire file.** The original continues with a `deploy` job, on pushes to `main` only, depending on **both** jobs and using Render deploy-hook secrets. The source's `deploy` job is studied in Cheatsheet 22; it is **not included** in our independent CI-only lab.

### 4.1 Why two jobs?

Frontend and backend have different scripts and dependencies. Their checks should fail independently, making diagnostics clearer. `frontend` installs root dependencies, runs ESLint, and builds Vite. `backend` switches into `backend/`, runs ESLint, and executes Jest tests.

### 4.2 `npm ci` vs `npm install`

- `npm install` is appropriate while developing and intentionally updating dependencies or lockfiles.
- `npm ci` is designed for reproducible automated installs and expects a **matching committed `package-lock.json`**; it fails instead of modifying the lockfile to resolve mismatches.
- **Root install:** `npm ci` uses `package-lock.json` at the repository root.
- **Backend install:** a step running with `working-directory: ./backend` uses `backend/package-lock.json`.

Do not use `npm install` to hide a CI lockfile mismatch. Fix and commit the lockfile locally.

### 4.3 Why `cache-dependency-path` matters

The frontend caches packages based on the root lockfile by default. The backend's `setup-node` cache lookup executes at the repository level, so the real source explicitly provides `backend/package-lock.json`. The npm **cache is not a substitute for `npm ci`** and does not directly restore the project's `node_modules` directory.

### 4.4 Jest and MongoDB Memory Server

The source `backend/package.json` defines:

```json
{
  "scripts": {
    "lint": "eslint .",
    "test": "cross-env NODE_OPTIONS=--experimental-vm-modules jest"
  }
}
```

The source `backend/jest.config.json` configures a Node test environment plus global setup, teardown, and per-test setup. The global setup creates `MongoMemoryServer` requesting **MongoDB binary version `7.0.25`**; per-test setup connects Mongoose, and global teardown stops the memory server.

**Consequences:** GitHub runners may need network access to download the MongoDB binary. Startup can fail because of unsupported OS/runtime combinations, downloads, or binary dependencies. A green Jest job is **not verified by reading YAML**; inspect actual run logs. Never claim source integration tests passed unless executed.

## 5. Reading GitHub Actions YAML, line by line

GitHub workflow files belong under `.github/workflows/` and commonly end in `.yml` or `.yaml`. YAML uses **spaces**, not tabs. Here are the most useful keys:

| Key | What it controls | Example |
| --- | --- | --- |
| `name:` | Workflow title in Actions UI | `name: Blog CI` |
| `on:` | Triggering events | `push`, `pull_request` |
| `jobs:` | Job definitions | `frontend`, `backend` |
| `runs-on:` | Machine image | `ubuntu-latest` |
| `steps:` | Ordered actions within a job | `checkout`, `npm ci` |
| `uses:` | Referenced action | `actions/setup-node@v4` |
| `run:` | Shell command | `npm test` |
| `with:` | Inputs to an action | `node-version: '22.x'` |
| `defaults.run.working-directory:` | Directory for **run** steps | `./backend` |
| `needs:` | Explicit job dependencies | `needs: [frontend, backend]` |
| `if:` | Conditional execution | push to main only |
| `permissions:` | Token scopes | `contents: read` |
| `concurrency:` | Cancel superseded runs | `cancel-in-progress: true` |

**Subtle detail:** `defaults.run.working-directory` affects shell `run:` steps, **not** every `uses:` action. That is why `setup-node` still needs `cache-dependency-path: backend/package-lock.json`.

**Trigger semantics:** `push.branches: [main]` means pushes to `main`; for `pull_request`, the `branches` filter refers to the **target/base branch**, not necessarily the feature branch. GitHub may require approval before certain workflows from external contributors execute.

## 6. A CI-only workflow you can reuse — LAB / IMPROVEMENT

Use this in a separate educational repository. It intentionally includes **no production deploy hook**. The `actions/checkout@v4` and `actions/setup-node@v4` versions mirror the inspected source; review newer maintained releases and runner compatibility before upgrading. For production-sensitive repositories, pin actions to vetted full commit SHAs as part of your supply-chain policy.

Save as `.github/workflows/ci.yml` at the repository root:

```yaml
name: CI - frontend and backend

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  frontend:
    name: Frontend lint and build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: npm
      - name: Install frontend dependencies
        run: npm ci
      - name: Lint frontend
        run: npm run lint
      - name: Build frontend
        run: npm run build

  backend:
    name: Backend lint and tests
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./backend
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: npm
          cache-dependency-path: backend/package-lock.json
      - name: Install backend dependencies
        run: npm ci
      - name: Lint backend
        run: npm run lint
      - name: Test backend
        run: npm test
```

**Why these additions?** `permissions: contents: read` gives the workflow a minimal repository-read token. `persist-credentials: false` avoids retaining checkout credentials for later scripts. `concurrency` can cancel older runs for the same ref, saving time; omit it if you need every historical run completed. No secrets are required for this CI-only workflow.

**Do not copy this file into the source repository and claim it is already running there.** This is a stand-alone, reviewed teaching version intended for your own safe testing.

## 7. Build the independent CI laboratory

The ZIP includes `lab/`, with a *deliberately tiny*, dependency-free web build and an HTTP backend. It teaches **workflow mechanics** without requiring MongoDB downloads or a paid deployment service. It is **not the MERN source project**, nor a replacement for the source Jest suite.

```text
lab/
|-- .github/workflows/ci.yml
|-- package.json                 <-- mini frontend at root
|-- package-lock.json
|-- src/main.js
|-- scripts/build.mjs            <-- writes dist/index.html
|-- backend/
|   |-- package.json
|   |-- package-lock.json
|   |-- src/server.mjs           <-- built-in Node HTTP
|   `-- test/server.test.mjs     <-- Node test runner
|-- .gitignore
`-- README.md
```

Extract the ZIP and open the `lab` directory in VS Code. From a **Windows PowerShell** terminal:

```powershell
# Run from the extracted lab/ directory
npm.cmd ci
npm.cmd run lint
npm.cmd run build
Test-Path .\dist\index.html

cd backend
npm.cmd ci
npm.cmd run lint
npm.cmd test
cd ..
```

**Expected local behavior:** `dist/index.html` is created and the backend reports **three passing tests**. During authoring, the actual lab passed these commands using Node.js 22, including all three HTTP tests. Dependency installations are minimal because the lab does not use React, Express, or MongoDB. That is intentional: the CI pattern is transferable.

The three tests check: a successful `GET /health`, a `404` for an unknown URL, and a `404` for an unsupported method. To inspect them, open `backend/test/server.test.mjs`.

### 7.1 Why a real HTTP test is useful

The backend test starts a temporary server bound to loopback and a random free port, calls it using Node's built-in `fetch()`, asserts status and JSON content, then closes the server. This tests more than one isolated function while remaining quick and deterministic.

### 7.2 What the lab does *not* prove

The lab does not test the original React interface, Jest, Mongoose, networked MongoDB, Render, or action execution on GitHub-hosted runners. You must run the actual project's commands and inspect its Actions run to validate those systems.

## 8. Run the same checks in the actual Chapter 05 project

Open a clean clone of the Chapter 05 project and run the same steps the real GitHub workflow uses:

```powershell
# From the Chapter 05 repository root
npm.cmd ci
npm.cmd run lint
npm.cmd run build

cd backend
npm.cmd ci
npm.cmd run lint
npm.cmd test
cd ..
```

The actual frontend uses **Vite** and the actual backend uses **Jest**; these may require downloads not necessary in the lab. The source frontend also uses `PropTypes` in several components without declaring `prop-types` in the inspected root manifest, and `DeletePost.jsx` references `DeletePostButton.propTypes` although the exported function is `DeletePost`. These are potential causes of build/lint failures that must be fixed in the source project if they are still present. Do **not** edit production files blindly or infer that `npm ci` or the build already pass.

**Pre-flight checklist:** confirm both lockfiles are tracked; `node_modules` and `.env` are ignored; the scripts exist in the relevant package; the `backend/` directory is correctly named; and no local-only environment assumption is hidden in the tests.

## 9. Publish your first CI workflow safely

Prefer creating a separate training repository. Then:

1. Copy the extracted `lab/` contents, including `.github/` and both `package-lock.json` files.
2. Initialize Git if necessary and create a remote repository in GitHub.
3. Commit changes on `main` or create a feature branch and pull request into `main`.
4. Open **GitHub → Repository → Actions → CI - frontend and backend**.
5. Click the latest run and inspect the `Frontend lint and build` and `Backend lint and tests` jobs.
6. Confirm successful check marks **only after GitHub actually runs them**.

For a brand-new test repo with an existing `origin` remote, an example PowerShell sequence is:

```powershell
git status
git add .github package.json package-lock.json src scripts backend .gitignore README.md
git commit -m "ci: add beginner frontend and backend checks"
git push origin main
```

Do **not** run `git push` against the original chapter repository unless you deliberately intend to change it. The copied lab has no deployment job. A branch-protection rule may be configured to require CI checks before merging; the availability and names of these controls depend on repository permissions and account policies.

### 9.1 Red/green exercise

In the **test repository only**, change the assertion in the `GET /health` test from `200` to `201`, commit on a feature branch, and open a pull request targeting `main`. The backend should become red if the job actually runs. Then restore `200` and push again. This is a deliberate, reversible exercise; it is not a bug fix for the real Chapter 05 API.

## 10. Read an Actions run like a professional

Open the Actions tab, choose a run, then open a job. The left panel shows steps; each step expands to logs. Find the **first actual error**, not just the final generic message `Process completed with exit code 1`.

| Symptom in logs | Likely explanation | First investigation |
| --- | --- | --- |
| `npm ci` fails | Lockfile mismatch or wrong directory | Compare `package.json` and lockfile, check path |
| `npm ERR! Missing script: lint` | Wrong package or script absent | Inspect that folder's `package.json` |
| `Cannot find module 'prop-types'` | Source manifest missing dependency | Install and commit the appropriate dependency locally |
| `DeletePostButton is not defined` | Existing component name mismatch | Inspect `src/components/DeletePost.jsx` |
| Jest times out | MongoDB Memory Server binary/startup | Inspect `globalSetup.js`, network, runner OS and binary version |
| `No such file or directory: backend` | Wrong repository structure | Use actual root-plus-`backend/` layout |
| Workflow not listed | Bad path, YAML or repo policy | Check `.github/workflows/`, branch, Actions settings |
| No job after a PR | Trigger/filter or approval rules | Confirm PR targets `main` and approval state |
| Backend cache missing | Lockfile cache path | Set `cache-dependency-path: backend/package-lock.json` |
| Deploy step fails | Secrets/Render hook configuration | Treat as CD issue; see Cheatsheet 22 |

**Failure hygiene:** Do not paste secrets into issues, logs, screenshots, or assistants. Use job-level diagnostics, non-sensitive reproduction steps, and locally reproducible commands.

## 11. Cache, artifacts and reproducibility

**npm caching:** `cache: npm` in `actions/setup-node` caches downloads used by npm. It can speed up future jobs. It does **not** replace `npm ci`. Each job has a fresh workspace and must install its own dependencies. A job cache is not a database and should never hold secrets.

**Artifacts (optional improvement):** if you need to inspect a frontend output after a successful run, add a supported version of `actions/upload-artifact` after `npm run build`, with path `dist/` and an appropriate retention policy. This is separate from the source workflow; it currently builds but does not upload a frontend artifact. Review available action versions, network policies and artifact restrictions before choosing the action. Do not store `.env`, private tokens, credentials, or database dumps in artifacts.

**Reproducibility:** commit lockfiles, use declared Node versions, keep scripts deterministic, seed test data explicitly, and avoid tests whose correctness depends on a particular wall-clock instant.

## 12. Security rules for GitHub Actions

**The workflow is code that runs with access to repository and runner resources.** Treat changes to `.github/workflows/` as sensitive code review material.

1. Start with `permissions: contents: read` if no writes are needed. Grant extra permissions only to jobs that genuinely require them.
2. Avoid running untrusted pull-request code with production secrets. Prefer standard `pull_request` CI and avoid `pull_request_target` with untrusted checkout or scripts.
3. Use supported official actions. For stronger supply-chain guarantees, **pin actions to a reviewed full commit SHA** rather than a mutable version tag. The lab mirrors the source's `@v4` tags for learning, **not** as the strictest security posture.
4. Never echo secrets or embed tokens in command-line output. Store deploy credentials or hook URLs in Actions secrets or environments with access controls.
5. Keep `GITHUB_TOKEN` privileges minimal. Be cautious with checkout credential persistence and third-party actions.
6. Require branch review and successful checks where supported by the repository's rules and plan.
7. Keep lockfiles and actions updated using a deliberate review process; updates can change runtime requirements.
8. Use separate, appropriately protected deployment jobs/environments. **Do not grant deployment secrets to ordinary PR CI.**

The real source file has no explicit top-level `permissions` declaration and calls external Render hooks in its deploy job. Adding least-privilege permissions and reviewing secret usage are **recommendations**, not statements that the source repository already follows them.

## 13. Source deploy job: why it belongs to Cheatsheet 22

The **SOURCE PROJECT** has a third job called `deploy` with the following essential behavior:

```yaml
  deploy:
    name: Deploy to Render
    runs-on: ubuntu-latest
    needs:
      - frontend
      - backend
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: Trigger Render backend deploy
        run: curl -X POST "${{ secrets.RENDER_BACKEND_DEPLOY_HOOK }}"
      - name: Trigger Render frontend deploy
        run: curl -X POST "${{ secrets.RENDER_FRONTEND_DEPLOY_HOOK }}"
```

**Meaning:** the job is eligible only on a **push to `main`**, after both CI jobs succeed. It does not deploy on the ordinary PR runs of this workflow. The two secret names are visible in the YAML, but their **values, existence, validity and deployment outcomes were not verified**. A deploy hook can trigger a service build or release, but a successful `curl` alone does **not** necessarily prove the app is healthy. Production deployment, service configuration, hooks, approvals, monitoring and rollback belong in **Cheatsheet 22 — Continuous Deployment with Render**.

## 14. Quality gates, job dependencies and status checks

**Without `needs`:** `frontend` and `backend` can start concurrently; one passing does not imply the other passed. **With `needs`:** a dependent job waits for successful completion of its prerequisites by default. The source uses `needs` only on `deploy`.

A status check is useful for a pull request when repository rules require it. However, simply creating a workflow **does not automatically protect `main`**. Branch protection or repository rules must be configured separately by someone authorized to do so. Never declare a branch protected solely because a CI file exists.

**Suggested policy:** require reviews and both relevant CI jobs to pass before merging. Keep optional deployment separate and protected. Assess how workflow skips and path filters may affect required checks.

## 15. Testing strategy: what runs where?

| Level | Example | Tool used | Scope |
| --- | --- | --- | --- |
| Syntax/lint | JS syntax or ESLint rules | `npm run lint` | Static checks; not runtime behavior |
| Unit tests | Pure helpers | Jest or Node test runner | Isolated logic |
| Integration | Mongoose + MongoMemoryServer | Source backend Jest | DB-backed service behavior |
| Build | Vite production bundle | `npm run build` | Bundling / dependencies |
| API smoke test | GET `/health` | LAB Node HTTP test | End-to-end HTTP in local process |
| Deployment smoke test | Deployed URL returns expected result | Not performed here | Real hosted service only |

**Do not equate passing unit tests with a production-ready application.** Each level reveals different failure modes. The learner should eventually run real React integration and browser tests, plus external service smoke checks, before production release.

## 16. Quick command reference

| Goal | PowerShell or command |
| --- | --- |
| Inspect workflow | `Get-Content .github/workflows/ci-cd.yaml` |
| Check Git state | `git status` |
| Confirm tools | `git --version`, `node --version` |
| Install root dependencies | `npm.cmd ci` |
| Lint frontend | `npm.cmd run lint` |
| Build frontend | `npm.cmd run build` |
| Install backend dependencies | `cd backend; npm.cmd ci` |
| Backend lint | `npm.cmd run lint` in `backend/` |
| Backend tests | `npm.cmd test` in `backend/` |
| Return to root | `cd ..` |
| Commit workflow | `git add .github/workflows/ci.yml` |
| Push training repository | `git push origin main` |
| View run | GitHub repository → Actions → workflow → run |

### Exit codes and expected results

Successful shell commands exit with code **0**. CI treats a nonzero exit as a failed step unless special behavior is configured. That failure normally causes the job to be marked unsuccessful and can block a dependent deployment step. A job skipped by a condition is not the same as a successfully executed job.

## 17. Beginner troubleshooting: YAML gotchas

- **Indentation:** use spaces (commonly two), not tabs. Nest `steps:` under its job.
- **`on` and YAML parsers:** GitHub Actions treats `on:` as a workflow key. Some generic YAML 1.1 parsers mistakenly interpret `on` as Boolean `true`. Use an appropriate Actions-aware parser or a parser configured for YAML 1.2 / base-string loading. This does not imply GitHub is wrong.
- **Wrong folder:** Chapter 05 has frontend at root; blindly copying a `frontend/` working directory from a different chapter causes failure.
- **Wrong case:** use `.github/workflows/` exactly; workflow files have `.yml` or `.yaml` extensions.
- **Unknown job output:** check named steps; the last summary may mask the first error.
- **Permissions:** organization or repository policies may disable Actions or require manual approval.
- **Node version:** the real workflow uses `22.x`; a different local version can change behavior.
- **Source project test binary:** MongoDB Memory Server requests version `7.0.25`; if the test fails, inspect binary availability and runner environment rather than hardcoding a fake passing result.

## 18. Practice exercises

1. **Read:** draw the Chapter 05 jobs and identify which ones run in parallel.
2. **Run:** execute `npm ci`, lint, build and tests in the independent lab.
3. **Observe:** upload the lab to a new test repo and inspect Actions logs.
4. **Break safely:** force a backend test assertion to fail in a training PR and confirm that CI turns red.
5. **Fix:** restore the assertion and confirm the new run turns green.
6. **Analyze:** explain why `cache-dependency-path` is needed for `backend/package-lock.json`.
7. **Secure:** explain why `permissions: contents: read` and no deploy secrets are good CI defaults.
8. **Compare:** find the source project's separate Render `deploy` job and explain its `needs` and `if` keys.

## 19. Reuse checklist

- [ ] Correct repository checked out; **root frontend + `backend/`** verified.
- [ ] Workflow stored inside `.github/workflows/` and YAML passes validation.
- [ ] Only intended events and branches trigger CI.
- [ ] Both `package-lock.json` files tracked and matching manifests.
- [ ] `npm ci` works in root and backend.
- [ ] `npm run lint` exists in both packages.
- [ ] Frontend build succeeds and creates the expected output.
- [ ] Backend tests pass, including any MongoDB test setup.
- [ ] CI has minimum required token permissions; secrets are not printed.
- [ ] Third-party and official action versions reviewed; stronger SHA pinning considered.
- [ ] Real GitHub Actions run inspected; local-only tests are not overstated.
- [ ] Branch rules configured if successful CI is required for merge.
- [ ] Deployment is handled separately with gated access (Cheatsheet 22).

## 20. Official references, source notes and next step

**Official documentation (consulted for workflow design and current practices):**

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Building and testing Node.js](https://docs.github.com/en/actions/tutorials/build-and-test-code/nodejs)
- [Workflow syntax reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Triggering a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)
- [GitHub Actions secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [actions/checkout](https://github.com/actions/checkout)
- [actions/setup-node](https://github.com/actions/setup-node)
- [Node.js downloads](https://nodejs.org/en/download)

**Project files inspected:**

- [`Chapter 05/.github/workflows/ci-cd.yaml`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops/blob/main/.github/workflows/ci-cd.yaml)
- [`Chapter 05/package.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops/blob/main/package.json)
- [`Chapter 05/backend/package.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops/blob/main/backend/package.json)
- [`Chapter 05/backend/jest.config.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops/blob/main/backend/jest.config.json)
- [`Chapter 05/backend/src/test/globalSetup.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops/blob/main/backend/src/test/globalSetup.js)

**Authoring verification:** three native Node HTTP tests passed; two lint commands passed; mini frontend build passed; workflow YAML was parsed and structurally inspected. The lab's GitHub Actions workflow, the source project's real CI execution, and external deployment **were not run** in GitHub's hosted environment. The independent lab's exact versions and instructions should be checked again when reused in the future.

**Next:** **Cheatsheet 22 — Continuous Deployment with Render**, the final chapter guide of our initial 22-cheatsheet roadmap. It will explain how to deploy only after validated CI, manage deployment secrets, verify a live service, and prepare rollback or recovery procedures.

---

*Learn → Practice → Document → Reuse.*