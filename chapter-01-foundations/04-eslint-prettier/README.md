# Cheatsheet 04 — ESLint & Prettier Configuration

**From Zero to Fullstack · Chapter 1: Foundations**  
**Audience:** Absolute beginners + developers creating reusable JavaScript/React projects  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Version:** 1.0 · 13 October 2026  
**Project source:** [Chapter 1 — Foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install, configure, understand, test, and reuse ESLint and Prettier in React/Vite projects. Reproduce what our Chapter 1 actually contains, then optionally improve the setup for future applications.

## Quick reference — ESLint vs Prettier

| Tool | Main responsibility | Common command |
| --- | --- | --- |
| **ESLint** | Find code-quality problems and rule violations | `npm run lint` |
| **ESLint auto-fix** | Fix supported ESLint issues (not everything) | `npm run lint:fix` |
| **Prettier** | Enforce consistent source-code formatting | `npx prettier src --write` |
| **Prettier check** | Detect files that do not match formatting rules | `npx prettier src --check` |
| **VS Code extensions** | Show diagnostics and format while editing | ESLint + Prettier — Code formatter |
| **eslint-config-prettier** | Turn off ESLint rules that may conflict with Prettier | Add configuration **after** other ESLint configs |

**Remember:** ESLint and Prettier solve **different problems**. A beautifully formatted file can still have an ESLint error. Passing ESLint does not prove the file follows Prettier's style.

**Windows note:** If PowerShell blocks `npm.ps1` or `npx.ps1`, replace `npm` with `npm.cmd` and `npx` with `npx.cmd`. Both forms refer to the same installed Node.js tools.

---

## 1. What are linting and formatting?

A **linter** analyzes source code and reports patterns that may lead to defects or violate defined project rules. For example, an unused variable can trigger ESLint's `no-unused-vars` rule.

A **formatter** rewrites whitespace, indentation, quotes, line breaks, and other supported syntax presentation consistently. Prettier is opinionated: the team's style lives in one shared config instead of personal preferences.

Example, before formatting:

```js
const sayHello=(name)=>{return "Hello, "+name;}
```

After formatting **with our actual Chapter 1 Prettier settings**:

```js
const sayHello = (name) => {
  return 'Hello, ' + name
}
```

An unused variable would remain an ESLint concern even if the file were formatted correctly.

**The professional workflow:** Write code → ESLint diagnoses problems → fix code issues → Prettier formats → run checks before committing.

## 2. Prerequisites — installation from a new Windows computer

You need a Windows computer, internet access, permissions to install software, and a terminal. No prior ESLint or Prettier knowledge is required.

### 2.1 Install Visual Studio Code

1. Download the **Windows User Installer** from [code.visualstudio.com/download](https://code.visualstudio.com/download).
2. Complete the installer and open VS Code.
3. Select **Terminal → New Terminal** to open the built-in terminal.
4. Restart PowerShell after installation and verify:

```powershell
code --version
```

**Expected:** A VS Code version. If `code` is not recognized, restart the terminal or use VS Code's **File → Open Folder** menu.

### 2.2 Install Node.js and npm

1. Download a compatible **Node.js LTS** installer from [nodejs.org/en/download](https://nodejs.org/en/download).
2. Install it using the official Windows installer; npm is normally included.
3. Restart PowerShell and verify:

```powershell
node --version
npm --version
```

**Expected:** Both commands show version numbers. The Chapter 1 repository uses **Vite 7**, which requires Node.js **20.19+ or 22.12+**. Check the requirements of any newer template you use.

If `npm.ps1` is blocked by PowerShell's execution policy, **do not change the system-wide policy** just for this tutorial:

```powershell
npm.cmd --version
```

### 2.3 Install Git

1. Download [Git for Windows](https://git-scm.com/install/windows).
2. Follow the normal installation process; include Git Bash if offered.
3. Restart PowerShell and check:

```powershell
git --version
```

**Why Git?** Git lets you review the exact source changes made by `eslint --fix` or `prettier --write` and undo unwanted edits safely.

## 3. Path A — Reproduce our real Chapter 1 configuration

This path is **verified against the published source files** in [XMAN-MHD/from-zero-to-fullstack-ch1-foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations). The following is a reconstruction procedure, not a claim that we reinstalled Windows while reviewing the repository.

### Step 1 — Clone and enter the project

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
code .
```

If the project is already cloned, simply navigate into the existing folder rather than cloning it a second time.

### Step 2 — Install locked dependencies

```powershell
npm ci
```

**Expected:** npm installs the dependency versions selected by `package-lock.json`. This repository also runs a Husky `prepare` script when dependencies are installed.

### Step 3 — Verify the tools

```powershell
npm run lint
npm run lint:fix
npx eslint --version
npx prettier --version
```

**Expected:** Both versions print. `npm run lint` reports errors if present. `npm run lint:fix` attempts fixes and may modify files; inspect `git diff` afterward. This project already contains Prettier as a development dependency.

**For safer first inspection:** Run `npm run lint` before `lint:fix`; do not auto-fix unfamiliar production code without reviewing changes.

### Step 4 — Check Prettier without editing

```powershell
npx prettier src --check
```

**Expected:** Either Prettier reports that files match the shared style or lists files needing formatting. **A formatting warning does not necessarily mean the existing project is broken.**

To format only the source directory (this **writes files**):

```powershell
npx prettier src --write
git diff
```

Avoid running `npx prettier . --write` without reviewing what files will be affected. This is a source-modifying operation, not simply a read-only check.

## 4. Understand our actual ESLint setup

Our Chapter 1 defines `eslint.config.js`, using ESLint **flat configuration**. This is the project's source configuration:

```js
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import { defineConfig, globalIgnores } from 'eslint/config'

export default defineConfig([
  globalIgnores(['dist']),
  {
    files: ['**/*.{js,jsx}'],
    extends: [
      js.configs.recommended,
      reactHooks.configs['recommended-latest'],
      reactRefresh.configs.vite,
    ],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
      parserOptions: {
        ecmaVersion: 'latest',
        ecmaFeatures: { jsx: true },
        sourceType: 'module',
      },
    },
    rules: {
      'no-unused-vars': ['error', { varsIgnorePattern: '^[A-Z_]' }],
    },
  },
])
```

### Line-by-line concepts

| Part | Meaning |
| --- | --- |
| `defineConfig([...])` | Builds an ESLint flat configuration array. |
| `globalIgnores(['dist'])` | Skips Vite's generated build output. |
| `files: ['**/*.{js,jsx}']` | Applies the following settings to JavaScript and JSX files. |
| `js.configs.recommended` | Enables recommended JavaScript error-detection rules. |
| `reactHooks.configs['recommended-latest']` | Adds React Hooks lint rules. |
| `reactRefresh.configs.vite` | Checks patterns relevant to React Fast Refresh with Vite. |
| `globals.browser` | Recognizes browser global names, such as `window`. |
| `ecmaFeatures: { jsx: true }` | Allows JSX syntax in relevant files. |
| `no-unused-vars` | Reports unused bindings; the regex exempts matching names such as capitalized identifiers. |

**Important:** Our repository also lists `eslint-plugin-jsx-a11y` as a dependency, but it is **not imported or activated** in the actual `eslint.config.js`. Installing a plugin does not automatically enable its rules.

**Modern configuration note:** This project uses `eslint.config.js` (flat config), not the old `.eslintrc` format. Avoid copying snippets from legacy ESLint guides without adapting them.

## 5. Understand our actual Prettier setup

The project's `.prettierrc.json` contains:

```json
{
  "trailingComma": "all",
  "tabWidth": 2,
  "printWidth": 80,
  "semi": false,
  "jsxSingleQuote": true,
  "singleQuote": true
}
```

| Option | Effect | Example |
| --- | --- | --- |
| `trailingComma: 'all'` | Adds trailing commas where supported | `const obj = { a: 1, }` (multiline where Prettier chooses) |
| `tabWidth: 2` | Indents using two spaces of visual width | Nested code has 2-space indentation. |
| `printWidth: 80` | Preferred line-wrap width, not a hard maximum | Long expressions may wrap. |
| `semi: false` | Omits semicolons where safe | `const name = 'Awa'` |
| `jsxSingleQuote: true` | Prefers single quotes in JSX attribute values | `<Tag label='Hello' />` |
| `singleQuote: true` | Prefers single quotes for JavaScript strings | `const city = 'Dakar'` |

Prettier is installed in `devDependencies` (`^3.6.2` in the reviewed `package.json`), but the **actual npm scripts do not include** `format` or `format:check`. The scripts in the repository are `prepare`, `dev`, `build`, `lint`, `lint:fix`, and `preview`.

## 6. Path B — Configure a new React/Vite project yourself

**This is a recommended reusable template, not the exact original Git history.** To build a new frontend, see [Cheatsheet 03](../03-react-vite/README.md) and create the React template:

```powershell
Set-Location "$HOME\Projects"
npm create vite@latest quality-demo -- --template react
Set-Location quality-demo
npm install
```

Vite's generated ESLint defaults **may differ by template version**. Inspect your generated `package.json` and `eslint.config.js` first; do not overwrite working configuration blindly.

### 6.1 Install necessary development dependencies

For a React/JavaScript project that does not yet contain the equivalent tooling:

```powershell
npm install --save-dev eslint @eslint/js globals eslint-plugin-react-hooks eslint-plugin-react-refresh
npm install --save-dev --save-exact prettier
npm install --save-dev eslint-config-prettier
```

`--save-dev` places tools under `devDependencies`, because they are used during development/build validation rather than for runtime browser features. `--save-exact` pins a specific Prettier release in `package.json`. Existing dependencies need not be installed twice.

If no ESLint config exists, create one from the **working Chapter 1 configuration in Section 4**, adapting it to the selected plugin versions. Another option for guided initial configuration is `npm init @eslint/config@latest` (answer the questions for JavaScript, modules and browser/React as appropriate).

### 6.2 Create the Prettier configuration

Place `.prettierrc.json` in your project root with the values shown in Section 5. Create an optional `.prettierignore`:

```text
# Do not reformat generated output
node_modules/
dist/
coverage/
```

Prettier also respects relevant entries in `.gitignore` when run from that directory. **A Prettier ignore file is not a substitute for `.gitignore`**: Git and Prettier have different jobs.

### 6.3 Add reusable npm scripts

In your **new project's** `package.json`, merge these scripts into the existing `scripts` object; do not replace the whole object:

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier . --write",
    "format:check": "prettier . --check"
  }
}
```

This is an **illustrative `scripts` excerpt**. Keep any existing `dev`, `build`, `preview`, and `prepare` entries. With `.prettierignore` configured, the new commands become:

```powershell
npm run lint
npm run lint:fix
npm run format:check
npm run format
```

**Do not copy the full JSON block as the entire `package.json`; merge only its entries.** The original Chapter 1 does not define the `format` scripts.

### 6.4 Avoid ESLint/Prettier rule conflicts

Our source project's `package.json` already installs `eslint-config-prettier` (`^10.1.8`) but its ESLint configuration **does not yet import it**.

For a future project, after installing the package, add a final configuration entry:

```js
// Additional line near the other imports:
import eslintConfigPrettier from 'eslint-config-prettier/flat'

// Add as the final item inside defineConfig([...]):
eslintConfigPrettier,
```

**Why last?** `eslint-config-prettier` disables competing stylistic lint rules so Prettier can focus on formatting. It **does not run Prettier**, and it does not replace recommended ESLint bug-detection rules.

For a full working configuration, use Section 4 as the base and add the import and final array item above; do not paste these partial snippets as a standalone config.

## 7. Integrate both tools into Visual Studio Code

1. Open VS Code's Extensions view (**Ctrl + Shift + X**).
2. Install **ESLint** (publisher Microsoft) and **Prettier — Code formatter** (publisher Prettier).
3. Open **Settings (JSON)** from the Command Palette (Ctrl + Shift + P).
4. To apply the following settings to **all projects on your computer**, place them in *User Settings*; for one project, place them in *.vscode/settings.json*:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[javascriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "eslint.validate": ["javascript", "javascriptreact"]
}
```

**Expected:** When saving a supported JS/JSX file, Prettier formats it; ESLint displays rule diagnostics in the editor. VS Code settings alone **do not add** packages or scripts to the project.

**Source caveat:** The original Chapter 1 `.gitignore` excludes most `.vscode/*` files. If you add project-local `.vscode/settings.json`, it may be ignored by Git. For sharing team settings, deliberately adjust the `.gitignore` rule or add an explicit tracked exception; do not assume it will be committed automatically.

## 8. Hands-on exercises — prove each tool works

### Exercise A — Trigger and fix a lint error

Work inside your cloned project and create a **temporary** file using PowerShell:

```powershell
Set-Content -Path src/lint-demo.js -Value 'const unusedValue = 42'
npx eslint src/lint-demo.js
```

**Expected:** ESLint reports a `no-unused-vars` problem. This is an intentional demonstration, not a bug in the project.

Remove the temporary file after the exercise:

```powershell
Remove-Item src/lint-demo.js
npm run lint
```

**Expected:** The temporary lint error disappears. **Do not** delete an existing user-created file: choose a unique temporary filename if `src/lint-demo.js` already exists.

### Exercise B — Let Prettier format a file

Create a different temporary file:

```powershell
Set-Content -Path src/format-demo.js -Value 'const greeting = "Hello";'
npx prettier src/format-demo.js --check
npx prettier src/format-demo.js --write
Get-Content src/format-demo.js
```

**Expected:** The first `--check` reports formatting differences. `--write` rewrites the example to `const greeting = 'Hello'` according to our project's settings.

Then remove the test file and inspect your worktree:

```powershell
Remove-Item src/format-demo.js
git status
git diff
```

**Why two tests?** They demonstrate linting and formatting independently instead of assuming either tool does both jobs.

### Exercise C — Production build verification

```powershell
npm run lint
npm run build
```

**Expected:** Linting finishes without blocking errors and the Vite production build generates a `dist/` folder. **A build success is not proof Prettier has run**; check formatting separately.

## 9. How the tools relate to Git hooks and CI

Our Chapter 1 already has two Husky hooks:

```text
.husky/pre-commit  → npm run lint
.husky/commit-msg  → npx commitlint --edit "$1"
```

- The pre-commit hook attempts lint validation before a commit completes.
- Commitlint validates the **commit message**, not JavaScript code formatting.
- **Prettier does not run automatically from these Chapter 1 hooks.**

Recommended future workflow:

```text
Write code → inspect ESLint diagnostics → npm run lint
           → npm run format:check → npm run build
           → review git diff → commit → push
```

`format:check` is part of the **optional new-project setup in Section 6**, not an existing Chapter 1 command. GitHub Actions checks belong in the future CI/CD cheatsheet.

## 10. Troubleshooting and common mistakes

| Symptom | Likely cause | First fix |
| --- | --- | --- |
| `npm.ps1` or `npx.ps1` blocked | PowerShell execution policy | Use `npm.cmd` / `npx.cmd`. |
| `eslint: command not found` | Dependencies missing or wrong folder | Check `pwd` and run `npm ci`. |
| `prettier: command not found` | Prettier not installed locally | Run `npm install -D prettier` in **your** project. |
| `npm ERR! Missing script: format` | `format` is not in `package.json` | Use `npx prettier ...` or add the optional script. |
| ESLint doesn't recognize JSX | Wrong config or file matcher | Check JSX support and `**/*.{js,jsx}`. |
| `no-undef` for `window` | Browser globals not configured | Use `globals.browser` for browser files. |
| Linter checks generated files | Ignore pattern missing | Configure `globalIgnores(['dist'])`. |
| Prettier does nothing on save | Wrong default formatter / missing extension | Set VS Code's Prettier formatter. |
| Formatter changes quotes unexpectedly | Repo config not loaded | Inspect `.prettierrc.json` and working directory. |
| ESLint & Prettier conflict | Competing style rules | Add `eslint-config-prettier/flat` last. |
| Format check fails after lint passes | Separate responsibilities | Run Prettier check and review differences. |
| Hook blocks commit | Linting found errors | Fix problems; avoid blindly disabling hooks. |

**Diagnostics:**

```powershell
Get-Location
node --version
npm --version
npx eslint --version
npx prettier --version
npm run lint
npx prettier src --check
git status
```

For detailed ESLint rules affecting a specific file, the ESLint CLI supports `--print-config`. Treat printed configuration as **diagnostic output**, not as a substitute for reading `eslint.config.js`.

## 11. Security, maintainability, and team best practices

1. **Keep project-local tooling.** Install ESLint/Prettier in `devDependencies` and commit the lockfile so teams and CI use consistent versions.
2. **Avoid blanket auto-fixes.** Run a read-only check first, then use `--fix` or `--write` deliberately and inspect `git diff`.
3. **Never format secrets into Git.** Ignore `.env` and other private credential files in new projects; maintain safe `.env.example` files instead.
4. **Ignore generated files.** Do not lint or format build outputs and dependencies unnecessarily.
5. **Do not equate a formatter with security scanning.** ESLint and Prettier are helpful but cannot guarantee application security.
6. **Separate responsibilities.** ESLint is for code quality; Prettier is for style; Commitlint is for commit messages; tests validate behavior.
7. **Prefer clear shared configs.** Document any additions to the original Chapter 1 instead of silently changing the demonstrated code.
8. **Verify before publishing.** Run lint, format check (when configured), application tests (when available), and production build.

## 12. Quick reference — reuse in a future React project

The following **PowerShell workflow** assumes the new project has been created and the dependencies/configuration from Section 6 are present:

```powershell
# In your project's root directory
node --version
npm --version

# Install checks/tools if not already present
npm install --save-dev eslint @eslint/js globals eslint-plugin-react-hooks eslint-plugin-react-refresh
npm install --save-dev --save-exact prettier
npm install --save-dev eslint-config-prettier

# Configure eslint.config.js, .prettierrc.json and scripts
# Then verify:
npm run lint
npm run format:check

# Optional automatic changes (review with Git afterward)
npm run lint:fix
npm run format
git diff

# Verify production bundle
npm run build
```

**Important:** These commands do not create the config files automatically; use the exact configuration and script instructions above. Also, the `format:*` scripts are **new reusable recommendations** rather than existing Chapter 1 scripts.

## 13. Final reuse checklist

- [ ] VS Code, Node.js, npm and Git are installed and verified.
- [ ] I understand the difference between linting and formatting.
- [ ] I can install project-local development dependencies.
- [ ] I know where `eslint.config.js` and `.prettierrc.json` belong.
- [ ] I can explain our Chapter 1 ESLint plugins and Prettier rules.
- [ ] I know that a dependency being installed does **not** mean its configuration is active.
- [ ] I can run read-only lint and format checks.
- [ ] I can deliberately apply fixes and inspect the Git diff.
- [ ] I can configure VS Code without confusing editor extensions and npm packages.
- [ ] I know how to add `eslint-config-prettier` to a future flat config.
- [ ] I can fix PowerShell command errors without weakening machine-wide security.
- [ ] I can separate **source-verified** behavior from **optional improvements**.

**Outcome:** You can establish and maintain consistent JavaScript/React code quality standards in future applications.

## 14. Source evidence and official documentation

### Source-reviewed project files

- [Chapter 1 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)
- [`eslint.config.js`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/eslint.config.js)
- [`.prettierrc.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.prettierrc.json)
- [`package.json`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/package.json)
- [`.husky/pre-commit`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.husky/pre-commit)
- [`.husky/commit-msg`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.husky/commit-msg)
- [`.gitignore`](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.gitignore)

### Official references

- [ESLint — Getting started](https://eslint.org/docs/latest/use/getting-started)
- [ESLint — Flat configuration files](https://eslint.org/docs/latest/use/configure/configuration-files)
- [Prettier — Installation](https://prettier.io/docs/install)
- [Prettier — CLI (`--check` and `--write`)](https://prettier.io/docs/cli)
- [Prettier — Editor integration](https://prettier.io/docs/editors)
- [eslint-config-prettier — Flat config](https://github.com/prettier/eslint-config-prettier)

**Verification boundary:** Project-specific facts are based on the published Chapter 1 source as reviewed on **13 October 2026**. General installation, suggested npm scripts, editor integration, and adding `eslint-config-prettier` are explicitly labeled as **reusable recommendations**. We did not modify the Chapter 1 project in preparing this document.

---

**Next:** [Cheatsheet 05 — Git Hooks with Husky & Commitlint](../05-husky-commitlint/README.md) (planned).  
[← Chapter 1 index](../README.md) · [Full library roadmap](../../README.md)