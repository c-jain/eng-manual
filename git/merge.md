**Status:** 🌳 Evergreen  
**Created:** 2026-04-02  
**Last Updated:** 2026-04-02

<br>

# Merge

`git merge` integrates the history of one branch into another. It identifies three key commits — the **tip of your current branch**, the **tip of the branch being merged in**, and their **common ancestor** — and uses a three-way diff to combine the changes. The current (target) branch is updated; the source branch is never modified.

```
Before:           A -- B -- C          (main)
                       \
                        D -- E         (feature)

After merge:      A -- B -- C ------- M    (main, with merge commit M)
                       \             /
                        D -- E ------      (feature, unchanged)
```

## Why does it exist?

Branches allow parallel development; merge is how that parallel work comes back together. The three-way diff means Git can auto-combine changes that touched different parts of the codebase, only asking for human intervention when both branches touched the same lines. This design replaced fragile, lock-based workflows from older VCS systems.

## The two merge modes

### Fast-forward (FF)

When the target branch is a direct ancestor of the source — no divergence — Git simply moves the target branch pointer forward. **No merge commit is created.**

```
Before:           A -- B -- C      (main, pointer here)
                             \
                              D -- E   (feature)

After ff merge:   A -- B -- C -- D -- E   (main pointer moved forward)
```

Git chooses this automatically when possible.

### Three-way merge (true merge)

When both branches have diverged, Git computes a diff from the common ancestor to each tip, combines those diffs, and creates a **merge commit** with two parent pointers.

```
git merge feature   →   creates commit M(C, E)
```

## Key Commands & Usage

### Basic merge

```bash
# Step 1: switch to the target branch (the one that will receive the changes)
git checkout main          # or: git switch main

# Step 2: merge the source branch into it
git merge feature/login
```

### Fast-forward control

```bash
# Force a merge commit even when ff is possible (preserves branch topology in history)
git merge --no-ff feature/login

# Only allow ff; abort if a merge commit would be needed
git merge --ff-only feature/login
```

### Squash merge

Collapses all source branch commits into a single staged diff. No merge commit, no parent link.

```bash
git merge --squash feature/login
git commit -m "feat: add login flow"
```

> ⚠️ `git branch --merged` won't list the branch — Git doesn't know it was merged.

### Merge strategies (`-s`)

Git has pluggable merge strategies. You rarely need to specify these manually but knowing they exist matters:


```bash
git merge -s ort feature/login        # default (Git ≥ 2.33)
git merge -s recursive feature/login  # default (Git < 2.33)
git merge -s octopus b1 b2 b3         # merge 3+ branches at once
git merge -s ours feature/dead-code   # keep our version entirely
```

### Strategy options (`-X`)

```bash
git merge -X ours feature/login           # auto-resolve conflicts preferring our side
git merge -X theirs feature/login         # auto-resolve preferring their side
git merge -X ignore-all-space feature     # ignore whitespace differences
```

### Conflict resolution workflow

```bash
git merge feature/settings
# CONFLICT (content): Merge conflict in src/settings.js

git status                     # see conflicted files

# Open file — edit the conflict markers to desired state:
# <<<<<<< HEAD
# ... our version ...
# =======
# ... their version ...
# >>>>>>> feature/settings

git add src/settings.js        # mark resolved
git merge --continue           # complete the merge

# OR abort the whole merge attempt:
git merge --abort
```

Visual merge tool:

```bash
git mergetool
# Configure VS Code:
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

### Inspection commands

```bash
# Commits on feature not yet on main
git log main..feature

# Dry-run (inspect conflicts before committing)
git merge --no-commit --no-ff feature/login
git merge --abort               # back out

# Branches already merged into current
git branch --merged

# Branches not yet merged
git branch --no-merged

# Merge with a custom message
git merge feature/login -m "Merge feature/login: adds OAuth support"
```

## What problems does it bring?

### Merge conflicts

Occur when both branches modified the same region of the same file. Git can't decide which wins and inserts conflict markers. You must resolve manually, stage, and commit.

### History noise

Busy repos accumulate many merge commits, cluttering `git log`. Use `git log --oneline --graph` to visualise branching structure, or consider `--no-ff` consistently so merges are at least meaningful rather than accidental.

### The re-merge trap

If you `git revert` a merge commit and later try to merge that branch again, Git considers those commits already incorporated. The feature changes won't reappear. Fix: revert the revert (`git revert <revert-commit>`), *then* merge.

### Merging in the wrong direction

The branch you're **on** is the target. Always confirm with `git branch` before merging. Running `git merge main` while on `feature` pulls main into feature — the opposite of what you usually want.

### Squash merge and "phantom unmerged" branches

After a squash merge, `git branch --no-merged` will still list the feature branch as unmerged. Delete it manually after you're done.

---

## Merge vs rebase — decision table

| Situation | Merge | Rebase |
|---|---|---|
| Integrating a finished feature into `main` | ✅ | ✅ (linear history) |
| Bringing `main` updates into feature branch | ✅ safe | ✅ cleaner, rewrites history |
| Shared / public branch | ✅ always | ❌ never |
| Preserve exact history of how work happened | ✅ | ❌ |
| Clean linear `git log` | ❌ | ✅ |

## References

- [Pro Git Book — Ch. 3.2: Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [Pro Git Book — Ch. 7.8: Advanced Merging](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging)
- [git-merge man page](https://git-scm.com/docs/git-merge)
- [Atlassian — Git Merge](https://www.atlassian.com/git/tutorials/using-branches/git-merge)
- [Atlassian — Merge Strategy](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)

