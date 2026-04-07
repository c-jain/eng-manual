**Status:** 🌳 Evergreen  
**Created:** 2026-04-07  
**Last Updated:** 2026-04-07

<br>

# Worktree

- `git worktree` lets you check out **multiple branches simultaneously**, each in its own separate
  directory on disk — all backed by one single `.git` repository.
- normally your working tree is tied 1:1 to one checked-out branch at a time. `git worktree` breaks
  that constraint.
- the original directory is called the **main worktree**. every additional directory is a
  **linked worktree**.
- each linked worktree gets a tiny `.git` *file* (not directory) that contains a `gitdir:` pointer
  back to `.git/worktrees/<name>/` inside the shared repo — the same mechanism Git submodules use.
- all worktrees share the same object store, refs, stash, config, remotes, and hooks. a commit
  made in one worktree is immediately visible in all others.

## Why does it exists?

- **context-switching is expensive.** the classic `git stash → checkout → fix → checkout → stash pop`
  workflow trashes your editor state, rebuilds, language-server caches, and compiled artifacts every
  time.
- **parallel work on separate branches is common** — you're deep in a feature when a critical bug
  lands, or you need to QA a colleague's branch without abandoning your own.
- `git worktree` was introduced in **Git 2.5 (2015)** to solve this: keep multiple branches live on
  disk simultaneously with zero duplication of the object store.
- also useful for CI-adjacent workflows: build and test multiple branches in separate directories
  without cloning the repo multiple times.

---

## Disk anatomy

after `git worktree add -b hotfix ../project-hotfix main`, Git creates:

```
~/project/                         ← main worktree
  .git/
    objects/                       ← ONE shared object store
    refs/
    worktrees/
      project-hotfix/              ← tracking metadata for the linked worktree
        gitdir                     ← absolute path back to ~/project-hotfix/.git
        HEAD                       ← refs/heads/hotfix
        locked                     ← only present if you ran git worktree lock

~/project-hotfix/                  ← linked worktree directory
  .git                             ← a FILE (not a dir) containing:
                                      "gitdir: /home/user/project/.git/worktrees/project-hotfix"
  src/
  package.json
  ...                              ← full working tree checked out at hotfix
```

```
                ┌─────────────────────────────┐
                │   .git/  (single shared)     │
                │  objects · refs · config     │
                └────┬──────────┬─────────┬───┘
                     │          │         │
           ┌─────────▼──┐  ┌────▼────┐  ┌─▼──────────┐
           │ main worktree│  │ linked 1│  │  linked 2  │
           │  ~/project   │  │~/proj-hf│  │~/proj-feat │
           │ branch: main │  │ hotfix  │  │  feature   │
           └─────────────┘  └─────────┘  └────────────┘
                                │                │
                          .git (file)       .git (file)
                        → worktrees/hf/   → worktrees/feat/
```

key insight: **only the working tree files are duplicated**, never Git objects or history.

## Key Commands & Usage

### adding a worktree

```bash
# check out an existing branch in a new directory
git worktree add ../project-hotfix hotfix/login-crash

# create a new branch and check it out in one step
git worktree add -b feature/dark-mode ../project-darkmode main

# check out at a specific commit (detached HEAD)
git worktree add --detach ../project-review abc1234
```

- first argument: **path** to the new directory (sibling dirs like `../project-hotfix` are idiomatic)
- second argument: **branch or commit**; can be omitted to infer the branch name from the path
- `-b <branch>` creates a new branch at `HEAD` (or at a specified base) and checks it out

---

### listing worktrees

```bash
git worktree list
```

example output:
```
/home/user/project         abc1234  [main]
/home/user/project-hotfix  def5678  [hotfix/login-crash]
/home/user/project-dark    9ab0cd1  [feature/dark-mode]
```

- shows path, HEAD commit, and branch for every worktree (main + all linked)
- machine-readable output:
```bash
git worktree list --porcelain
```

### removing a worktree

```bash
# preferred: removes directory and cleans up .git/worktrees entry
git worktree remove ../project-hotfix

# force removal even with uncommitted changes
git worktree remove --force ../project-hotfix

# if you deleted the directory manually, prune the stale entry
git worktree prune
```

- `remove` refuses by default if the worktree has uncommitted changes
- `git gc` calls `prune` automatically for stale entries older than 3 months

### locking and unlocking

```bash
# lock: prevent prune from removing this entry
# useful for worktrees on removable drives or network mounts
git worktree lock ../project-hotfix --reason "on USB drive, not always mounted"

# unlock
git worktree unlock ../project-hotfix
```

- locking writes a `locked` file inside `.git/worktrees/<name>/` containing the reason string
- `prune` always skips locked worktrees, even when their directory is missing

### moving a worktree

```bash
git worktree move ../project-hotfix ../project-oldfix
```

- updates both the directory and the `.git/worktrees/` bookkeeping atomically
- available from Git 2.17+

### repairing broken paths

```bash
git worktree repair
```

- if you moved a worktree directory manually (without `git worktree move`), run this to
  re-sync the internal `gitdir` pointer files
- run from any of the affected worktrees

## real-world example

```
Scenario:
  You are on main, 4 hours into a feature.
  A critical prod bug arrives. Fix it on hotfix/session-token immediately.
  You cannot lose your feature work-in-progress.
```

```bash
# from inside the main project directory
git worktree add -b hotfix/session-token ../project-hotfix origin/main

cd ../project-hotfix
# fix the bug
git add src/auth/session.ts
git commit -m "fix: regenerate session token on login"
git push origin hotfix/session-token

# return to your feature — nothing has changed
cd ../project
# editor state, build cache, and working files are exactly as you left them
```

## Problems & Pitfalls

- **one branch per worktree, enforced.** Git refuses to check out a branch already active in
  another worktree:
  ```bash
  # ERROR: 'main' is already checked out at '/home/user/project'
  git worktree add ../project-copy main
  ```
  workaround: create a new branch from the target branch for the second worktree.

- **stale `.git/worktrees/` entries accumulate** if you delete worktree directories manually
  (common mistake) or a machine crashes mid-session. run `git worktree prune` periodically, or
  after any manual directory deletion.

- **detached HEAD propagation risk.** if you force-move a branch ref (e.g., `git branch -f main
  abc123`) while it is checked out in another worktree, that worktree silently becomes detached.

- **path-dependent tooling breaks.** IDEs, LSPs, `pre-commit`, and CI scripts that assume a single
  working-tree root at a fixed path may behave incorrectly inside a linked worktree.

- **hooks run per worktree, config is shared.** `core.hooksPath` is repo-wide. hooks that assume
  they are running in the main project root will also fire in linked worktrees, potentially with
  wrong paths or unintended side effects.

- **relative paths in scripts and Makefiles break.** any hardcoded relative path (e.g.,
  `cd ../../scripts`) that works from the main root will fail in a sibling directory at a
  different depth.

- **`git submodule` interactions.** submodules inside linked worktrees are managed independently —
  running `git submodule update` in one worktree does not update the others, which can cause
  divergent nested module state.

- **disk duplication of working tree files.** the object store is shared, but every linked worktree
  gets its own full copy of the working tree. for repos with large non-Git files (media, build
  artifacts) this cost adds up fast.

## 🔗 Further reading

- [git-worktree man page — official and authoritative](https://git-scm.com/docs/git-worktree)