# Cheatsheet 22 — Continuous Deployment with Render

**From Zero to Fullstack · Chapter 05: DevOps**  
**Audience:** Beginner full-stack developers and returning developers creating a safe release workflow  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code · GitHub Actions · Render  
**Version:** 1.0 · 22 October 2026  
**Source project:** [Chapter 05 — DevOps](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops)  
**Learning method:** **Learn → Practice → Document → Reuse**

> **Evidence and scope.** **SOURCE PROJECT** refers only to files inspected directly in the real Chapter 05 repository. **INDEPENDENT LAB** refers to the additional Node.js + static HTML teaching project supplied in the ZIP; the lab is deliberately **not React and does not run MongoDB**. **RECOMMENDED CHANGE** denotes code or hosting configuration not confirmed in the source project. During authoring, the lab passed **14 Node.js tests**, its static frontend build completed and its example GitHub Actions YAML was parsed successfully. **No actual Render account, deployment, MongoDB Atlas connection, GitHub Actions runner execution or public URL was verified.** A successful deploy-hook request starts or queues a deployment; it does not prove the website is live.

## 1. Learning outcomes

At the end of this last cheatsheet, you should be able to:

1. Distinguish **Continuous Integration**, **Continuous Delivery** and **Continuous Deployment**.
2. Explain the production path: **GitHub → CI → Render backend + Render frontend → MongoDB Atlas**.
3. Configure an Express web service and a Vite/React static site in Render using this project's actual folder layout.
4. Set safe runtime and build-time environment variables without committing credentials.
5. Connect a Render backend to MongoDB Atlas while respecting network restrictions.
6. Ensure deployment is triggered **only after successful CI**, without accidental duplicate auto-deploys.
7. Store separate Render deploy-hook URLs as GitHub Actions secrets.
8. Explain `needs`, push-vs-pull-request conditions and restricted job permissions.
9. Distinguish **deployment requested** from **deployment completed, healthy and verified**.
10. Diagnose startup, CORS, API prefix, build and deployment failures; plan a rollback.
11. Run an independent local release-gating laboratory without Render credentials.
12. Reuse a safe release checklist in later MERN projects.

### The big picture

```text
Developer pushes to GitHub main
         |
         v
GitHub Actions workflow
    |                 |
Frontend: lint/build  Backend: lint/tests
    +--------+--------+
             | both pass
             v
    Deployment job (main push only)
       |                  |
       v                  v
Render Web Service    Render Static Site
Express /api/v1      Vite/React compiled files
       |                  |
       v                  | browser requests to API URL
   MongoDB Atlas <-------+

Deployment request ≠ finished deployment ≠ verified release.
```

## 2. Prerequisites and system setup (Windows)

You should understand Cheatsheets **01–21** and have accounts for GitHub, Render and—if you want a real MERN release—a managed MongoDB database such as Atlas. Local prerequisites: Git, Node.js compatible with the project, VS Code, a browser and PowerShell. Docker is **optional** for the Render native Node/static-site route explained here.

Verify the tools:

```powershell
node --version
npm.cmd --version
git --version
```

`npm.cmd` is useful if Windows PowerShell blocks the `npm.ps1` shim. Follow your organisation's script-execution policy rather than bypassing security controls blindly.

Clone the **source application**, not the cheatsheets repository:

```powershell
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops.git
cd from-zero-to-fullstack-ch5-devops
npm.cmd ci
npm.cmd run lint
npm.cmd run build
cd backend
npm.cmd ci
npm.cmd run lint
npm.cmd test
cd ..
```

**Expected:** these are checks to run, not proof that the current source passes. Both dependency trees have a `package-lock.json`. The backend Jest suite uses MongoDB Memory Server and can need a MongoDB binary download and a compatible runner. The frontend source references `prop-types` without listing it in its `package.json` and contains a `DeletePostButton.propTypes` reference in `src/components/DeletePost.jsx` although the component is named `DeletePost`. Correct these locally and re-run validation before a production release; **this documentation does not change the source project**.

## 3. Vocabulary: CI, delivery and deployment

| Term | Meaning | What it proves |
| --- | --- | --- |
| **CI (Continuous Integration)** | Automated checks for every proposed integration | The configured tests, lint and builds ran successfully |
| **Continuous Delivery** | Code is kept deployable; a person may approve the release | The release candidate is prepared, not necessarily published |
| **Continuous Deployment** | A passing change is released automatically, subject to configured gates | The automation attempted a release; readiness still needs verification |
| **Deploy hook** | Secret URL that requests a Render service deployment | Request was accepted or queued, **not** that deployment succeeded |
| **Health check** | Probe indicating whether an instance can serve requests | A particular instance is responding; business workflows need separate tests |
| **Rollback** | Restore a known-working release | A recovery action that must be verified afterwards |

A public production release has three distinct checkpoints: **(1) CI passed**, **(2) Render deployed successfully**, **(3) user-critical API/UI journeys were verified**. Do not collapse these into one green badge.

## 4. SOURCE PROJECT: what actually exists

The real Chapter 05 GitHub repository contains:

```text
from-zero-to-fullstack-ch5-devops/
├── .github/workflows/ci-cd.yaml
├── package.json                   # React/Vite at repo root
├── src/api/posts.js               # fetch() uses VITE_BACKEND_URL
├── dockerfile                     # lowercase; Dockerfile alternative
├── compose.yaml                   # blog-frontend + blog-backend
├── backend/
│   ├── package.json               # Node/Express/Mongoose
│   ├── dockerfile                 # lowercase
│   └── src/
│       ├── index.js               # waits for database, starts app
│       ├── app.js                 # express.json() + cors()
│       └── db/init.js             # mongoose.connect()
└── README.md
```

The `.github/workflows/ci-cd.yaml` file runs on `push` and `pull_request` targeting `main`. It has two independent jobs:

- `frontend`: root-level `npm ci`, `npm run lint`, `npm run build`.
- `backend`: `working-directory: ./backend`, `npm ci`, `npm run lint`, `npm test`.

The `deploy` job declares `needs: [frontend, backend]` and has this condition:

```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

It POSTs separately to `${{ secrets.RENDER_BACKEND_DEPLOY_HOOK }}` and `${{ secrets.RENDER_FRONTEND_DEPLOY_HOOK }}` with `curl -X POST`. **Those two secret names are established by the source code; their values and even their existence in repository settings have not been inspected.** The workflow does **not** wait for Render deployment completion or verify a deployed release. It also does not contain a database hosting service definition.

> **Security audit warning.** The source repository tree includes a tracked top-level `.env` file. Its contents were deliberately **not read or reproduced** in this guide. Check whether any secrets were committed, rotate exposed credentials if necessary, remove secret files from Git tracking and consider history cleanup if warranted. A `.gitignore` entry does not retroactively untrack a committed file.

## 5. Choose the production hosting layout

For a beginner-friendly native Render setup, use three independently managed parts:

| Component | Recommended hosting | Location and essential setting |
| --- | --- | --- |
| **Frontend** | Render **Static Site** | Repository root; `npm ci && npm run build`; publish `dist` |
| **Backend** | Render **Web Service** (Node) | Root Directory `backend`; `npm ci`; `npm start` |
| **Database** | MongoDB Atlas or suitable managed MongoDB | Private `DATABASE_URL` configured in backend environment |

This matches the existing repo's real structure, where `package.json` for Vite lives **at the root**, whereas the Express application lives under `backend/`. Source deployment settings in the Render Dashboard are **not** available for inspection, so these are **recommended values**, not confirmed deployment settings.

A different production approach is to deploy Docker images, but then build contexts, image publishing and Render Docker settings have to be configured deliberately; our Chapter 20 covers Docker. Do **not** assume a local `compose.yaml` is automatically run by a native Render web service. The current Compose file has only two services and injects `DATABASE_URL` from outside; it does not start MongoDB.

## 6. Prepare MongoDB Atlas before starting the backend

1. Create an Atlas project and database deployment under your own account.
2. Create a database user with only the permissions the application needs. Use a strong random password.
3. Configure Atlas **Network Access** for the outbound IP address/range used by the Render backend; verify the current address in the Render Dashboard. Avoid opening database access to the entire internet except as a carefully bounded temporary diagnostic measure.
4. Choose the MongoDB Node driver connection format and copy the connection-string template.
5. URL-encode special characters in credentials and choose your database name (for example `blog`).
6. Store the final URI as the backend's Render environment variable `DATABASE_URL`. Never place it in a frontend `VITE_` variable, GitHub commit, screenshot, workflow YAML or release logs.
7. Verify the database connection from **the deployed backend**. Being able to connect locally does not prove Render's outbound access is allowed.

Safe illustration (not a credential):

```text
DATABASE_URL=mongodb+srv://<USERNAME>:<URL_ENCODED_PASSWORD>@<CLUSTER_HOST>/blog?retryWrites=true&w=majority
```

The source's `backend/src/db/init.js` uses `mongoose.connect(DATABASE_URL)` with legacy options `useNewUrlParser` and `useUnifiedTopology`; review those options for the installed Mongoose version. The source also logs the **entire** database URL after successful connection. **RECOMMENDED CHANGE:** log only a generic connection-success message, never a URI containing credentials.

## 7. Create the Render backend web service

1. In Render, choose **New → Web Service** and link the Chapter 05 GitHub repository.
2. Set the branch to `main` and the **Root Directory** to `backend`.
3. Choose Render's Node runtime (native Web Service, not Static Site).
4. Set **Build Command**: `npm ci`.
5. Set **Start Command**: `npm start` (the backend script runs `node src/index.js`).
6. Add `DATABASE_URL` in the backend Render **Environment** panel, not in source control. Render supplies `PORT` by default; the source honours `process.env.PORT`.
7. Select a compute plan that matches your availability and usage needs; check limits, regions and current prices in Render before choosing.
8. Deploy the **first version manually** from the dashboard, inspect logs, then test a safe read endpoint such as `GET /api/v1/posts`.
9. Optionally add a dedicated `/healthz` endpoint and configure an HTTP **Health Check Path** to `/healthz`.

**Important:** Render web services need to listen on `0.0.0.0` at the configured port. Node's `app.listen(PORT)` without a host generally listens on all interfaces, but make that intent explicit when refactoring. Avoid binding only to `127.0.0.1` in production.

### RECOMMENDED CHANGE: a database-aware health endpoint

The following code is **not present in the inspected source**. Add it to `backend/src/app.js` after `const app = express()` if you decide to use an HTTP health check:

```js
import mongoose from 'mongoose'

app.get('/healthz', (req, res) => {
  if (mongoose.connection.readyState !== 1) {
    return res.status(503).json({ status: 'unavailable' })
  }
  return res.status(200).json({ status: 'ok' })
})
```

A `200` on this route is a readiness hint, not an exhaustive application test. A `503` is appropriate while database access is essential and unavailable. Do not expose internal stack traces or database credentials in a public health response.

## 8. Create the Render frontend static site

1. Choose **New → Static Site** and connect the **same** GitHub source repository.
2. Set the branch `main`; leave **Root Directory** empty (or set to the repository root, depending on the dashboard's input conventions).
3. Set **Build Command** to `npm ci && npm run build`.
4. Set **Publish Directory** to `dist` because this project uses Vite's standard output folder.
5. Set this **build-time, public** environment variable:

```text
VITE_BACKEND_URL=https://<YOUR-BACKEND-SERVICE>.onrender.com/api/v1
```

6. Trigger a new frontend **build** after changing this variable; Vite embeds it into generated JavaScript at build time.
7. Confirm the UI's network requests go to `<backend>/api/v1/posts`, not `/posts` at the domain root.
8. If you later introduce client-side routing, configure a Render Static Site **Rewrite** from `/*` to `/index.html`, rather than a redirect; only do so if your routing needs it.

Source `src/api/posts.js` appends `/posts` to `import.meta.env.VITE_BACKEND_URL`. The `VITE_` prefix makes a value accessible in the shipped frontend bundle; **never put passwords, deploy hooks or tokens in this variable**. Because frontend and backend are separate origins, the backend must send appropriate CORS headers for the actual frontend origin.

## 9. Configure CORS deliberately

The real backend's `backend/src/app.js` uses an unrestricted `app.use(cors())`, which is convenient during development but overly broad for a production site. A narrower **RECOMMENDED CHANGE** is:

```js
import cors from 'cors'

const frontendOrigin = process.env.FRONTEND_ORIGIN
if (!frontendOrigin) throw new Error('FRONTEND_ORIGIN is required')
app.use(cors({ origin: frontendOrigin }))
```

Set the backend's `FRONTEND_ORIGIN` to the **exact frontend origin**, e.g. `https://<YOUR-FRONTEND>.onrender.com`, with no path and no trailing `/`. If multiple trusted origins are genuinely needed, use an explicit allowlist. Do not confuse CORS with authentication: it is a browser access control, **not** authorization for an API. For cookie-based authentication, additional credential and cookie settings will be required.

### Production configuration matrix

| Setting | Where it belongs | Is it secret? |
| --- | --- | --- |
| `DATABASE_URL` | Render backend Environment | **Yes** |
| `PORT` | Render backend runtime (usually supplied) | No |
| `FRONTEND_ORIGIN` (proposed) | Render backend Environment | No |
| `VITE_BACKEND_URL` | Render static-site build environment | No; public |
| `RENDER_BACKEND_DEPLOY_HOOK` | GitHub Actions secret | **Yes** |
| `RENDER_FRONTEND_DEPLOY_HOOK` | GitHub Actions secret | **Yes** |

## 10. Select one deployment trigger strategy

Render supports more than one strategy. **Choose one main strategy to avoid duplicate releases and accidentally bypassing CI:**

**Strategy A — Native Render checks integration:** choose **Auto-Deploy: After CI Checks Pass** for each Git-linked Render service. Render then watches the repository checks and deploys when they pass. You do not need deploy-hook steps for that path. Understand which checks Render considers successful and whether any checks are skipped.

**Strategy B — GitHub Actions controls the release (used in our source):** configure **Auto-Deploy: Off** for both Render services, then let the `deploy` job trigger the two secret deploy hooks after its `needs` jobs pass. This avoids the default **On Commit** behaviour deploying a push before CI finishes. Follow Render's manual/deploy-hook instructions to create and inspect deployments.

Our source currently implements a GitHub Actions deploy job; it does not tell us which Render auto-deploy setting the user selected. Therefore **do not claim the release pipeline is safe until Render's Auto-Deploy configuration has been checked**.

## 11. Create and protect the two Render deploy hooks

In the backend Render Web Service dashboard, open **Settings → Deploy Hook**, copy its secret URL to a password-safe temporary location, and save it as a GitHub Actions repository secret named `RENDER_BACKEND_DEPLOY_HOOK`. Repeat for the frontend Static Site using the name `RENDER_FRONTEND_DEPLOY_HOOK`.

GitHub UI: repository **Settings → Secrets and variables → Actions → New repository secret**. Repository settings and access can vary according to account permissions. For an additional approval gate, attach the deploy job to a **production environment** with protection rules that your GitHub plan supports; you may instead configure environment secrets, but keep the workflow's secret scope consistent with where they are stored.

**Important distinctions:**

- A deploy hook is a **secret URL**, not an ordinary public service URL; do not publish it.
- Render accepts HTTP `GET` or `POST` to a valid hook. HTTP `200` indicates a deploy started; `202` can indicate it was queued. It does **not** indicate that the new release became healthy.
- If compromised, regenerate the Render hook immediately and update the corresponding GitHub secret.
- The two hooks trigger **two separate service deployments**. Triggering them sequentially does not mean that the backend became ready before the frontend is deployed.
- Hooks can deploy the latest linked-branch commit. To pin a specific tested commit, Render also supports a `ref` query parameter; review the effect this has on auto-deploys and use your source commit SHA deliberately.

## 12. Understand the existing GitHub Actions deployment gate

In `.github/workflows/ci-cd.yaml`, the source has the essence of this deployment job:

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

`needs` prevents this job from running normally when required CI jobs fail. The `if` expression excludes pull requests and non-`main` branches. In the original workflow, there is **no explicit HTTP `curl` failure flag**, no readiness polling and no post-deployment smoke test. A `curl` command can exit successfully even when the HTTP status is an error, unless instructed to fail. These points are **recommendations**, not claims about steps already implemented.

## 13. RECOMMENDED CHANGE: safer workflow template

The example below illustrates an incremental improvement. Preserve your project's existing frontend/backend CI steps (covered in Cheatsheet 21) and adapt only the deployment job. The YAML is intentionally self-contained enough to understand but is **not automatically installed in the source project**.

```yaml
name: Blog CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run build

  backend:
    runs-on: ubuntu-22.04
    defaults:
      run:
        working-directory: ./backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm
          cache-dependency-path: backend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm test

  deploy:
    needs: [frontend, backend]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Request backend deploy
        env:
          DEPLOY_HOOK: ${{ secrets.RENDER_BACKEND_DEPLOY_HOOK }}
        run: |
          test -n "$DEPLOY_HOOK" || { echo 'Backend deploy hook missing'; exit 1; }
          curl --fail-with-body --silent --show-error --retry 2 \
            --max-time 30 --output /dev/null --request POST "$DEPLOY_HOOK"
      - name: Request frontend deploy
        env:
          DEPLOY_HOOK: ${{ secrets.RENDER_FRONTEND_DEPLOY_HOOK }}
        run: |
          test -n "$DEPLOY_HOOK" || { echo 'Frontend deploy hook missing'; exit 1; }
          curl --fail-with-body --silent --show-error --retry 2 \
            --max-time 30 --output /dev/null --request POST "$DEPLOY_HOOK"
```

**What improved?** Restricted default token permissions, a named deployment environment, checks that required hook variables exist, and `curl --fail-with-body` to fail the job on HTTP errors. The URLs are passed via `env:` rather than interpolated directly into the command text. The output is suppressed to avoid noisy logs. **Never enable this template in an account with production secrets until you have reviewed it.** Branch protections, environment approvals and locked-down permissions should be configured separately.

**What it still does not do:** It does not wait until either Render deploy is live, validate deployed commit identity, enforce backend-before-frontend readiness, execute end-to-end smoke tests, or roll back automatically. Those require additional steps and appropriate Render API access and release-state checks. Also, triggering the latest commit can race with newer pushes; consider a `ref`-pinned deploy tied to the tested SHA or Render's native **After CI Checks Pass** mode. Do not claim this snippet is a fully atomic zero-downtime two-service rollout.

## 14. Separate a successful trigger from a healthy release

A reliable runbook checks at least these levels:

```text
A. GitHub Actions
  [ ] Frontend lint/build passed
  [ ] Backend lint/tests passed
  [ ] deploy job ran for main push (not PR)

B. Render deployment status
  [ ] Backend deploy reports live/healthy and expected revision
  [ ] Frontend deploy reports live and expected revision
  [ ] No crash loops or persistent build failures

C. External smoke tests
  [ ] Backend GET /healthz returns 2xx (if implemented)
  [ ] GET /api/v1/posts returns a valid JSON array or expected response
  [ ] Frontend loads and requests the correct HTTPS API origin
  [ ] Create/edit/delete succeeds in a safe test environment
  [ ] No browser CORS, mixed-content or configuration errors
```

After deployment, an authorized operator can test the public URLs using **PowerShell** with placeholder domains:

```powershell
$api = 'https://<YOUR-BACKEND-SERVICE>.onrender.com'
$front = 'https://<YOUR-FRONTEND-SERVICE>.onrender.com'
Invoke-WebRequest "$api/api/v1/posts" -UseBasicParsing
Invoke-WebRequest $front -UseBasicParsing
# Only after adding /healthz:
Invoke-RestMethod "$api/healthz"
```

A status of `200` alone does not prove the **new** version is deployed; when this matters, compare an expected commit/version endpoint and the Render deploy history. Avoid testing destructive CRUD actions on valuable production records. Use staging or disposable sample data and proper authorisation.

## 15. Recover, roll back and keep an audit trail

When a new release fails:

1. Stop subsequent deployments and inspect GitHub Actions + Render logs for **which** service failed.
2. Compare `DATABASE_URL`, network access, API prefix and frontend origin configuration without printing secrets.
3. Use Render's deploy history and **Manual Deploy → Deploy a specific commit** to restore a known-good version when appropriate. Review auto-deploy behaviour: an automatic latest-commit release can overwrite a rollback.
4. If the failure is a database schema or data migration problem, reverting application code might not reverse it. Have backups and a migration rollback plan.
5. Re-run external smoke tests, verify the production revision, document the incident and only then restore normal release automation.

**Operational advice:** keep source commit SHA, CI run URL, Render backend/frontend deploy IDs, deploy timestamps, health-check results and the person who approved any manual release in a release log. Do not store hook URLs or database passwords in that log.

## 16. Independent laboratory: simulate release gates safely

The ZIP includes a purpose-built **Node.js** lab (not the actual React/Mongoose application), with **no external npm dependencies**:

```text
lab/
├── backend/
│   ├── server.mjs                 # /healthz, /readyz, /api/v1/posts
│   └── server.test.mjs            # five real HTTP tests
├── frontend/
│   ├── index.html                 # tiny static page; NOT React
│   └── build.mjs                  # inject PUBLIC_API_BASE
├── scripts/
│   ├── release-checks.mjs         # HTTPS URL and release-gate helpers
│   └── release-checks.test.mjs    # nine tests
├── workflow-examples/
│   └── ci-cd.yml                  # example, not automatically active
└── README.md
```

Inside `lab/`, run in Windows PowerShell:

```powershell
node --version
node --test backend/server.test.mjs scripts/release-checks.test.mjs
$env:PUBLIC_API_BASE = 'http://localhost:3001/api/v1'
node frontend/build.mjs
node backend/server.mjs
# Use another terminal:
Invoke-RestMethod http://localhost:3001/healthz
Invoke-RestMethod http://localhost:3001/api/v1/posts
```

The 14 tests check healthy/unhealthy readiness, sample API responses, HTTPS-only public URLs, safe parsing of the frontend API URL and the release conditions `push` + `main` + successful frontend and backend CI. The demo frontend uses **plain static HTML**; it demonstrates build-time public configuration without pretending to run a React or Vite production build.

The file `workflow-examples/ci-cd.yml` is kept **outside** `.github/workflows/` intentionally: copying it into a working repository would activate GitHub Actions. It contains secret **references only**, no real secrets. Do not run it with real production hook values until the environment is configured and reviewed. The demo's `/readyz` toggles a simulated readiness value; it does not connect to MongoDB.

## 17. Common problems and how to diagnose them

| Symptom | Likely cause | First useful check |
| --- | --- | --- |
| Backend fails to start | Database URI missing / Atlas network blocked | Backend service logs, Render env, Atlas Network Access |
| Backend deploy says live but API fails | Wrong route path or runtime failure | Test `/api/v1/posts` and inspect logs |
| Frontend loads but shows fetch error | Incorrect `VITE_BACKEND_URL`, CORS or TLS | Browser Network tab and build-time variable |
| Browser mixed-content warning | Frontend HTTPS but API URL HTTP | Use HTTPS backend URL at build time |
| Render runs twice for one push | Both On Commit auto-deploy and hook deploy enabled | Set a single primary trigger strategy |
| Deploy job green but new release broken | Hook acceptance was mistaken for finished deploy | Inspect Render deploy history + smoke tests |
| `curl` reports success despite HTTP 4xx | No `--fail` flag in existing command | Add `--fail-with-body` and alerting |
| CI fails on `prop-types` | Frontend dependency missing | Install the declared runtime dependency locally, commit reviewed fix |
| Browser crashes at DeletePost | `DeletePostButton` reference in source | Fix the component-name mismatch, lint and rebuild |
| 404 on page refresh after adding router | Missing SPA rewrite | Add Render `/*` → `/index.html` Rewrite |
| CORS error only in production | Wrong frontend origin / broad or missing CORS policy | Compare exact Origin and API response headers |
| Connection fails only from Render | Atlas outbound IP access or credentials | Compare Render outbound IP configuration and Atlas allowlist |
| Render re-deploys failed release after rollback | Latest-commit auto-deploy was re-enabled | Review Auto-Deploy setting and commit history |
| Secret value appears in logs | Hook URL printed or command traced | Rotate hook immediately, remove traces and restrict log access |

## 18. Security, cost and production readiness

- Prefer **HTTPS** on all public endpoints and never put credentials in `VITE_` variables.
- Keep deploy hook URLs only in restricted secret stores; rotate if exposed.
- Enable protected branches and review requirements for your risk level. Keep GitHub token permissions minimal (`contents: read` is sufficient for the example workflow).
- Restrict CORS and authorise all write endpoints; the source API is a learning app, **not a verified production-hardened API**.
- Use least-privilege Atlas users, backups and suitable network restrictions.
- Pin/test runtime versions and action dependencies; review third-party Actions permissions. A concrete action version or commit SHA should be selected for the project's maintenance policy.
- Budget for hosting/database/network capacity; free services may suspend on inactivity and may not provide production SLAs. Confirm current Render and Atlas offerings before committing.
- Do not keep production credentials in screenshots, PDFs, commits or troubleshooting tickets.
- Use staging, realistic smoke tests, error monitoring and a documented recovery procedure before treating deployment as production-ready.

## 19. Quick reference

| Task | Command or location |
| --- | --- |
| Check frontend build | `npm.cmd ci; npm.cmd run build` from repository root |
| Run backend tests | `cd backend; npm.cmd ci; npm.cmd test` |
| Find workflow | `.github/workflows/ci-cd.yaml` |
| Source backend directory | `backend/` |
| Source frontend directory | Repository root |
| Backend Render type | Web Service — Node |
| Frontend Render type | Static Site — Vite build to `dist` |
| Runtime DB variable | `DATABASE_URL` — backend only |
| Public frontend API variable | `VITE_BACKEND_URL` — includes `/api/v1` |
| Backend hook secret | `RENDER_BACKEND_DEPLOY_HOOK` |
| Frontend hook secret | `RENDER_FRONTEND_DEPLOY_HOOK` |
| Required CI dependency | `needs: [frontend, backend]` |
| Render trigger strategy | Either **After CI Checks Pass**, **or** hooks with Auto-Deploy Off |
| Health endpoint | `/healthz` — recommended addition; not in original source |
| Release evidence | CI + deploy status + health and smoke tests |

## 20. Reuse checklist — copy for the next project

- [ ] Confirm actual frontend/backend paths and start/build commands.
- [ ] Fix existing lint, missing dependencies and failing tests **before** deployment.
- [ ] Inspect the entire tracked file tree for `.env` and other secrets; rotate compromised secrets.
- [ ] Provision Atlas with least-privilege credentials, network access and backups.
- [ ] Create backend web service with root `backend`, `npm ci`, `npm start` and private `DATABASE_URL`.
- [ ] Create frontend static site at root, build into `dist`, with public `VITE_BACKEND_URL`.
- [ ] Configure CORS to match the exact frontend origin; add readiness route if required.
- [ ] Select **one** automated deployment strategy, with an explicit CI gate.
- [ ] Protect GitHub secrets; deploy only on successful push to `main`, not pull requests.
- [ ] Avoid logging credentials or deploy URLs; regenerate compromised hooks.
- [ ] Verify the actual deployed revision and perform smoke tests in a safe environment.
- [ ] Confirm rollback, release record and incident response procedure.
- [ ] Revisit service settings, prices, platform behaviour and API docs before future releases.

## 21. Official references, source notes and what comes next

**Render documentation** (consulted for this guide, October 2026):

- [Render — Your First Deploy](https://render.com/docs/your-first-deploy) — web services, static sites, root directory and settings.
- [Render — Deploy Hooks](https://render.com/docs/deploy-hooks) — secret URLs, HTTP response codes and GitHub Actions integration.
- [Render — Deploying on Render](https://render.com/docs/deploys) — auto-deploy modes, CI checks, manual releases and rollback options.
- [Render — Deploy Node Express](https://render.com/docs/deploy-node-express-app) — Node/Express web service.
- [Render — Static Sites](https://render.com/docs/static-sites) and [Redirects/Rewrites](https://render.com/docs/redirects-rewrites).
- [Render — Health Checks](https://render.com/docs/health-checks) — app-level readiness and response criteria.
- [Render — Connecting to MongoDB Atlas](https://render.com/docs/connect-to-mongodb-atlas).
- [Render — Environment Variables](https://render.com/docs/configure-environment-variables).
- [GitHub Docs — Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions).
- [GitHub Docs — Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax).

**Source:** [XMAN-MHD/from-zero-to-fullstack-ch5-devops](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops), specifically `.github/workflows/ci-cd.yaml`, `compose.yaml`, root `package.json`, `backend/package.json`, `backend/src/index.js`, `backend/src/db/init.js`, `backend/src/app.js` and `src/api/posts.js`. This guide is a documentation deliverable; it does not provision, change or deploy a production service.

**Next chapter?** Cheatsheet **22/22** completes the **original five-chapter documentation roadmap**: Foundations, Backend Fundamentals, Backend Development, Frontend Development and DevOps. Further learning can extend into authentication, Next.js, production observability or AI integration, but those topics are **outside this 22-guide series** unless a new curriculum is defined.

---

*Learn → Practice → Document → Reuse.*