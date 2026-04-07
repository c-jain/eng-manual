**Status:** 🌳 Evergreen  
**Created:** 2026-04-07  
**Last Updated:** 2026-04-07

<br>

## Branch

A **branch** in Git is a lightweight, movable pointer to a commit. That's it — at the filesystem level, a branch is a 41-byte file inside `.git/refs/heads/` containing a single commit SHA.

```
.git/
  refs/
    heads/
      main          ← contains "a1b2c3d4..."
      feature/auth  ← contains "e5f6a7b8..."
  HEAD              ← contains "ref: refs/heads/main"
```

When you make a new commit on a branch, Git:
1. Creates the commit object (tree + parent + metadata).
2. Moves the branch pointer forward to point at the new commit.
3. Does **nothing else** — no copying, no branching overhead.

This is fundamentally different from older VCS tools (SVN, CVS) where branching meant copying an entire directory tree. In Git, creating a branch is an **O(1) operation** — it costs the same whether your repo has 10 commits or 10 million.

## Why Does It Exist?

### The Core Problem It Solves

Without branches, all developers share one linear history. Every work-in-progress change is immediately visible to everyone, half-finished features can break the build, and you can't work on two things simultaneously without stepping on yourself.

Branches solve this by giving each line of work **an isolated context** — its own pointer into the commit graph. Changes in one branch don't affect any other until you explicitly merge or rebase them.

### What Branches Enable

| Capability | Without Branches | With Branches |
|---|---|---|
| Parallel feature development | Impossible cleanly | Natural — each feature is a branch |
| Hotfix on production while WIP exists | You must stash/shelve everything | Switch to `main`, branch off, fix, merge |
| Experimentation | Risky — pollutes shared history | Safe — throw away the branch if it fails |
| Code review workflows | Hard to isolate changes | PRs are branch diffs |
| Release management | Messy | Stable `main` + release branches |
| Bisecting regressions | Must manually revert | Branch preserves all history |

### Philosophy

Linus Torvalds designed Git branches to be so cheap that the right reaction to *any* uncertainty is to branch. The mantra: **branch early, branch often**.

## How Git Stores Branches Internally

Understanding internals removes the mystery from every branch command.

### The Four Object Types

Git's object store (`.git/objects/`) has four types:

```
blob     → file contents
tree     → directory listing (maps names → blobs/trees)
commit   → snapshot metadata (tree + parent(s) + author + message)
tag      → annotated tag (points to commit + extra metadata)
```

A branch pointer references a **commit** object. A commit references a **tree**, which recursively references **blobs**.

### What "Switching Branches" Does

`git switch feature-x` (or `git checkout feature-x`):
1. Reads `.git/refs/heads/feature-x` to find the target commit SHA.
2. Reads that commit's tree.
3. Updates your **working tree** files to match the tree.
4. Updates the **index** (staging area) to match the tree.
5. Writes `ref: refs/heads/feature-x` into `.git/HEAD`.

No files are "moved" — Git is diffing object trees and applying the delta to your working directory.

### Packed Refs

When you have many branches, Git consolidates them into `.git/packed-refs` for performance. The behavior is identical — it's just a storage optimization.

## The HEAD Pointer

`HEAD` is a special pointer that tells Git: **"which branch (or commit) am I currently on?"**

### Attached HEAD (normal state)

```
HEAD → refs/heads/main → commit C3
```

`HEAD` points to a branch name (a "symbolic ref"). When you commit, the branch moves forward, and `HEAD` follows it automatically.

### Detached HEAD

```
HEAD → commit C2  (directly, not via a branch)
```

This happens when you:
- `git checkout <commit-sha>`
- `git checkout v1.2.3` (a tag)
- During a rebase or bisect

In detached HEAD state, new commits are made but **no branch pointer moves**. If you switch branches without creating a branch first, those commits become unreachable (orphaned). Git will warn you and eventually garbage-collect them.

```bash
# Safe way to work from a detached HEAD
git checkout abc1234          # detached
git switch -c rescue-branch   # create a branch here — safe now
```

## Branch Diagrams

### Basic Branch Creation

```
Initial state: one branch (main), three commits

  C1 ← C2 ← C3
             ↑
            main
             ↑
            HEAD


After: git switch -c feature/login

  C1 ← C2 ← C3
             ↑
            main
             ↑
           feature/login
             ↑
            HEAD
```

Both `main` and `feature/login` point to the same commit. HEAD now tracks `feature/login`.

### Diverging Branches

```
After two commits on feature/login, one commit on main:

  C1 ← C2 ← C3 ← C4 ← C5
                       ↑
                      main

              C3 ← C6 ← C7
                         ↑
                    feature/login
                         ↑
                        HEAD
```

Wait — this is misleading as a flat diagram. The real structure shares history:

```
             C4 ← C5    ← main
            /
C1 ← C2 ← C3
            \
             C6 ← C7    ← feature/login  ← HEAD
```

C3 is the **merge base** — the common ancestor. This is what `git merge-base main feature/login` returns.

### Fast-Forward Merge

When the target branch hasn't diverged (no new commits since the branch point):

```
Before merge:

  C1 ← C2 ← C3             ← main
              \
               C4 ← C5     ← feature/login  ← HEAD


git switch main
git merge feature/login

After (fast-forward — just moves the pointer):

  C1 ← C2 ← C3 ← C4 ← C5 ← main ← HEAD
                       ↑
                   feature/login
```

No merge commit is created. History is linear.

### Three-Way Merge (True Merge)

```
Before:

  C1 ← C2 ← C3 ← C4 ← main
              \
               C5 ← C6 ← feature/login


git switch main
git merge feature/login

After:

  C1 ← C2 ← C3 ← C4 ← M7 ← main ← HEAD
              \       ↙
               C5 ← C6
                     ↑
                 feature/login
```

`M7` is the **merge commit** — it has two parents: `C4` and `C6`.

### Remote Tracking Branches

```
Local:                          Remote (origin):
  main  → C5                      main → C5
  origin/main → C5 (cached)       feature → C7

After git fetch:
  main  → C5 (unchanged)
  origin/main → C7 (updated)     ← mirrors remote exactly

After git merge origin/main (or git pull):
  main  → C7
```

`origin/main` is a **remote-tracking branch** — a read-only local copy of what the remote's `main` looked like at last fetch. You never commit to it directly.

## Key Commands & Usage

### Creating Branches

```bash
# Create a branch (don't switch to it)
git branch feature/auth

# Create and switch in one step (modern — preferred)
git switch -c feature/auth

# Create and switch (classic syntax)
git checkout -b feature/auth

# Create branch from a specific commit/tag
git switch -c hotfix/bug-123 v2.1.0
git switch -c debug-point abc1234

# Create a branch tracking a remote branch
git switch -c feature/auth origin/feature/auth
# shorthand (Git infers tracking if names match):
git switch feature/auth   # if origin/feature/auth exists, auto-tracks
```

### Switching Branches

```bash
# Switch to existing branch
git switch main
git checkout main          # classic equivalent

# Switch back to the previous branch (like `cd -`)
git switch -
git checkout -

# Switch and discard local changes (dangerous!)
git switch -f main

# Detach HEAD to a specific commit
git switch --detach abc1234
git checkout abc1234       # classic equivalent
```

### Listing Branches

```bash
# Local branches
git branch

# Remote-tracking branches
git branch -r

# All (local + remote)
git branch -a

# With last commit info
git branch -v

# With tracking info and ahead/behind counts
git branch -vv

# Branches merged into current branch (safe to delete)
git branch --merged

# Branches NOT yet merged (unsafe to delete with -d)
git branch --no-merged

# Filter by pattern
git branch --list 'feature/*'
```

### Renaming Branches

```bash
# Rename current branch
git branch -m new-name

# Rename a specific branch
git branch -m old-name new-name

# After renaming, update the remote:
git push origin --delete old-name
git push origin -u new-name
```

### Deleting Branches

```bash
# Safe delete (refuses if unmerged)
git branch -d feature/auth

# Force delete (bypasses merge check — be careful)
git branch -D feature/auth

# Delete a remote branch
git push origin --delete feature/auth
# shorthand:
git push origin :feature/auth
```

### Comparing Branches

```bash
# Commits in feature not in main (what will be merged)
git log main..feature/auth

# Commits in either but not both (symmetric difference)
git log main...feature/auth

# Files changed between branch tips
git diff main feature/auth

# Files changed (names only)
git diff --name-only main feature/auth

# Find the merge base (common ancestor)
git merge-base main feature/auth
```

### Branch Tracking

```bash
# Set upstream for current branch
git branch -u origin/main
git branch --set-upstream-to=origin/main

# See current tracking relationships
git branch -vv

# Push and set upstream in one command
git push -u origin feature/auth

# Pull using tracking config (no need to specify remote/branch)
git pull   # uses configured upstream
```

### Fetching, Pulling, Pushing

```bash
# Fetch all remotes (updates remote-tracking branches, doesn't touch local)
git fetch
git fetch origin

# Fetch and prune stale remote-tracking refs
git fetch --prune
git fetch -p

# Pull = fetch + merge (or rebase if configured)
git pull
git pull --rebase          # rebase instead of merge
git pull origin main       # explicit

# Push local branch to remote
git push origin feature/auth

# Push all local branches
git push --all origin

# Force push (rewriting remote history — use only on personal branches!)
git push --force-with-lease   # safer: fails if remote has new commits
git push --force              # dangerous: overwrites remote unconditionally
```

### Useful Inspection Commands

```bash
# Which branch am I on?
git branch --show-current
git rev-parse --abbrev-ref HEAD

# Full graph of all branches
git log --oneline --graph --decorate --all

# Find which branches contain a commit
git branch --contains abc1234
git branch -r --contains abc1234  # remote branches

# Find which branch a commit was introduced in
git log --oneline --all | grep abc1234

# Show the tip commit of a branch
git rev-parse feature/auth
git log -1 feature/auth
```

## Branching Workflows

Different teams use different conventions for organizing branches. Here are the three most common:

### GitHub Flow (Simple, Recommended for Most Teams)

```
main (always deployable)
  └── feature/add-login
  └── fix/crash-on-logout
  └── chore/update-deps
```

Rules:
1. `main` is always production-ready.
2. Any work → new branch from `main`.
3. Open a PR for review.
4. Merge to `main` → deploy immediately (or via CI/CD).
5. Delete the branch.

**Best for:** Teams with continuous deployment, small-to-medium projects.

### Git Flow (Complex, For Scheduled Releases)

```
main          ← production, tagged with versions
develop       ← integration branch
  └── feature/auth       (branches from develop)
  └── feature/payments   (branches from develop)
  └── release/2.1.0      (branches from develop, merges to main + develop)
  └── hotfix/crit-bug    (branches from main, merges to main + develop)
```

Key branches:
- `main` — only receives merges from `release/*` and `hotfix/*`
- `develop` — integration; `feature/*` branches from and merges to this
- `feature/*` — individual features; merge back to `develop`
- `release/*` — stabilization before release; merges to `main` and `develop`
- `hotfix/*` — emergency production fixes; merges to `main` and `develop`

**Best for:** Software with versioned releases (mobile apps, packaged software).

### Trunk-Based Development (TBD)

```
main (trunk)  ← everyone commits here (or via very short-lived branches)
  └── short-lived/feature-x  (lives < 2 days, then merged)
```

Rules:
1. Everyone integrates to `main` at least daily.
2. Feature flags hide incomplete features at runtime.
3. Branches (if used) are extremely short-lived.

**Best for:** Large engineering orgs with mature CI/CD and feature flagging (Google, Facebook style).

### Workflow Comparison

| Criterion | GitHub Flow | Git Flow | Trunk-Based |
|---|---|---|---|
| Release cadence | Continuous | Scheduled | Continuous |
| Branch complexity | Low | High | Minimal |
| Merge conflicts | Low | High | Very low |
| CI/CD dependency | High | Medium | Very high |
| Team size | Any | Medium–Large | Large |
| Feature flag need | Sometimes | Rare | Always |

## Remote Branches & Tracking

### How Remote Tracking Works

When you `git clone`, Git creates:
- A local branch `main` (checked out)
- A remote-tracking ref `origin/main` pointing to the same commit
- A tracking relationship: `main` upstream = `origin/main`

```
Local                Remote (origin)
------               ------
main → C5  ←── tracks ──── main → C5
origin/main → C5  (local mirror of remote main, updated on fetch)
```

### The Fetch/Pull/Push Cycle

```
1. git fetch
   Downloads new objects from remote.
   Moves origin/* refs to reflect remote state.
   Does NOT touch your local branches.

2. git merge origin/main  (or git pull = fetch + merge)
   Merges remote changes into your local branch.
   Now your local main matches remote.

3. git push
   Uploads your commits to remote.
   Moves remote's branch pointer forward.
   Fails if remote has diverged (someone else pushed).
```

### Tracking Relationships

```bash
# See what's tracked
git branch -vv
# output:
# * main           a1b2c3d [origin/main: ahead 2, behind 1] my commit
#   feature/auth   e5f6a7b [origin/feature/auth] add login form

# "ahead 2"  → you have 2 commits not yet on remote
# "behind 1" → remote has 1 commit you haven't pulled yet
```

### Diverged Branches

If both local and remote have new commits, `git pull` (merge) creates a merge commit. `git pull --rebase` replays your local commits on top of the remote's tip instead, keeping history linear.

```
Before pull:        After pull --rebase:
  R1 ← R2           R1 ← R2 ← L1' ← L2'
       ↑                             ↑
  R1 ← L1 ← L2      (L1, L2 replayed as L1', L2')
```

## Problems & Pitfalls

### Long-Lived Branches → Merge Hell

The longer a branch lives without syncing with `main`, the worse the eventual merge. Each day of divergence compounds the conflict surface.

**Symptom:** "I spent 3 days on this feature and now the merge is a nightmare."  
**Fix:** Merge or rebase `main` into your branch frequently (daily if possible).

### Detached HEAD Confusion

```bash
git checkout v2.0.0  # you're now in detached HEAD
# ... make some commits ...
git switch main      # WARNING: those commits are now orphaned
```

**Fix:** Always `git switch -c <new-branch>` before committing in detached HEAD.

### Deleting Branches With Unmerged Work

```bash
git branch -d feature/experiment
# error: The branch 'feature/experiment' is not fully merged.
```

`-d` (lowercase) is safe — it refuses to delete unmerged branches.  
`-D` (uppercase) is force-delete — you **will lose** commits if the branch isn't reachable from any other ref.

**Recovery:** If you force-deleted by accident, use `git reflog` to find the commit SHA and recreate the branch:
```bash
git reflog
# find the commit: abc1234 HEAD@{5}: commit: last commit on deleted branch
git switch -c recovered-branch abc1234
```

### Stale Remote-Tracking Branches

When teammates delete their branches on the remote, your local `origin/their-branch` refs stay until you prune them.

```bash
git branch -r        # shows stale remote refs
git fetch --prune    # removes tracking refs for deleted remotes
# or configure automatically:
git config --global fetch.prune true
```

### Branch Name Confusion (local vs. remote)

`main` ≠ `origin/main`. One is your local branch; the other is a cached view of the remote.

```bash
git log main          # local history
git log origin/main   # remote history as of last fetch
git log main..origin/main   # commits on remote not yet in local
```

### Pushing to Wrong Branch

```bash
git push origin feature-x:main  # pushes feature-x → remote's main
```

This silently rewrites the shared `main` if it's a fast-forward. Always double-check push refspecs. Use branch protection rules on GitHub to prevent force-pushes to `main`.

### Too Many Stale Local Branches

Over time, merged branches accumulate locally. They don't cost disk space (just 41 bytes each) but they create cognitive noise.

```bash
# Delete all local branches already merged into main
git branch --merged main | grep -v '^\*\|main' | xargs git branch -d
```

## 🔗 Further reading

- [Pro Git Book — Branches in a Nutshell](https://book.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
- [Pro Git Book — Branch Management](https://book.git-scm.com/book/en/v2/Git-Branching-Branch-Management)
- [Pro Git Book — Remote Branches](https://book.git-scm.com/book/en/v2/Git-Branching-Remote-Branches)
- [Pro Git Book — Branching Workflows](https://book.git-scm.com/book/en/v2/Git-Branching-Branching-Workflows)
- [Atlassian — Git Branch Tutorial](https://www.atlassian.com/git/tutorials/using-branches)
- [Atlassian — Comparing Workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)
- [Git Official Docs — git-branch](https://git-scm.com/docs/git-branch)
- [Git Official Docs — git-switch](https://git-scm.com/docs/git-switch)
- [Git Official Docs — git-checkout](https://git-scm.com/docs/git-checkout)
- [GitHub Flow Guide](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Trunk Based Development](https://trunkbaseddevelopment.com)
- [A Successful Git Branching Model (Git Flow)](https://nvie.com/posts/a-successful-git-branching-model/)