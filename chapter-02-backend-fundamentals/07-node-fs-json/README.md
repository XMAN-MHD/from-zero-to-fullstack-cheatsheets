# Cheatsheet 07 — Node.js File System & JSON Handling

**From Zero to Fullstack · Chapter 2: Backend Fundamentals**  
**Audience:** Absolute beginners, backend learners, and developers looking for reusable file/JSON patterns  
**Primary platform:** Windows 10/11 · PowerShell · VS Code  
**Version:** 1.0 · 13 October 2026  
**Real project:** [Chapter 2 — Backend Fundamentals](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Read and write local files safely with Node.js, understand JSON serialization, reproduce the actual Chapter 2 file-backed HTTP example, and build a small reusable JSON file utility.

## Quick reference — read, parse, update, save

```js
// json-store.mjs — standalone reusable example (not Chapter 2 source code)
import { readFile, writeFile } from 'node:fs/promises'

const file = new URL('./users.json', import.meta.url)
const users = JSON.parse(await readFile(file, 'utf8'))
users.push({ name: 'Awa', email: 'awa@example.test' })
await writeFile(file, JSON.stringify(users, null, 2) + '\n', 'utf8')
console.log(`${users.length} users saved`)
```

Create a `users.json` beside `json-store.mjs` containing `[]`, then run `node json-store.mjs` from a terminal. **This exercise changes that file:** make a backup or use a dedicated practice folder, not an important dataset.

| Task | Built-in API / command |
| --- | --- |
| Check Node installation | `node --version` |
| Run JavaScript | `node script.mjs` |
| Read a file (async) | `await readFile(file, 'utf8')` |
| Write a file (async) | `await writeFile(file, text, 'utf8')` |
| Convert JSON text into data | `JSON.parse(text)` |
| Convert data into JSON text | `JSON.stringify(value, null, 2)` |
| Resolve a file beside a module | `new URL('./users.json', import.meta.url)` |
| Inspect JSON in PowerShell | `Get-Content .\users.json -Raw` |
| Test the source HTTP example | `node backend/files.js` then `curl.exe -i http://localhost:3000/` |

**No external npm package is required for `node:fs`, `node:fs/promises`, or JSON.** They are built into Node.js/JavaScript. The original project already uses the Node.js `fs` module.

## 1. Core concepts: files, text, JSON, and JavaScript values

A **file** is stored data on disk. The Node.js **File System API** (`fs`) lets your server read, write, rename, or remove files. **JSON (JavaScript Object Notation)** is a text format for representing objects, arrays, strings, numbers, booleans, and `null`.

A JSON file is **text**—it is not a JavaScript variable. Read the text first, then **parse** it into a JavaScript value. When saving, **stringify** the JavaScript value back to text.

```text
    users.json on disk          JavaScript in memory
  '[{"name":"Adam"}]'        [{ name: 'Adam' }]
             |                        |
             | readFile() + parse()   |
             +----------------------->+
             |                        |
             | writeFile() + stringify()
             +<-----------------------+
```

**Vocabulary:**

- **Read:** obtain file contents from storage.
- **Write:** store new contents; `writeFile()` normally **replaces** an existing file.
- **Buffer:** a collection of bytes that `readFile()` returns if no encoding is requested.
- **UTF-8:** a text encoding; specify `'utf8'` when you need a string.
- **Serialize:** turn a JavaScript value into a text representation with `JSON.stringify()`.
- **Deserialize:** convert JSON text back with `JSON.parse()`.
- **Synchronous:** the JavaScript thread waits until the I/O completes.
- **Asynchronous:** the operation can complete later; `await` waits for its Promise without synchronous file-I/O blocking of the event loop.

**JSON rules:** double-quoted property names and strings, no comments, no trailing commas, and a single top-level JSON value. An array of users is a valid JSON value. `undefined`, functions, and `BigInt` are not directly representable as ordinary JSON values.

## 2. Prerequisites — install tools from scratch

You need a Windows machine, internet access, and permission to install ordinary developer tools. Recommended: 8 GB RAM, a few GB of free space, and familiarity with opening a terminal. If you completed Cheatsheet 01, the tools should already be installed.

### 2.1 Install Node.js LTS (required)

1. Open the [official Node.js downloads page](https://nodejs.org/en/download).
2. Choose a **supported LTS** release for Windows (usually the x64 installer).
3. Run the installer with its ordinary defaults. npm is included by default.
4. Close and reopen PowerShell; verify:

```powershell
node --version
npm.cmd --version
node -e "console.log('Node can execute JS:', 2 + 2)"
```

**Expected:** valid Node/npm version strings and `Node can execute JS: 4`. A current LTS such as Node.js 22 or 24 supports everything in this guide. For an existing project, also check its specific engine requirements.

**Windows tip:** If PowerShell blocks the `npm.ps1` script, use `npm.cmd` rather than changing system-wide execution policy. The exercises below use `node` directly.

### 2.2 Install Visual Studio Code (recommended)

Install from [code.visualstudio.com/download](https://code.visualstudio.com/download), restart your terminal, and run:

```powershell
code --version
```

If `code` is unavailable, launch VS Code from the Start menu and open the desired folder using **File → Open Folder**.

### 2.3 Install Git (only needed to clone the source)

Download from [git-scm.com/install/windows](https://git-scm.com/install/windows), install, reopen PowerShell, and run:

```powershell
git --version
```

You can alternatively choose **Code → Download ZIP** on GitHub and extract it. You do not need MongoDB, Docker, Express, a database account, or extra npm modules for this file system lesson.

## 3. What our actual Chapter 2 source contains

We examined the current default-branch files of [from-zero-to-fullstack-ch2-backend-fundamentals](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals).

```text
from-zero-to-fullstack-ch2-backend-fundamentals/
├── backend/
│   ├── files.js        # Native HTTP server reads a local JSON file
│   ├── users.json      # Example users in JSON format
│   └── mongodbweb.js   # Separate MongoDB exercise (not needed here)
├── package.json        # React/Vite starter; type: module
├── package-lock.json
└── src/                 # Existing frontend starter
```

The entire **active** source file `backend/files.js` is short:

```js
import { createServer } from 'http'
import { readFileSync } from 'fs'

const server = createServer((req, res) => {
  res.statusCode = 200
  res.setHeader('Content-Type', 'application/json')
  res.end(readFileSync('backend/users.json'))
})
const host = 'localhost'
const port = 3000

// The source also contains a commented-out exercise using:
// writeFileSync(), JSON.stringify(), readFileSync(), JSON.parse()

server.listen(port, host, () => {
  console.log(`Server listening on http://${host}:${port}`)
})
```

> **Code fidelity:** The snippet above preserves the active logic but condenses the commented exercises and normalizes the indentation for readability. Open the [original file](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/files.js) to see its complete exact text.

The original `backend/users.json` contains exactly one example user:

```json
[{"name":"Adam Ondra","email":"adam.ondra@climb.ing"}]
```

**Important facts about the source:**

1. `readFileSync('backend/users.json')` returns a **Buffer**, because no encoding is supplied. Passing a Buffer to `res.end()` is valid.
2. The file path is relative to the **current working directory**, not necessarily to `files.js`. Run the script from the repository root.
3. The file is read on **every HTTP request**; edits to `users.json` appear on the next request.
4. The HTTP callback does not inspect `req.method` or `req.url`: it returns the same body for any route/method.
5. Reading failures have **no try/catch** and can crash the request handler/process. The guide's improved examples will handle errors.
6. The source uses sync I/O to teach basics. **Async I/O is recommended** for file operations in a long-running server.
7. The top-level `package.json` has `"type": "module"`, enabling `import` syntax in `.js` files. `npm run dev` starts **Vite**, not this backend server.

No independent `writeFileSync()` program is committed; the write/read/parse/stringify example appears as **comments** in `files.js`. The exercises below make those operations executable without modifying the historical project.

## 4. Reproduce the original file-backed HTTP server

### 4.1 Download the source

Open **PowerShell** in a convenient folder:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals.git
Set-Location from-zero-to-fullstack-ch2-backend-fundamentals
Get-Location
Get-Content .\backend\users.json -Raw
```

**Expected:** the current directory ends with `from-zero-to-fullstack-ch2-backend-fundamentals` and the printed JSON includes the single sample user. If you have cloned the repository already, just `Set-Location` into it; do not clone a second copy accidentally.

### 4.2 Run without installing an HTTP framework

```powershell
node .\backend\files.js
```

**Expected terminal message:** `Server listening on http://localhost:3000`.

Open a **second PowerShell window** and test:

```powershell
curl.exe -i http://localhost:3000/
Invoke-RestMethod http://localhost:3000/
```

**Expected:** HTTP `200`, content type `application/json`, and an array containing the sample user. Press **Ctrl + C** in the server's original terminal when done.

**Do not confuse ports:** this example uses `3000`; the Vite development frontend often uses `5173`. Two programs cannot independently listen on the same host/port combination.

### 4.3 Observe file changes (safe practice)

While the source server is running, back up your local demo file **before changing it**:

```powershell
Copy-Item .\backend\users.json .\backend\users.backup.json
```

Edit `backend/users.json` to add a second **fictional** user while preserving a valid array. Refresh `http://localhost:3000/` to see the change. Restore the original when finished:

```powershell
Copy-Item .\backend\users.backup.json .\backend\users.json -Force
Remove-Item .\backend\users.backup.json
```

**Why it works:** the original server calls `readFileSync()` on each incoming request, not only at startup. If `node backend/files.js` is executed from the wrong folder, you may instead receive an `ENOENT` error.

## 5. Practice lab — file I/O without an HTTP server

The remainder of the guide uses a **separate exercise folder**, so that experiments do not accidentally change the course repository.

### 5.1 Create a new folder and starter JSON

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects\node-json-lab"
Set-Location "$HOME\Projects\node-json-lab"
'[]' | Set-Content -Path .\users.json -Encoding ascii
code .
```

If you do not have `code` on the PATH, open the folder manually in VS Code. The ASCII encoding is intentional for this minimal `[]` fixture: **Windows PowerShell 5.1** can add a UTF-8 byte-order mark (BOM) with `-Encoding utf8`, and the BOM can make `JSON.parse()` fail. Once Node.js starts writing the file using `'utf8'`, Unicode names and accents work normally. For this lab, `[]` is an empty user list.

We use `.mjs` files, which allow ES-module `import` statements **without** adding a `package.json`. That keeps the exercise independent from npm configuration.

### 5.2 Read a file as text

Create `read-users.mjs`:

```js
import { readFile } from 'node:fs/promises'

const file = new URL('./users.json', import.meta.url)
const text = await readFile(file, 'utf8')
console.log('JSON text:', text)
```

Run:

```powershell
node .\read-users.mjs
```

**Expected:** `JSON text: []` (possibly followed by a newline). `readFile()` is asynchronous and returns a Promise; `await` waits until it resolves. The `'utf8'` option makes the result a **string**.

**Important:** without `'utf8'`, you receive a **Buffer**. It contains bytes; convert to text with `.toString('utf8')` when needed.

### 5.3 Parse text into an array

Create `parse-users.mjs`:

```js
import { readFile } from 'node:fs/promises'

const file = new URL('./users.json', import.meta.url)
const text = await readFile(file, 'utf8')
const users = JSON.parse(text)

console.log('Is array?', Array.isArray(users))
console.log('Number of users:', users.length)
```

**Expected:** `Is array? true` and `Number of users: 0`. `JSON.parse()` is **synchronous**; it throws `SyntaxError` if the text is invalid JSON. In reliable programs, handle this failure explicitly.

### 5.4 Serialize and write data

Create `save-users.mjs`:

```js
import { writeFile } from 'node:fs/promises'

const file = new URL('./users.json', import.meta.url)
const users = [
  { name: 'Awa', email: 'awa@example.test' },
  { name: 'Moussa', email: 'moussa@example.test' },
]

const text = JSON.stringify(users, null, 2) + '\n'
await writeFile(file, text, 'utf8')
console.log('Saved', users.length, 'users')
```

Run:

```powershell
node .\save-users.mjs
Get-Content .\users.json -Raw
```

**Expected:** a pretty-printed JSON array with **two fictional users**. The `null, 2` arguments to `JSON.stringify()` mean “no replacer” and “indent by two spaces.” `writeFile()` **overwrites the whole file**, so it must not be pointed at a real dataset casually.

### 5.5 Update the file (read → modify → write)

Create `add-user.mjs`:

```js
import { readFile, writeFile } from 'node:fs/promises'

const file = new URL('./users.json', import.meta.url)
const users = JSON.parse(await readFile(file, 'utf8'))

if (!Array.isArray(users)) {
  throw new TypeError('Expected a JSON array')
}

users.push({ name: 'Fatou', email: 'fatou@example.test' })
await writeFile(file, JSON.stringify(users, null, 2) + '\n', 'utf8')
console.log('Now stored:', users.length)
```

**Expected:** `Now stored: 3` on the first run after the previous step; **running it again adds another Fatou**. This reveals a limitation: a simple append script does not prevent duplicates. An improved version would validate input and check an identifier before adding a record.

## 6. Understand paths and working directories

Two different kinds of relative paths are easy to confuse.

**A. Relative to the terminal's working directory** (as in our Chapter 2 source):

```js
readFileSync('backend/users.json')
```

Here, Node resolves the string relative to `process.cwd()`, the folder from which you started `node`.

**B. Relative to the JavaScript module** (recommended for a companion data file):

```js
const file = new URL('./users.json', import.meta.url)
await readFile(file, 'utf8')
```

Here, Node resolves the URL relative to the `.mjs`/ES-module file itself. It still works if the terminal starts elsewhere.

To debug:

```js
console.log('Working directory:', process.cwd())
console.log('Module URL:', import.meta.url)
```

**Windows gotcha:** PowerShell accepts `\.\backend\users.json` as a path, but JavaScript string literals containing backslashes need correct escaping. Prefer `new URL(...)` or `node:path` (`join()`, `resolve()`) instead of manually building paths with backslash strings.

## 7. Synchronous vs asynchronous file APIs

- **`readFileSync(path)`** — Blocks the JavaScript thread. Appropriate for small one-time CLI or startup reads and introductory lessons.
- **`writeFileSync(path, data)`** — Blocks the JavaScript thread. Suitable for small single-purpose scripts, not high-volume server requests.
- **`await readFile(path, 'utf8')`** — Asynchronous file I/O. Prefer for general application and server code when reading text.
- **`await writeFile(path, data)`** — Asynchronous file I/O. Prefer for general application writes, with errors and concurrency handled appropriately.

The source demonstrates `readFileSync()` in the HTTP callback. That works for a tiny example, but under concurrent load synchronous reads can stall other requests. **Recommended reusable pattern:** import from `node:fs/promises` and use `await`.

### Promises and `try/catch`

```js
import { readFile } from 'node:fs/promises'

try {
  const contents = await readFile(new URL('./users.json', import.meta.url), 'utf8')
  console.log(JSON.parse(contents))
} catch (err) {
  if (err.code === 'ENOENT') {
    console.error('The file does not exist')
  } else if (err instanceof SyntaxError) {
    console.error('The JSON file is not valid')
  } else {
    console.error('Could not load users:', err.message)
  }
  process.exitCode = 1
}
```

This handles the two most important beginner failure cases: **missing file (`ENOENT`)** and **invalid JSON (`SyntaxError`)**. File access may also fail with `EACCES` or `EPERM` depending on operating-system permissions.

## 8. Reusable utility — validated JSON array reader/writer

For a future **small CLI tool or practice project**, you can save a reusable file named `json-utils.mjs`:

```js
import { readFile, writeFile } from 'node:fs/promises'

export async function readJsonArray(file) {
  const text = await readFile(file, 'utf8')
  const data = JSON.parse(text)
  if (!Array.isArray(data)) {
    throw new TypeError('Expected a JSON array')
  }
  return data
}

export async function writeJsonArray(file, records) {
  if (!Array.isArray(records)) {
    throw new TypeError('Expected an array of records')
  }
  const text = JSON.stringify(records, null, 2) + '\n'
  await writeFile(file, text, 'utf8')
}
```

Now create `demo.mjs` in the same folder:

```js
import { readJsonArray, writeJsonArray } from './json-utils.mjs'

const file = new URL('./users.json', import.meta.url)
const users = await readJsonArray(file)
users.push({ name: 'Ali', email: 'ali@example.test' })
await writeJsonArray(file, users)
console.log('Saved and verified', (await readJsonArray(file)).length, 'records')
```

Run `node demo.mjs` in your lab. The utility is **not** claimed to exist in the Chapter 2 repository. Its use is appropriate for learning and small single-process scripts, not for concurrent production writes.

**Limitations:** Each write replaces the entire file. Multiple processes or simultaneous requests can overwrite one another's changes. Large files cost memory; invalid input and partial writes must be handled; actual applications may need a database, file-locking, serialization of writes, or atomic replacement. The next chapter introduces stronger persistence concepts.

## 9. JSON pitfalls — what to avoid

Common mistakes and their remedies:

- **`{name: 'Awa'}` in a `.json` file:** Invalid JSON. Use `{"name":"Awa"}` with double quotes.
- **`{ "name": "Awa", }`:** Invalid trailing comma. Remove the comma.
- **`// comment` inside a JSON file:** JSON does not permit comments. Document the structure separately.
- **`JSON.parse(object)`:** `JSON.parse()` expects JSON text. Read the file as a string first.
- **`JSON.stringify()` followed by `.push()` on the result:** The result is text, not an array. Update the array *before* stringifying.
- **Calling `writeFile()` without reading old data:** Existing content will be replaced. Back up and merge if needed.
- **`JSON.stringify(undefined)`:** Returns `undefined` for a top-level value. Supply a valid JSON-compatible value.
- **Serializing `BigInt`:** Throws by default. Convert the data intentionally before serialization.
- **Serializing `Date`:** Produces an ISO string. Convert it back to a `Date` explicitly if needed.
- **Reading a missing path:** Causes `ENOENT`. Check paths and handle errors.

### Encoding and accents

Always prefer **UTF-8** for text files. JSON can store characters such as `é` and Arabic script. Avoid assumptions about your Windows terminal's display encoding; inspect the saved data with VS Code if console output looks incorrect.

### JSON vs JavaScript object

```js
const object = { name: 'Awa', active: true }
const text = JSON.stringify(object)
const restored = JSON.parse(text)

console.log(typeof text)       // 'string'
console.log(typeof restored)   // 'object'
console.log(restored.name)     // 'Awa'
```

Remember that `typeof null` is `'object'` for historical JavaScript reasons; use `Array.isArray(value)` to test for an array.

## 10. Test your understanding — three exercises

### Exercise A — round-trip JSON

Create an array with three fictional users. Convert it to JSON using `JSON.stringify()`, convert back with `JSON.parse()`, then verify `Array.isArray(result)` and the number of users. **Goal:** understand serialization/deserialization.

### Exercise B — missing and invalid files

Rename `users.json` to `users.hidden.json`; run `node read-users.mjs` and observe `ENOENT`. Restore the name. Next, place invalid JSON such as `[{ name: 'Bad' }]` in `users.json` and run `node parse-users.mjs`; observe `SyntaxError`. Restore valid JSON after testing.

### Exercise C — module path independence

From a different folder, run your lab's script by absolute path:

```powershell
Set-Location $HOME
node "$HOME\Projects\node-json-lab\read-users.mjs"
```

It should still find its companion `users.json` because the script uses `import.meta.url`. Compare this with how the **original** Chapter 2 script depends on `process.cwd()`.

**Testing note:** The standalone lab is separate from any version-controlled course project. It should never contain actual personal data or credentials.

## 11. Troubleshooting on Windows

- **`node` is not recognized:** Install Node.js LTS, restart PowerShell, and run `where.exe node`.
- **`Cannot use import statement outside a module`:** Use `.mjs` or set `"type": "module"` in your package.
- **`ENOENT: no such file or directory`:** Check `Get-Location`, `Get-ChildItem`, and file paths. Prefer module-relative URLs.
- **`Unexpected token` in JSON:** Check JSON syntax (quotes, commas, brackets) and remove any leading UTF-8 BOM.
- **`TypeError: ... push is not a function`:** Use `Array.isArray()` to verify the loaded JSON is an array.
- **`EACCES` / `EPERM`:** Use an editable folder you own and close conflicting programs.
- **File changes are unexpectedly overwritten:** Remember that `writeFile()` replaces existing contents. Back up first.
- **`EADDRINUSE` on port 3000:** Stop the earlier process before running the Chapter 2 server.
- **`npm run dev` starts Vite:** Correct: this repository does not define a backend npm script. Run `node backend/files.js`.
- **JSON prints as a Buffer:** Supply `'utf8'` when reading text or use `.toString('utf8')`.
- **PowerShell `curl` behaves unexpectedly:** Use `curl.exe -i` or `Invoke-RestMethod`.

Diagnostics:

```powershell
Get-Location
Get-ChildItem .\backend
Get-Content .\backend\users.json -Raw
node --version
where.exe node
```

If the issue is in your standalone lab, replace `backend` with the lab path. Never share secrets or private user data while posting logs.

## 12. Security and production considerations

1. **Never allow untrusted input to choose arbitrary filesystem paths.** A route that accepts `../../private-file` can expose sensitive data. Use a known safe directory and validate the input if dynamic paths are necessary.
2. **Keep secrets out of source and JSON fixtures.** Use safe example addresses ending in `.example.test` or `.test`, not real user details. Never publish `.env` credentials.
3. **Use asynchronous I/O in servers.** `readFileSync()` inside a request handler blocks the event loop; the chapter example is instructional rather than production-ready.
4. **Handle errors.** A missing, corrupt, or inaccessible file should not bring down a public service or leak server stack traces.
5. **Validate data shape.** `JSON.parse()` checks syntax, not that the result matches your business rules.
6. **Limit file size when handling untrusted input.** `readFile()` loads an entire file into memory.
7. **Avoid concurrent read-modify-write races.** A JSON file is not a safe substitute for a database for multi-user updates.
8. **Protect data integrity.** For important files, consider temp-file-plus-rename patterns with correct error handling, backups, and an appropriate data store. A rename alone does not solve all concurrency or durability problems.
9. **Be explicit about privacy.** Never use real customers' names, emails, tokens, or passwords in course data committed publicly.
10. **Separate demos and persistent stores.** Use small local fixtures for learning; use MongoDB for the persistent Blog application in later chapters.

## 13. Reuse checklist for your next project

- [ ] Node.js LTS, VS Code, and (optionally) Git are installed and verified.
- [ ] I know how to start a `.mjs` script with `node`.
- [ ] I can distinguish text, Buffer, JavaScript object, and JSON string.
- [ ] I can explain `JSON.parse()` and `JSON.stringify()`.
- [ ] I ran our actual Chapter 2 `backend/files.js` from the repository root.
- [ ] I know that `npm run dev` launches Vite in this source repository.
- [ ] I understand why the source uses a working-directory-relative path.
- [ ] I can use module-relative `new URL(..., import.meta.url)` safely.
- [ ] I have successfully read, modified, and written JSON in a disposable lab.
- [ ] I can handle missing files and invalid JSON.
- [ ] I understand the difference between sync and async file APIs.
- [ ] I validate JSON data shape and avoid overwriting important files.
- [ ] I know why file-based JSON is not ideal for concurrent multi-user writes.
- [ ] No credentials or private data are present in my demo repository.

## 14. Project evidence and official references

**Verified Chapter 2 source files (13 October 2026):**

- [Chapter 2 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals)
- [backend/files.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/files.js) — synchronous local JSON response + commented serialization exercises
- [backend/users.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/users.json) — demo user
- [package.json](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/package.json) — ESM and existing Vite scripts
- [backend/mongodbweb.js](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals/blob/main/backend/mongodbweb.js) — next topic, using MongoDB

**Official documentation:**

- [Node.js file system API](https://nodejs.org/api/fs.html)
- [Node.js promises file system API](https://nodejs.org/api/fs.html#promises-api)
- [Node.js ECMAScript modules](https://nodejs.org/api/esm.html)
- [Node.js download](https://nodejs.org/en/download)
- [MDN — JSON.parse()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse)
- [MDN — JSON.stringify()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify)
- [VS Code download](https://code.visualstudio.com/download)
- [Git for Windows](https://git-scm.com/install/windows)

**Verification boundary:** Chapter-specific claims reflect the actual published source files. The standalone exercises, asynchronous helper, security guidance, and troubleshooting patterns are **recommended educational additions**, not features claimed to be already implemented in Chapter 2. The code snippets are designed to run on a modern LTS Node.js installation; the practice lab is verified separately before publication, not claimed to have been run on every possible Windows configuration.

---

**Next:** Cheatsheet 08 — MongoDB Installation & Native Node.js Driver.  
[← Chapter 2 index](../README.md) · [Full library](../../README.md)