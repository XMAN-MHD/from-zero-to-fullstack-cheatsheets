# Cheatsheet 02 — Git & GitHub from Scratch

**From Zero to Fullstack · Chapter 1: Foundations**  
**Audience:** Complete beginners + reusable professional reference  
**Primary platform:** Windows 10/11 · PowerShell · Visual Studio Code  
**Version:** 1.0 · 13 October 2026  
**Source:** [Chapter 1 — Foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)

> **Learn → Practice → Document → Reuse.**  
> **Goal:** Install and configure Git, understand the complete local-to-GitHub workflow, create and publish a repository, work safely with branches, and recover from common mistakes.

## Quick start: the complete workflow in 30 seconds

```text
YOUR COMPUTER                                    GITHUB
Working directory  →  Staging area  →  Local history   →   Remote repository
(edit files)          (git add)       (git commit)         (git push)
                                                          ↓
                                  git fetch / git pull  ←  Changes from others
```

| Task | Command |
| --- | --- |
| Verify installation | `git --version` |
| See what's changed | `git status` |
| Preview edits | `git diff` |
| Stage a file | `git add README.md` |
| Save a commit | `git commit -m "docs: add project overview"` |
| View history | `git log --oneline --graph --decorate --all` |
| See remote addresses | `git remote -v` |
| Download updates safely | `git pull --ff-only` |
| Upload current branch | `git push` |

**Important:** Git and GitHub are different products. A local commit does **not** automatically upload anything. `git push` sends commits to a remote Git repository.

---

## 1. Prerequisites: install everything from a fresh computer

### 1.1 What you need

- A Windows computer with an internet connection.
- Permission to install software.
- A text editor; we use **Visual Studio Code**.
- A **GitHub account**, only for publishing online. Local Git works without an account.

Node.js is **not required for Git itself**. You only need Node.js/npm later when running our React/Vite Chapter 1 starter.

### 1.2 Install Git for Windows

1. Visit [https://git-scm.com/install/windows](https://git-scm.com/install/windows).
2. Download the official **Git for Windows** installer.
3. Follow the setup wizard. Recommended for beginners: retain **Git Bash**, **Git Credential Manager** (if offered), and command-line access from PowerShell.
4. Finish installation, then **close and reopen** PowerShell.
5. Verify:

```powershell
git --version
where.exe git
```

**Expected:** A version such as `git version 2.x.x.windows.x` and a path to `git.exe`. Version numbers vary.

> **Windows tip:** Open PowerShell from the Start menu or VS Code's **Terminal → New Terminal**. In this guide, commands are PowerShell-compatible. Do not paste Bash-only commands such as `touch` or `export` into PowerShell.

### 1.3 Install VS Code (if needed)

1. Visit [https://code.visualstudio.com/download](https://code.visualstudio.com/download).
2. Install the official Windows version.
3. Reopen your terminal.
4. Verify with `code --version`.

Use `code .` **inside a project folder** to open that folder. If the command is unavailable, open the folder using **File → Open Folder**.

### 1.4 Set up GitHub

1. Create a free account at [https://github.com/signup](https://github.com/signup), if you do not already have one.
2. Verify your email, use a strong password, and enable **two-factor authentication (2FA)**.
3. Sign in and check that you can open your profile.

You can clone **public** repositories without logging in; pushing requires authentication and permission.

---

## 2. Configure Git for the first time

Set the author identity that will appear on your **future commits**:

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global --list
```

Use your own name and a real email associated with GitHub, or your GitHub **noreply** email to protect your private address. The `--global` option stores values for your user on this machine. A specific repository can override these settings using the same commands **without** `--global`.

Verify only the relevant keys:

```powershell
git config --global user.name
git config --global user.email
git config --list --show-origin
```

**Expected:** Your chosen values. `--show-origin` reveals the files supplying each configuration setting.

**Optional:** Configure VS Code as the Git editor:

```powershell
git config --global core.editor "code --wait"
```

If `code --version` failed earlier, skip this setting until you fix the VS Code command.

### Two kinds of identity

- **Commit author:** `user.name` and `user.email` stored in Git commits.
- **GitHub authentication:** permission to push, obtained through Git Credential Manager, SSH, or a token. Setting `user.email` does **not** log you in.

---

## 3. Understand Git's four places

```text
WORKING TREE       STAGING AREA        LOCAL REPOSITORY        REMOTE REPOSITORY
Your edited files  Proposed snapshot   Saved commit snapshots  GitHub copy
      |                  |                   |                       |
      +--- git add ----->+--- git commit --->+------ git push ------>+
      |                  |                   |<----- git fetch ------+
      +<------ git restore / merge / pull (depending on operation) --+
```

1. **Working tree** — the files on disk, including new and edited files.
2. **Staging area (index)** — what your **next commit** will contain.
3. **Local repository (`.git/`)** — the project history stored on your computer.
4. **Remote repository** — a hosted repository, commonly GitHub. Its usual remote name is `origin`.

A **commit** is a saved snapshot with an identifier and message. A **branch** is a movable name pointing to a line of commits. `HEAD` usually means the branch/commit you currently have checked out.

### What `git status` can tell you

- **`Untracked files`** — Git has not begun tracking these files. Inspect, then run `git add FILE` if appropriate.
- **`Changes not staged`** — A tracked file has changed, but is not staged. Run `git diff` and then `git add FILE`.
- **`Changes to be committed`** — Staged changes are ready for a commit. First review with `git diff --staged`.
- **`nothing to commit, working tree clean`** — No pending tracked changes. Proceed to the next task.

---

## 4. Practice A — create a repository locally from zero

**This is an additional learning exercise**, not a claim that this exact sequence was used in our Chapter 1 history. We create a tiny documentation project that requires no Node.js.

### Step 1: create a directory in PowerShell

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects"
New-Item -ItemType Directory -Force -Path "$HOME\Projects\my-first-git-project"
Set-Location "$HOME\Projects\my-first-git-project"
code .
```

**Expected:** Both folders exist. Check your location with `pwd` before continuing.

### Step 2: initialize Git

```powershell
git init
git branch --show-current
git status
```

**Expected:** A `.git` repository is created, the initial branch is `main` (if configured), and no commits exist yet. The command `git branch --show-current` may display `main` even before the first commit.

### Step 3: create a file

```powershell
Set-Content -Path README.md -Value "# My First Git Project"
Get-Content README.md
git status
```

**Expected:** `README.md` appears as untracked.

### Step 4: stage and commit

```powershell
git add README.md
git diff --staged
git commit -m "docs: add initial README"
git log -1 --oneline
git status
```

**Expected:** A new commit appears and the working tree is clean. A successful local commit is not yet on GitHub.

### Step 5: modify, inspect, and commit again

```powershell
Add-Content -Path README.md -Value "Learning Git step by step."
git diff
git add README.md
git diff --staged
git commit -m "docs: describe learning goals"
git log --oneline --decorate
```

This is your first complete **edit → stage → commit** loop.

**Tip:** Prefer staging specific files (`git add README.md`) while learning. `git add .` stages **all** eligible changes beneath the current directory, which may include files you did not intend to commit.

---

## 5. Publish your local project to GitHub

### Step 1: create an empty GitHub repository

1. Sign in at [https://github.com/new](https://github.com/new).
2. Choose a repository name, for example **`my-first-git-project`**.
3. Choose **Public** or **Private**.
4. **Do not initialize with a README, license, or `.gitignore`** for this exercise: you already have a local first commit. This avoids unrelated histories.
5. Select **Create repository** and copy its HTTPS URL.

### Step 2: connect `origin` and publish

Replace `YOUR-USERNAME` with your own GitHub username:

```powershell
git remote add origin https://github.com/YOUR-USERNAME/my-first-git-project.git
git remote -v
git push -u origin main
```

**Expected:** The remote is registered, authentication may open a browser, and GitHub receives the commits. Refresh your GitHub repository page to see `README.md`.

- `origin` is the conventional remote nickname; it is **not** a special server.
- `-u` sets the upstream so that later `git push` and `git pull` can often omit the remote and branch.
- A push **uploads commits**, not every uncommitted file on disk.

**If the remote already exists:** inspect `git remote -v` first. Only if you **intend** to replace the address, use:

```powershell
git remote set-url origin https://github.com/YOUR-USERNAME/my-first-git-project.git
```

Do **not** switch an existing project's remote to a new destination by accident.

---

## 6. Practice B — clone our real Chapter 1 source

Our reviewed source repository is:

[https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)

```powershell
Set-Location "$HOME\Projects"
git clone https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations.git
Set-Location from-zero-to-fullstack-ch1-foundations
git status
git remote -v
git log --oneline --graph --decorate -5
code .
```

**Expected:** A local copy with an `origin` pointing to the XMAN-MHD repository. The files include `package.json`, `.husky/`, `eslint.config.js`, and `src/`.

**Don't run `git init` inside a cloned repository**: it already contains a `.git/` directory and history.

### What was actually verified in Chapter 1

- Its published README teaches Git/GitHub setup, commits, branches, merging, and remote collaboration as **learning objectives**.
- Its Git history currently includes commits with messages such as `first commit`, `add ch1`, and later README updates. Those historical messages do **not** imply the current Commitlint convention was enforced from the beginning.
- Its `.husky/pre-commit` runs `npm run lint`.
- Its `.husky/commit-msg` runs `npx commitlint --edit "$1"` and the config extends `@commitlint/config-conventional`.

These are **source-verified configurations**. The examples for branches, pull requests, and merge conflicts below are **new reusable exercises**, not claims about already-merged Chapter 1 pull requests.

> **Permission note:** You can clone the public Chapter 1 project, but you cannot push to its original repository unless its owner grants you write access. Fork it to your account or work in a repository you own.

---

## 7. Branches: work safely without changing `main`

### Why branches?

Use a separate branch for a focused change, such as writing documentation, fixing a bug, or adding a feature. It gives your team a place to review changes before they affect the main branch.

Use your **own** `my-first-git-project` repository from Practice A:

```powershell
Set-Location "$HOME\Projects\my-first-git-project"
git switch main
git pull --ff-only
git switch -c docs/add-setup-notes
git branch --show-current
```

Create an example change:

```powershell
Set-Content -Path SETUP.md -Value "# Setup Notes"
git add SETUP.md
git commit -m "docs: add setup notes"
git push -u origin docs/add-setup-notes
```

**Expected:** Your GitHub repository contains a new branch. From GitHub, you may see **Compare & pull request**. Open the pull request, describe what changed, review the difference, and merge it if appropriate.

After the pull request is merged on GitHub, sync your local `main`:

```powershell
git switch main
git pull --ff-only
git branch -d docs/add-setup-notes
```

**Important:** `git branch -d` may refuse deletion if your local Git cannot verify the branch was merged into the current local history. In that case, check the GitHub merge and your local log first; don't force-delete a branch without confirming its work is preserved. Optionally remove the remote branch via GitHub's UI after merging.

### Merging locally (alternative to a pull request)

In a local exercise on a branch you created, you may merge changes into `main`:

```powershell
git switch main
git merge docs/add-setup-notes
```

This is **an alternative workflow**; do not run it after the same changes were already merged through a pull request. For shared repositories, prefer reviewed pull requests and follow branch protection rules.

### Main branch vs feature branch

| Branch | Typical use |
| --- | --- |
| `main` | Stable integration branch |
| `feature/login-page` | New user-facing capability |
| `fix/post-filter` | Bug correction |
| `docs/git-guide` | Documentation-only change |

---

## 8. Collaborate using fetch, pull, push, and pull requests

### Fetch vs pull

```powershell
git fetch origin
git status
git log --oneline --decorate --graph --all -10
```

`git fetch` downloads remote Git information **without merging** it into your current branch.

```powershell
git pull --ff-only
```

`git pull --ff-only` fetches and updates your local branch **only when Git can fast-forward**. It safely refuses to invent a merge commit if histories diverge; you must then investigate how to reconcile them.

`git push` uploads committed changes to the upstream remote branch. It does **not** automatically create a pull request.

### Pull-request workflow (recommended)

```text
Update local main → Create feature branch → Implement + test
            → Stage + commit → Push branch → Open pull request
            → Review + checks → Merge on GitHub → Pull updated main
```

A pull request (PR) proposes changes between branches and allows review, discussion, and automated checks. Its precise required checks depend on the repository settings.

### GitHub vs local responsibilities

- **Git**: local history, commits, diffs, branches, merges, remotes.
- **GitHub**: repository hosting, PR review, issues, permissions, Actions and settings.
- **GitHub Actions**: CI/CD automation (covered separately in the DevOps chapter).

---

## 9. `.gitignore`, files, and protecting secrets

Create a `.gitignore` in a **new JavaScript project** before committing:

```gitignore
# Installed dependencies and build output
node_modules/
dist/
coverage/

# Local configuration and secrets
.env
.env.*
!.env.example

# OS / editor noise
.DS_Store
Thumbs.db
```

**Explanation:** `.gitignore` prevents **untracked** matching files from being added through ordinary staging. It does not magically untrack or delete files already committed. Never store live database passwords, API keys, JWT secrets, or deployment credentials in Git.

Inspect before staging:

```powershell
git status --short
git diff
git add README.md
git diff --staged
git status
```

If a secret-bearing `.env` file was **already tracked**, stop and rotate exposed credentials where needed, then remove that file from **future tracking** while keeping the local copy:

```powershell
git rm --cached .env
```

Add `.env` to `.gitignore`, commit the changes, and consult GitHub's sensitive-data removal guidance as needed. **Removing a secret from the latest commit does not remove it from Git history**. Do not rely on `.gitignore` to erase an exposure.

### Check what is ignored

```powershell
git check-ignore -v .env
```

**Expected:** The matching `.gitignore` rule is reported when `.env` matches an ignore pattern and is not tracked. If you already track it, handle that separately.

---

## 10. Commit messages and Husky (our actual Chapter 1 configuration)

In professional projects, structured commit messages make history easier to understand and automate.

**Conventional Commits format:**

```text
type(optional-scope): brief description
```

Examples:

```powershell
git commit -m "feat: add blog search"
git commit -m "fix: handle missing post"
git commit -m "docs: explain development setup"
git commit -m "chore: update project tooling"
```

Use one actual `git commit` for each meaningful change. The lines above are **examples**, not four commands to run without new staged changes.

In [Chapter 1](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations):

| Actual project file | Verified behavior |
| --- | --- |
| `.husky/pre-commit` | `npm run lint` before committing |
| `.husky/commit-msg` | `npx commitlint --edit "$1"` checks message |
| `commitlint.config.js` | Extends `@commitlint/config-conventional` |
| `package.json` | `prepare` script runs `husky` |

**You must install dependencies for hooks to work:** `npm ci` from the Chapter 1 folder. The default template or other repositories do not automatically have these hooks. If the commit is rejected, read the displayed lint/Commitlint error, fix the underlying issue, and retry. **Do not bypass hooks** just to make errors disappear.

---

## 11. Understand and resolve a merge conflict

A merge conflict occurs when Git cannot safely combine two sets of edits, for example competing changes to the same line.

Typical markers inside a conflicting text file look like this:

```text
<<<<<<< HEAD
Our text from the currently checked-out branch
=======
Their text from the incoming branch
>>>>>>> feature/example
```

**These markers are illustrative. Never leave them in the final committed file.**

### Safe resolution steps

1. Run `git status` to find the conflicted files.
2. Open a conflicted file in VS Code.
3. Decide what the correct final text should be (sometimes combining both sides).
4. Delete the marker lines after resolving the content.
5. Re-run tests or check the file carefully.
6. Stage the resolved file and complete the merge:

```powershell
git status
git add README.md
git commit
```

The final `git commit` applies when an in-progress merge requires a merge commit. Some merge situations are completed automatically; check the state with `git status` before running commands.

To back out of an **uncommitted merge in progress**, where safe:

```powershell
git merge --abort
```

If `merge --abort` cannot restore the situation because of other changes, stop and inspect the worktree instead of running destructive reset commands. Consult GitHub's merge-conflict documentation for more complex cases.

---

## 12. Undo changes without losing your work

**First, always run `git status` and `git diff` and decide whether changes are already committed.**

- **Unstage:** `git restore --staged README.md` — keeps your actual file edits.
- **Discard local edits:** `git restore README.md` — **destroys** unsaved/uncommitted edits to that tracked file. Check before running.
- **Inspect a commit:** `git show COMMIT_SHA` — read-only; replace the placeholder with a real commit SHA.
- **Save unfinished work temporarily:** `git stash push -m "wip: notes"` — by default, untracked files are **not** included.
- **Bring a stash back:** `git stash pop` — may produce conflicts.
- **Undo a shared published commit:** `git revert COMMIT_SHA` — creates a new inverse commit, preserving existing history.

**Avoid `git reset --hard`, force push, and deleting `.git/`** as beginner shortcuts. They can permanently discard work or rewrite shared history.

A safe way to undo a shared published change is usually **`git revert`**, not rewriting history. For security incidents involving exposed secrets, removing the commit from visible history requires additional careful steps and credential rotation.

---

## 13. Troubleshooting: common real-world problems

- **`git` not recognized** — Git is missing from PATH. Install Git for Windows, restart the terminal, and check `where.exe git`.
- **`fatal: not a git repository`** — You are outside a repository. Run `pwd` and `ls`, then enter the correct cloned/initialized folder.
- **`Author identity unknown`** — `user.name`/`user.email` are missing. Configure them in Section 2.
- **`nothing to commit`** — No staged changes. Use `git status`, then edit and stage a file first.
- **`remote origin already exists`** — `origin` is configured. Inspect `git remote -v`; change it only if necessary.
- **`src refspec main does not match any`** — Possibly wrong branch name or no initial commit. Check `git branch --show-current` and `git log -1`.
- **`Repository not found`** — Repository URL is wrong, the repo is private, or you lack access. Check the owner, URL, visibility and your login.
- **Authentication failed** — Credentials are wrong or expired. Use Git Credential Manager browser login or another authorized method.
- **`Permission denied (publickey)`** — SSH is not configured for this account. Use HTTPS or configure SSH correctly.
- **Push rejected (`non-fast-forward`)** — Remote has commits not in your local branch. Fetch and inspect history; don't force-push.
- **`would be overwritten by merge`** — Local edits conflict with incoming changes. Commit or stash intended changes before pulling.
- **`git pull --ff-only` fails** — Histories diverged. Fetch and inspect before choosing how to reconcile them.
- **`refusing to merge unrelated histories`** — Local and hosted repos were initialized independently. Start with an empty remote or clone first; do not reflexively add `--allow-unrelated-histories`.
- **Commit rejected by hook** — Lint/Commitlint check failed. Read the output and repair code or message.
- **Merge conflict markers** — Some edits could not be combined. Follow Section 11.

### Diagnostics to copy into support requests

```powershell
pwd
git --version
git status
git branch -vv
git remote -v
git log --oneline --graph --decorate --all -10
git config --list --show-origin
```

**Redact** private remote URLs, usernames/email addresses where necessary, credentials, tokens, and confidential commit messages before sharing logs.

---

## 14. Reuse recipes for your future projects

### Recipe A — an existing public project

```powershell
git clone https://github.com/OWNER/REPOSITORY.git
Set-Location REPOSITORY
git status
git remote -v
```

Replace placeholders with the exact repo URL and folder name. Then read that project's `README.md` for its install/test scripts.

### Recipe B — a completely new local project

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\Projects\new-project"
Set-Location "$HOME\Projects\new-project"
git init
Set-Content README.md "# New Project"
git add README.md
git commit -m "docs: initialize project"
# Create an EMPTY GitHub repository in your browser first.
git remote add origin https://github.com/YOUR-USERNAME/new-project.git
git push -u origin main
```

Create `$HOME\Projects` first if it does not exist. Never run the placeholder URL unchanged.

### Recipe C — typical day-to-day change

```powershell
git switch main
git pull --ff-only
git switch -c feature/my-change
# Edit and test your files.
git status
git diff
git add PATH-TO-MY-CHANGED-FILE
git diff --staged
git commit -m "feat: implement my change"
git push -u origin feature/my-change
# Open and review the pull request on GitHub.
```

Replace the file placeholder before running `git add`. A branch should describe its actual purpose, and the commit type should match what changed.

---

## 15. Final beginner-to-reuse checklist

- [ ] I installed Git from the official site and `git --version` works.
- [ ] I configured my name, email, and default branch.
- [ ] I understand the difference between Git and GitHub.
- [ ] I can explain working tree → staging → commit → remote.
- [ ] I created a local repo and made at least two commits.
- [ ] I created an **empty** GitHub repo and pushed using `origin`.
- [ ] I know the difference between `git clone` and `git init`.
- [ ] I cloned and inspected the real Chapter 1 source.
- [ ] I created and published a feature branch.
- [ ] I know how to open a pull request and sync `main` after merge.
- [ ] I can use `git fetch`, `git pull --ff-only`, and `git push` appropriately.
- [ ] I understand why `.env` secrets must not be committed.
- [ ] I can resolve a basic merge conflict, or safely stop and seek help.
- [ ] I know which undo commands preserve my changes.

**Outcome:** You can create, track, collaborate on, and publish your next full-stack project with a safe Git workflow.

---

## 16. Source evidence, limits, and official documentation

### Source-verified facts (Chapter 1)

- [Chapter 1 repository](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations)
- [README learning objectives](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/README.md)
- [Actual package scripts](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/package.json)
- [Commitlint configuration](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/commitlint.config.js)
- [Husky pre-commit hook](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.husky/pre-commit)
- [Husky commit message hook](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations/blob/main/.husky/commit-msg)

### Official references used for further teaching

- [Install Git](https://git-scm.com/install/windows)
- [Pro Git — First-Time Git Setup](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [Pro Git — Git Basics](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository)
- [GitHub — Creating a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository)
- [GitHub — HTTPS authentication](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)
- [GitHub — Managing personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [GitHub — Creating a pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)
- [GitHub — Resolving merge conflicts](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/resolving-a-merge-conflict-using-the-command-line)
- [GitHub — Removing sensitive data](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)

**Review boundary:** Chapter 1 configurations, files, README, and public commit messages were checked on 13 October 2026. Local repo creation, branch/PR workflows, and merge conflict examples are **teaching exercises** based on Git/GitHub official documentation; they are not represented as historical actions already performed in Chapter 1. Results shown as “Expected” are illustrative and depend on the learner's environment and permissions.

---

**Next:** [Cheatsheet 03 — React Project Setup with Vite (planned)](../)  
[← Chapter 1 index](../) · [← Complete library roadmap](../../README.md)