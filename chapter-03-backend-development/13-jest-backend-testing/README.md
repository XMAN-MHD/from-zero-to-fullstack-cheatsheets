# Cheatsheet 13 — Automated Backend Testing with Jest

**From Zero to Fullstack · Chapter 3: Backend Development**  
**Audience:** Beginners learning backend testing, and developers reusing a clean testing workflow  
**Primary environment:** Windows 10/11 · PowerShell · VS Code · Node.js  
**Version:** 1.0 · 17 October 2026  
**Source:** [Chapter 3 — Backend Development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development)  
**Method:** **Learn → Practice → Document → Reuse**

> **Evidence note.** Sections marked **SOURCE PROJECT** describe files actually present in Chapter 3. Sections marked **IMPROVEMENT** or **INDEPENDENT LAB** introduce optional examples; they are not claims about code already committed to the source repository. The independent Node.js smoke tests passed in the authoring environment, but the source project's Jest + MongoDB integration suite was not executed here.

## 1. Learning objectives

By the end of this guide, you can:

1. Explain unit, integration and HTTP/API testing without mixing them up.
2. Install Node.js and run Jest in a JavaScript ESM backend on Windows.
3. Read the **real** Jest configuration and 16 test cases in Chapter 3.
4. Understand how `mongodb-memory-server` creates a disposable MongoDB instance.
5. Write assertions and setup/cleanup hooks (`test`, `expect`, `beforeEach`, `beforeAll`, `afterAll`).
6. Reproduce the existing tests against a temporary database and diagnose failures.
7. Add a separate **unit-only test suite** that does not start MongoDB.
8. Recognize fragile tests and improve them without hiding real source behavior.
9. Carry the test patterns into a future Express/Mongoose project.

### What is being tested?

| Scope | Collaborators | Example | In source Chapter 3? |
|:--|:--|:--|:--|
| Unit | One small function; no network or database | Sort option validation | **No**; provided in independent lab |
| Integration | Service + Mongoose model + temporary MongoDB | `createPost()` then `Post.findById()` | **Yes**, in `posts.test.js` |
| HTTP/API | Real HTTP request through Express + persistence | `POST /api/v1/posts` returns `201` | **Not covered** by the current source tests |
| End-to-end | Whole deployed application | Browser creates and reads a post | Not covered here |

**Key distinction:** The existing source tests are **service/database integration tests**. Merely using Jest does not make a test a unit test; a test's scope depends on the components it runs.

## 2. Prerequisites and installation from zero (Windows)

Install or verify the tools:

- **Git:** https://git-scm.com/downloads
- **Node.js (supported LTS release):** https://nodejs.org/en/download
- **Visual Studio Code:** https://code.visualstudio.com/
- **PowerShell / Windows Terminal:** installed or available from Microsoft.

In **PowerShell**, check:

```powershell
node --version
npm.cmd --version
git --version
```

If PowerShell blocks `npm.ps1`, use **`npm.cmd`** or open the VS Code **Command Prompt** terminal. Do not disable execution-policy protections globally just to run npm.

Clone the **actual** Chapter 3 repository:

```powershell
mkdir $HOME\projects -ErrorAction SilentlyContinue
cd $HOME\projects
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development.git
cd from-zero-to-fullstack-ch3-backend-development
npm.cmd ci
```

**Important path correction:** `package.json`, `jest.config.json` and `src/` are at the **repository root**. Although the source README pictures a `backend/` directory, the committed project does **not** use that nested layout. Run `npm.cmd ci` and `npm.cmd test` in the root directory.

`npm ci` uses the existing lockfile and is the preferred reproducible install. It removes/recreates `node_modules` and requires `package-lock.json`; use `npm install` only if the lockfile is missing or intentionally updated.

Confirm required packages:

```powershell
npm.cmd ls jest mongodb-memory-server mongoose cross-env --depth=0
npm.cmd run
```

**SOURCE PROJECT — declared package versions:** Jest `^29.7.0`, `mongodb-memory-server` `^9.1.1`, Mongoose `^8.0.2`, `cross-env` `^10.1.0`. Actual installed versions depend on `package-lock.json`; a caret range is not an exact installed version.

## 3. The real Chapter 3 test architecture

**SOURCE PROJECT:**

```text
from-zero-to-fullstack-ch3-backend-development/
├── package.json                         # npm test script
├── jest.config.json                     # global lifecycle configuration
├── src/
│   ├── __tests__/
│   │   └── posts.test.js                # 16 integration tests
│   ├── test/
│   │   ├── globalSetup.js              # start MongoDB Memory Server
│   │   ├── globalTeardown.js           # stop temporary MongoDB
│   │   └── setupFileAfterEnv.js        # Mongoose connect / disconnect
│   ├── services/posts.js               # functions under test
│   ├── db/
│   │   ├── init.js                     # mongoose.connect(DATABASE_URL)
│   │   └── models/post.js              # schema and model
│   ├── routes/posts.js                 # API HTTP routes (not tested here)
│   ├── app.js
│   └── index.js
└── .gitignore                          # excludes .env and node_modules
```

### Lifecycle: one suite, fresh documents per test

```text
npm test
   |
   v
Jest: globalSetup.js
   |-- starts temporary mongod (version pinned to 6.0.4)
   |-- sets process.env.DATABASE_URL to temporary connection URI
   v
Jest: setupFileAfterEnv.js
   |-- beforeAll: initDatabase() -> Mongoose connects
   |
   v
posts.test.js (16 tests)
   |-- beforeEach: delete posts + create 4 sample posts
   |-- test: call service function and assert database state
   |-- repeat beforeEach for every test
   v
setupFileAfterEnv.js: afterAll -> mongoose.disconnect()
   v
globalTeardown.js -> stop temporary mongod
```

**Isolation detail:** `beforeEach` runs before **every** test, including the creation tests. It deletes records and inserts four sample posts each time. This is convenient for predictable data, but adds repeated database work. Setup hooks in the suite execute after the one-time Mongoose connection is established.

## 4. Configure Jest with ESM

**SOURCE PROJECT — `package.json` excerpts:**

```json
{
  "type": "module",
  "scripts": {
    "test": "cross-env NODE_OPTIONS=--experimental-vm-modules jest",
    "dev": "nodemon src/index.js"
  }
}
```

**SOURCE PROJECT — `jest.config.json`:**

```json
{
  "testEnvironment": "node",
  "globalSetup": "<rootDir>/src/test/globalSetup.js",
  "globalTeardown": "<rootDir>/src/test/globalTeardown.js",
  "setupFilesAfterEnv": ["<rootDir>/src/test/setupFileAfterEnv.js"]
}
```

Why these settings matter:

| Setting | Meaning |
|:--|:--|
| `"type": "module"` | Node treats `.js` files as ES modules. |
| `NODE_OPTIONS=--experimental-vm-modules` | Enables Jest's experimental ESM execution path. |
| `cross-env` | Makes environment variable syntax work on Windows and other systems. |
| `testEnvironment: "node"` | Use backend Node environment, not a browser-like DOM. |
| `globalSetup` | Runs once before the test suites to prepare shared infrastructure. |
| `setupFilesAfterEnv` | Registers Jest hooks in each test environment. |
| `globalTeardown` | Runs once afterwards to shut infrastructure down. |

**Optional improvement:** Jest's ESM guide also recommends disabling transforms (`"transform": {}`) when no code transformation is required. The committed source configuration does **not** contain that property. Test compatibility before changing it.

**Reference:** [Jest 29.7 — ECMAScript Modules](https://jestjs.io/docs/29.7/ecmascript-modules) and [Jest — Configuration](https://jestjs.io/docs/29.7/configuration).

## 5. Temporary MongoDB: the source lifecycle

**SOURCE PROJECT — `src/test/globalSetup.js`:**

```javascript
import { MongoMemoryServer } from 'mongodb-memory-server'

export default async function globalSetup() {
  const instance = await MongoMemoryServer.create({
    binary: { version: '6.0.4' },
  })
  global.__MONGOINSTANCE = instance
  process.env.DATABASE_URL = instance.getUri()
}
```

This package launches a **real temporary MongoDB server process**. Despite the name, it is not a JavaScript mock of MongoDB. The first launch may download a MongoDB binary; network access and an OS-compatible binary are needed. It does not need your usual MongoDB Windows service or an Atlas account.

**SOURCE PROJECT — `src/test/setupFileAfterEnv.js`:**

```javascript
import mongoose from 'mongoose'
import { beforeAll, afterAll } from '@jest/globals'
import { initDatabase } from '../db/init.js'

beforeAll(async () => {
  await initDatabase()
})

afterAll(async () => {
  await mongoose.disconnect()
})
```

`initDatabase()` reads `process.env.DATABASE_URL` or connects locally on port `27018` (database `blog`). In the test run, global setup assigns an **ephemeral URI** first, so the test connection should not use the normal `27018` fallback.

**SOURCE PROJECT — `src/test/globalTeardown.js`:**

```javascript
export default async function globalTeardown() {
  if (global.__MONGOINSTANCE) {
    await global.__MONGOINSTANCE.stop()
  }
}
```

Jest allows the global object created in `globalSetup` to be read by `globalTeardown`; test files themselves cannot rely on that same global object. Environment variables created in global setup are used for the test run.

**Security:** This is an isolated test database, but never put real production connection strings or client records into automated tests. `src/db/init.js` prints its MongoDB URI; do **not** print full database URIs when credentials might be present. Prefer a generic connection success message.

**Version caution:** MongoDB binary `6.0.4` is explicitly pinned by Chapter 3. Some newer OS/platform combinations may reject or fail to run an old binary. If that happens, review a compatible MongoDB version and the installed `mongodb-memory-server` documentation; this is a deliberate change to the source setup, not a step already completed in the repository.

## 6. Read the source tests: arrange, act, assert

A reliable test usually has three parts:

1. **Arrange:** prepare database documents or input objects.
2. **Act:** call one operation under test.
3. **Assert:** verify the result **and**, for writes, the saved state.

**SOURCE PROJECT — creation test pattern (abridged):**

```javascript
const post = {
  title: 'Hello Mongoose!',
  author: 'Daniel Bugl',
  contents: 'This post is stored using Mongoose.',
  tags: ['mongoose', 'mongodb'],
}
const createdPost = await createPost(post)
expect(createdPost._id).toBeInstanceOf(mongoose.Types.ObjectId)
const found = await Post.findById(createdPost._id)
expect(found).toEqual(expect.objectContaining(post))
```

This is **integration testing**, because `createPost()` saves through Mongoose into the temporary MongoDB server, and `Post.findById()` reads the persisted document back.

### The 16 tests that exist today

| Group | Tests | What is checked |
|:--|--:|:--|
| Creating posts | 3 | Full payload, title validation, minimal payload |
| Listing posts | 5 | All, default sort, custom sort, author filter, tag filter |
| Getting one post | 2 | Existing ID and nonexistent ID |
| Updating posts | 4 | Requested field, preservation of other fields, timestamp, missing ID |
| Deleting posts | 2 | Existing record removed, missing ID |
| **Total** | **16** | Service and database behaviors, not HTTP routes |

**Test data:** four sample posts are inserted by `beforeEach`, including posts tagged `react`, `redux` and `nodejs`. This dataset is the basis for the filtering and sorting assertions.

### Common Jest matchers

| Matcher | Use it to assert |
|:--|:--|
| `toBe(value)` | Strict identity for a primitive or exact same reference |
| `toEqual(value)` | Recursive equality of arrays / objects |
| `toStrictEqual(value)` | Stricter structure/prototype-sensitive equality |
| `toBeInstanceOf(Constructor)` | Object instance type |
| `toHaveLength(n)` | Number of array/string elements |
| `toContain(value)` | Value is present in an array or string |
| `toThrow(ErrorType)` | A **synchronous** function throws |
| `rejects.toThrow(...)` | A Promise rejects with an error |
| `resolves.toEqual(...)` | A Promise fulfills with expected value |

**Reference:** https://jestjs.io/docs/29.7/expect

## 7. Run and interpret the source project tests

From the **Chapter 3 repository root**, after `npm.cmd ci`:

```powershell
# Run everything; Jest starts/stops temporary MongoDB.
npm.cmd test

# Easier to diagnose ordering/resources with one worker.
npm.cmd test -- --runInBand

# Run just the service/database test file.
npm.cmd test -- src/__tests__/posts.test.js --runInBand

# List matched test files (useful when tests aren't discovered).
npm.cmd test -- --listTests
```

**Do not run `npm run dev` just to run Jest.** Integration tests start their **own** temporary MongoDB and call services directly. A local production-like MongoDB service on port `27018` is not required for the test suite when global setup succeeds.

Expected when all assertions and dependencies work:

```text
PASS  src/__tests__/posts.test.js
Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
```

This is an **expected example**, not an execution result from this document's authoring environment. Exact timings, warnings and status depend on your installed packages and machine.

A failed Jest run is useful information. Read the **first actual error**, not merely the final `Tests: ... failed` line. If the MongoDB binary cannot start, none of the service assertions can run successfully.

## 8. Improve fragile tests (proposed changes)

These examples are **IMPROVEMENTS**, not edits already merged in Chapter 3.

### 8.1 Prevent a false-positive validation test

In the source's **"without title should fail"** test, the pattern is:

```javascript
try {
  await createPost(invalidPost)
} catch (err) {
  expect(err).toBeInstanceOf(mongoose.Error.ValidationError)
}
```

If `createPost()` unexpectedly **does not throw**, this test contains no failing assertion and may still pass. A stronger test is:

```javascript
test('without title rejects with a validation error', async () => {
  const invalidPost = { author: 'A tester', tags: ['invalid'] }
  await expect(createPost(invalidPost)).rejects.toBeInstanceOf(
    mongoose.Error.ValidationError,
  )
})
```

Now a fulfilled Promise fails the test, instead of silently bypassing the `catch`.

### 8.2 Avoid flaky timestamp precision checks

The source checks that `updatedAt` after an update is **strictly greater** than the old time. On a fast machine, both dates could be recorded with the same millisecond. An optional stabilization is to set a known older timestamp on an existing record first, then update it, or to assert a meaningful nondecreasing time plus the intended data change. Choose the assertion based on what the feature actually promises; do not add arbitrary sleeps to every test.

### 8.3 Clean up after a setup failure

A database download or startup failure can happen **before** the test assertions. Keep `globalTeardown()` defensive and close connections in `afterAll`. A CI job should never reuse the production database URL for its tests.

### 8.4 Test HTTP behavior separately

The existing `posts.test.js` imports **services**, not `app` or `postsRoutes`. Therefore, it does **not** verify HTTP status codes `200`, `201`, `204`, `400`, `404`, or `500`. Future HTTP integration tests can use an HTTP test client such as Supertest to assert Express route behavior. This library has not added those tests to the source repository.

## 9. Independent lab — a unit test suite without MongoDB

**INDEPENDENT LAB:** Included in this cheatsheet ZIP as `unit-lab/`. Use it when you want to learn Jest without a database or change to your Blog API. It validates sort query options in a small, pure function that follows a safer, allow-listed approach.

```text
unit-lab/
├── package.json
├── jest.config.json
├── src/
│   ├── utils/sort-options.js
│   └── __tests__/sort-options.test.js
└── tests/node-smoke.test.js
```

### 9.1 Create or open the laboratory

Unzip the supplied bundle, then in VS Code PowerShell:

```powershell
cd .\unit-lab
npm.cmd install
npm.cmd test
```

`npm install` is correct for this **new lab**, because it intentionally ships without a generated lockfile. After installation, keep the resulting `package-lock.json` for later reproducible `npm ci` runs.

If the npm registry is inaccessible, the included **Node.js smoke test** does not require Jest dependencies:

```powershell
node --test tests/node-smoke.test.js
```

### 9.2 The pure function to test

`src/utils/sort-options.js`:

```javascript
const allowedFields = new Set(['createdAt', 'updatedAt', 'title'])

export function normalizeSortOptions(input = {}) {
  const { sortBy = 'createdAt', sortOrder = 'descending' } = input
  if (!allowedFields.has(sortBy)) {
    throw new RangeError('Unsupported sort field')
  }
  if (!['ascending', 'descending'].includes(sortOrder)) {
    throw new RangeError('Unsupported sort order')
  }
  return { sortBy, sortOrder, direction: sortOrder === 'ascending' ? 1 : -1 }
}
```

This utility has no database, network, timers or user-specific global state. Each input has an immediate output, ideal for unit tests.

### 9.3 A Jest unit test

`src/__tests__/sort-options.test.js` (extract):

```javascript
import { describe, expect, test } from '@jest/globals'
import { normalizeSortOptions } from '../utils/sort-options.js'

describe('normalizeSortOptions', () => {
  test('uses safe defaults', () => {
    expect(normalizeSortOptions()).toEqual({
      sortBy: 'createdAt', sortOrder: 'descending', direction: -1,
    })
  })

  test.each([['ascending', 1], ['descending', -1]])(
    '%s maps to %i', (sortOrder, direction) => {
      expect(normalizeSortOptions({ sortOrder }).direction).toBe(direction)
    },
  )

  test('rejects unsupported fields', () => {
    expect(() => normalizeSortOptions({ sortBy: '$where' })).toThrow(RangeError)
  })
})
```

The **full example** has six Jest cases (a parameterized test contributes two) and three separate Node smoke tests. The Node smoke tests **were executed and passed** during preparation of this guide. The Jest cases require `npm install` and **were not run here**, so do not confuse the two checks.

### 9.4 Why use two Jest configurations?

Our main source `jest.config.json` always includes the MongoDB Memory Server lifecycle. Running a pure unit test within that configuration would **still start MongoDB**, even if the utility does not use a database. The lab therefore has an isolated `jest.config.json` with no `globalSetup` or `globalTeardown`:

```json
{
  "testEnvironment": "node",
  "transform": {},
  "testMatch": ["**/src/__tests__/**/*.test.js"]
}
```

Its `package.json` runs Jest with `cross-env NODE_OPTIONS=--experimental-vm-modules jest --runInBand`. Use this template for future **unit-only test commands** if you keep both test types inside one larger repository.

## 10. Test design principles

### Arrange–Act–Assert

```text
Arrange: seed or construct necessary input
Act:     call one function / API behavior
Assert:  compare result AND important side effects
```

### Make test isolation deliberate

- Tests should not depend on execution order.
- Database writes should be reset **between** tests when a shared instance is used.
- Avoid using a production or personally identifiable dataset.
- Favor deterministic fixtures over random data without stable seeds.
- Close MongoDB and Node handles to prevent hanging CI jobs.

### Name tests as behavior

Prefer `"rejects a post without title"` instead of `"test 3"`. Use failed tests to decide if the **implementation** or the **assertion** is wrong; never weaken an assertion merely to make CI green.

### What to mock?

| Concern | Best starting choice |
|:--|:--|
| Pure formatter/validator | Test real function with no mocks |
| Mongoose CRUD behavior | Use temporary MongoDB integration test |
| Third-party email/payment provider | Mock external integration at its boundary |
| Express HTTP route status/JSON | Test actual app via HTTP test client |
| Time-sensitive logic | Inject/freeze time when required |

## 11. Debugging and troubleshooting

| Symptom | Likely cause | First corrective step |
|:--|:--|:--|
| `npm.ps1 cannot be loaded` | PowerShell execution-policy block | Run `npm.cmd` instead |
| `jest is not recognized` | Dependencies missing or wrong directory | Verify root; run `npm.cmd ci` |
| `Cannot use import statement outside a module` | Jest ESM configuration missing | Check `"type": "module"`, test script and transforms |
| `SyntaxError: Unexpected token export` | Test runner treating ESM as CommonJS | Recheck `--experimental-vm-modules` |
| MongoDB binary download failed | Offline machine, proxy or blocked downloads | Check network/cache and binary version; retry |
| MongoDB binary not supported | Pinned `6.0.4` incompatible with OS | Use a supported binary version after review |
| `ECONNREFUSED` / server selection timeout | MongoDB instance did not start or URI wrong | Inspect global setup logs and `DATABASE_URL` |
| Timeout / open handle after tests | Missing `disconnect()` / `stop()` | Review `afterAll` and global teardown |
| `Test Suites: 0 total` | Test glob or wrong folder | Use `npm.cmd test -- --listTests` |
| Unit test starts MongoDB | Reusing main Jest configuration | Run unit-only config without DB lifecycle |
| Test passes after missing throw | `try/catch` lacks explicit failure assertion | Use `await expect(promise).rejects` |
| Timestamp test flakes | Equal millisecond timestamps | Stabilize test data/clock; avoid unnecessary sleeps |

**Never solve a failing test by pointing at the production database.** Investigate the setup itself, fix assertions or fixtures, and document any version change.

## 12. Quick reference — command sheet

| Goal | Windows PowerShell |
|:--|:--|
| Clone Chapter 3 | `git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development.git` |
| Install source dependencies | `npm.cmd ci` |
| Show npm scripts | `npm.cmd run` |
| Run source tests | `npm.cmd test` |
| Run serially | `npm.cmd test -- --runInBand` |
| Run posts suite | `npm.cmd test -- src/__tests__/posts.test.js --runInBand` |
| List tests | `npm.cmd test -- --listTests` |
| Run source lint | `npm.cmd run lint` |
| Install standalone lab | `cd unit-lab; npm.cmd install` |
| Run standalone Jest unit tests | `npm.cmd test` (inside `unit-lab`) |
| Run zero-dependency lab smoke tests | `node --test tests/node-smoke.test.js` |

**Concept glossary:** *test case* = single assertion scenario; *fixture* = sample data; *setup* = preparation; *teardown* = cleanup; *assertion* = checked expectation; *mock* = substitute dependency; *integration test* = interaction between real components.

## 13. Repeatable checklist for your next backend

- [ ] Select current Node.js LTS and verify `node`/`npm.cmd` on Windows.
- [ ] Keep `type: module` / test runner configuration consistent with code imports.
- [ ] Separate **unit**, **database integration** and **HTTP/API** testing goals.
- [ ] Use a **temporary test database**; never production MongoDB.
- [ ] Put real service/model integration tests near the code (`src/__tests__/`).
- [ ] Seed predictable fixtures and reset state before independent test cases.
- [ ] Prefer `expect(promise).rejects` for async failure expectations.
- [ ] Test both returned values and persisted document state for writes.
- [ ] Cover missing IDs, invalid input, sorting and filtering.
- [ ] Use dedicated configuration for pure unit tests without MongoDB setup.
- [ ] Run tests serially initially; only parallelize after understanding isolation.
- [ ] Track and review pinned MongoDB binary versions across operating systems.
- [ ] Ensure Mongoose disconnects and the temporary MongoDB process stops.
- [ ] Run lint + tests before pushing and integrate them with CI later.
- [ ] Never expose secrets, private data or real connection URIs in test logs.

## 14. Source map, verification and references

### Verified in the actual Chapter 3 GitHub repository

- [`package.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/package.json) — Jest script and dependencies.
- [`jest.config.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/jest.config.json) — setup / teardown configuration.
- [`src/test/globalSetup.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/test/globalSetup.js) — ephemeral MongoDB startup, pinned binary `6.0.4`.
- [`src/test/globalTeardown.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/test/globalTeardown.js) — stop MongoDB.
- [`src/test/setupFileAfterEnv.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/test/setupFileAfterEnv.js) — connect/disconnect Mongoose.
- [`src/__tests__/posts.test.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/__tests__/posts.test.js) — 16 source tests.
- [`src/services/posts.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/services/posts.js) — real functions tested.
- [`src/db/models/post.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/db/models/post.js) — Mongoose schema.
- [`src/db/init.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development/blob/main/src/db/init.js) — connection helper.

### Official learning resources

- [Jest 29.7 — Getting Started](https://jestjs.io/docs/29.7/getting-started)
- [Jest 29.7 — ECMAScript Modules](https://jestjs.io/docs/29.7/ecmascript-modules)
- [Jest 29.7 — Setup and Teardown](https://jestjs.io/docs/29.7/setup-teardown)
- [Jest 29.7 — Configuration](https://jestjs.io/docs/29.7/configuration)
- [Jest 29.7 — Expect](https://jestjs.io/docs/29.7/expect)
- [MongoDB Memory Server — Integration with test runners](https://typegoose.github.io/mongodb-memory-server/docs/guides/integration-examples/test-runners/)
- [Mongoose — Validation](https://mongoosejs.com/docs/validation.html)

### Authoring verification status (17 October 2026)

| Check | Result |
|:--|:--|
| Real source files examined on GitHub | **Yes** |
| Existing source test cases counted | **16** |
| Independent lab Node.js smoke tests | **3 passed** |
| Lab JavaScript static syntax | **Checked** |
| Independent Jest lab suite | **Not executed** (Jest not installed here) |
| Source Jest/MongoDB integration suite | **Not executed** (temporary mongod not available here) |
| PDF rendered and inspected | **Yes** |

**Next:** Chapter 4 — Frontend Development. Cheatsheet 14: *React Components, JSX, Props & State*.
