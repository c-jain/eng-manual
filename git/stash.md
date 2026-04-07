**Status:** 🌳 Evergreen  
**Created:** 2026-04-07  
**Last Updated:** 2026-04-07

<br>

# Stash

`git stash` temporarily **shelves** (stashes) changes you have made to your working tree and index so you can switch context — then come back later and re-apply them.

Think of it as a **clipboard for your uncommitted work**: you pick up everything on your desk, put it in a drawer, clear the desk, do something else, and then pull the drawer open again when you're ready.

### What gets stashed by default

| Area | Stashed? |
|---|---|
| Modified tracked files (working tree) | ✅ Yes |
| Staged changes (index) | ✅ Yes |
| New untracked files | ❌ No (opt-in with `-u`) |
| Ignored files | ❌ No (opt-in with `-a`) |

> **Key point:** After a `git stash push`, your working tree and index are restored to the state of `HEAD` — clean.

## Why Does It Exist?

### The problem it solves

Git's fundamental rule is: **you cannot switch branches if you have uncommitted changes that would be overwritten by the switch**.

```
$ git checkout main
error: Your local changes to the following files would be overwritten by checkout:
    src/auth.js
Please commit your changes or stash them before you switch branches.
```

Before `git stash` existed, your only options were:
- Make a messy "WIP" commit you'd have to clean up later
- Copy files out of the repo, switch, come back, and copy them back

`git stash` provides a **lightweight, first-class mechanism** to save and restore partial work without polluting your commit history.

### Real-world scenarios

| Scenario | How stash helps |
|---|---|
| Hot-fix urgency: you're mid-feature but `main` has a production bug | Stash feature work → fix bug → unstash |
| Code review: a colleague asks you to review their branch | Stash → checkout their branch → review → pop |
| Wrong branch: you started work on `main` instead of a feature branch | Stash → checkout correct branch → pop |
| Experiment: try something risky without committing | Stash current state as a save point, then experiment |
| Pull with conflicts: `git pull` fails because of local changes | Stash → pull → pop |

## How Stash Works Internally

### The stash stack

`git stash` maintains a **stack** (LIFO — Last In, First Out) of stash entries stored as special commits in `refs/stash`.

```
refs/stash
    │
    ▼
┌─────────────────────────────────┐
│  stash@{0}  ← most recent       │  ← push adds here
│  stash@{1}                      │
│  stash@{2}                      │
│  stash@{3}  ← oldest            │  ← pop removes from top
└─────────────────────────────────┘
```

Each stash entry is actually **two or three real commits** stored as a small DAG (Directed Acyclic Graph) hanging off `refs/stash`.
 
#### The commit DAG
 
`W` (the stash commit, `stash@{0}`) is a **merge commit**. Its parents are listed in order:
- **1st parent** (`stash@{0}^1`) → `H`: the HEAD commit at the time of the stash
- **2nd parent** (`stash@{0}^2`) → `I`: snapshot of the **Index** (staged changes)
- **3rd parent** (`stash@{0}^3`) → `u`: snapshot of **untracked files** (only with `-u`/`-a`)
 
Arrows below point from child → parent (standard Git DAG convention):
 
```
refs/stash
    │
    ▼
    W   (stash@{0})        ← merge commit; your working-tree snapshot
    |\ \
    |  \ \
    |   \ '————————————> u  (stash@{0}^3)   ← untracked files
    |    \                    (only with -u or -a)
    |     \
    |      ——————————————> I  (stash@{0}^2)  ← staged changes (Index)
    |
    '————————————————————> H  (stash@{0}^1)  ← HEAD at time of stash
                               |
                               ▼
                         (prior commits…)
```
 
**Legend:**
 
| Node | `stash@{0}` ref | What it stores | Role |
|---|---|---|---|
| `W` | `stash@{0}` | Working-tree snapshot | The stash commit itself (merge commit) |
| `H` | `stash@{0}^1` | The HEAD commit you were on | 1st parent — base of everything |
| `I` | `stash@{0}^2` | Staged changes (Index) | 2nd parent |
| `u` | `stash@{0}^3` | Untracked/ignored files | 3rd parent (only with `-u`/`-a`) |
 
> `W` being a merge commit with `H`, `I`, and `u` as parents is exactly what lets `git stash apply --index` reconstruct the staged vs. unstaged split faithfully.

#### Before and after `git stash push`
 
```
BEFORE:
  HEAD (H)      Index           Working Tree
  ──────────    ──────────      ──────────────────
  auth.js v1    auth.js v2      auth.js v3
  utils.js v1   utils.js v1     utils.js v1
                                newfile.js          ← untracked
 
AFTER `git stash push -u`:
  HEAD (H)      Index           Working Tree        refs/stash
  ──────────    ──────────      ──────────────      ─────────────────────
  auth.js v1    auth.js v1      auth.js v1          W (stash@{0})
  utils.js v1   utils.js v1     utils.js v1         ├── I: auth.js v2 (staged)
                                                    ├── W: auth.js v3 (WT)
                                                    └── u: newfile.js (untracked)
```
 
Both index and working tree are reset to `H`. Your changes are fully captured in the DAG above.
 
This is why stash entries look like commits in `git log` and `git reflog` — they literally *are* commits, just reachable only via `refs/stash` rather than a branch.

## Key Commands & Usage

### `git stash push` (save)

```bash
# Basic stash (tracked changes only)
git stash

# With a descriptive message
git stash push -m "WIP: add OAuth login flow"

# Include untracked files
git stash push -u
git stash push --include-untracked

# Include everything (untracked + ignored)
git stash push -a
git stash push --all

# Stash only specific files (partial stash)
git stash push -m "just the auth changes" -- src/auth.js src/login.js

# Keep changes in the index after stashing (--keep-index)
# Useful to test: "does my staged code work without the working-tree noise?"
git stash push --keep-index
```

### `git stash list`

```bash
git stash list
# Output:
# stash@{0}: On feature/auth: WIP: add OAuth login flow
# stash@{1}: WIP on main: 1a2b3c4 fix navbar
```

### `git stash show`

```bash
# Summary diff of stash@{0} (most recent)
git stash show

# Full diff
git stash show -p
git stash show --patch

# Show a specific stash
git stash show stash@{2}
git stash show -p stash@{2}
```

### `git stash pop` (apply + drop)

```bash
# Apply most recent stash and drop it from the stack
git stash pop

# Apply a specific stash and drop it
git stash pop stash@{2}

# Apply to a specific index state too (restores staged/unstaged split)
git stash pop --index
```

> ⚠️ On conflict, the stash entry is **kept** and must be manually dropped after resolving.

### `git stash apply` (apply without dropping)

```bash
# Apply most recent stash but KEEP it in the stack
git stash apply

# Apply specific stash
git stash apply stash@{1}

# Restore the index state as well
git stash apply --index
```

> Use `apply` when you want to apply the same stash to multiple branches, or when you're unsure and want a safety net.

### `git stash drop`

```bash
# Drop the most recent stash
git stash drop

# Drop a specific stash
git stash drop stash@{2}
```

### `git stash clear`

```bash
# Delete ALL stash entries — IRREVERSIBLE (practically)
git stash clear
```

> ⚠️ This cannot be undone with a simple command. The commits still exist until GC runs, but recovery is difficult.

### `git stash branch`

```bash
# Create a new branch from the commit where the stash was made,
# apply the stash there, and drop it
git stash branch feature/oauth-wip

# From a specific stash
git stash branch bugfix/navbar stash@{1}
```

This is the **safest way to apply a stash** — it checks out the exact commit the stash was based on, so conflicts are impossible by definition.

### Recovering a dropped stash

If you accidentally drop a stash, the underlying commits are still reachable via the reflog (until GC):

```bash
# Find the dangling stash commit
git fsck --no-reflogs | grep "dangling commit"

# Inspect it
git log --oneline <sha>
git show <sha>

# Recover by applying it
git stash apply <sha>
```

## Common Workflows & Examples

### Workflow A: Context switch mid-feature

```bash
# You're on feature/auth, mid-work
git stash push -u -m "WIP: OAuth middleware"

# Switch to fix a bug on main
git checkout main
# ... fix bug, commit ...

# Return to your feature
git checkout feature/auth
git stash pop
```

### Workflow B: Partial stash (stash only some files)

```bash
# Only stash changes to two specific files
git stash push -m "auth changes only" -- src/auth.js tests/auth.test.js

# Now src/utils.js changes remain in working tree
# Work on utils, commit it
git add src/utils.js
git commit -m "fix: update utility helpers"

# Restore auth changes
git stash pop
```

### Workflow C: Test your staged code in isolation

```bash
# Stage some changes
git add src/auth.js

# Stash only working-tree changes, keep index intact
git stash push --keep-index -m "unstaged noise"

# Run tests — only staged code is active
npm test

# Restore working-tree changes
git stash pop
```

### Workflow D: Wrong branch — move work to correct branch

```bash
# Oops: you made changes on main instead of feature/my-feature
git stash push -u -m "changes meant for feature branch"

git checkout feature/my-feature
git stash pop

# Now commit properly
git add .
git commit -m "feat: implement my feature"
```

### Workflow E: Apply stash safely with a new branch

```bash
# Stash is old and you're not sure about conflicts
git stash branch feature/stash-recovery stash@{0}
# → checks out the original commit, applies stash, drops it
# → you can now commit cleanly or cherry-pick to another branch
```

### Workflow F: Recovering a dropped stash

```bash
git fsck --no-reflogs | grep "dangling commit"
# dangling commit a1b2c3d4e5f6...

git show a1b2c3d4e5f6
# Confirm it's your stash

git stash apply a1b2c3d4e5f6
```

## Stash vs. Alternatives

| Approach | When to use | Pros | Cons |
|---|---|---|---|
| `git stash` | Short-lived context switch | Quick, reversible, no history noise | Local only, easy to forget, no branch tracking |
| WIP commit (`git commit -m "WIP"`) | You want it in remote too | Pushed to remote, survives clones | Pollutes history; need `git commit --amend` or `git reset` to clean up |
| `git worktree` | Parallel work on two branches simultaneously | No stashing needed; both trees live side-by-side | More disk space; unfamiliar to many |
| New branch | Long-lived partial work | Proper history, shareable | Heavier; not for quick context switches |

### Quick Decision Guide

```
Need to switch context quickly?
    ├── Will you be back soon AND don't need it on remote?
    │       └── git stash push -u -m "..."
    ├── Want it backed up remotely?
    │       └── WIP commit + git push
    └── Need to work on both branches simultaneously?
                └── git worktree
```

## Problems & Pitfalls

### Stash is not a backup

Stash entries live **only in your local repository**. They are not pushed with `git push`. If you delete the repo or the reflog expires, stash entries are gone.

> ✅ For anything you want to keep long-term, make a real commit or a branch.

### Pop conflicts

`git stash pop` attempts a merge. If the stash conflicts with the current state:

```
$ git stash pop
Auto-merging src/auth.js
CONFLICT (content): Merge conflict in src/auth.js
The stash entry is kept in case you need it again.
```

- The stash entry is **not dropped** automatically on conflict (unlike a successful pop)
- You must resolve conflicts manually, then `git stash drop` the entry yourself

### Untracked files are silently excluded

By default, `git stash push` does **not** stash untracked files. If you stash, switch branches, and come back, new files you created are still sitting in your working tree — they don't follow the stash.

```bash
# This new file is NOT saved in the stash
touch newfeature.js
git stash          # stashes tracked changes only
git checkout main  # newfeature.js still here!
```

> ✅ Always use `-u` (or `--include-untracked`) when you have new files.

### Ignored files are doubly excluded

`-u` includes untracked files but **not** ignored files. To stash everything including ignored files, use `-a` (`--all`). Be careful — this can stash build artifacts, `.env` files, `node_modules`, etc.

### Stash does not track the branch you were on

When you later `git stash pop`, Git applies the stash to whatever branch you're currently on — it has no memory of where the stash originated. This can cause unexpected conflicts.

> ✅ Use `git stash branch <branchname>` to create a branch from the stash's original commit, which avoids conflicts.

### Multiple stashes are easy to lose track of

A long-lived stash list becomes confusing:

```
stash@{0}: WIP on feature/auth: 3a4b5c6 add login
stash@{1}: WIP on feature/auth: 3a4b5c6 add login
stash@{2}: WIP on main: 1a2b3c4 initial setup
```

> ✅ Always use `git stash push -m "descriptive message"` so you know what each stash contains.

### Stash reflog expiry

Like the general reflog, stash entries can expire. The default is 90 days (`gc.reflogExpire`) for reachable entries and 30 days for unreachable ones. After that, `git gc` can prune them.

## 🔗 Further reading

- [Pro Git Book — Stashing and Cleaning](https://git-scm.com/book/en/v2/Git-Tools-Stashing-and-Cleaning)
- [Official Git man page: `git-stash`](https://git-scm.com/docs/git-stash)
- [Atlassian Git Tutorial — Git Stash](https://www.atlassian.com/git/tutorials/saving-changes/git-stash)
- [Git reflog (for recovery)](https://git-scm.com/docs/git-reflog)