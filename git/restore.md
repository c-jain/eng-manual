**Status:** 🌳 Evergreen  
**Created:** 2026-03-30  
**Last Updated:** 2026-03-30

<br>

# Restore

`git restore` is a focused Git command introduced in **Git 2.23 (August 2019)** for one purpose: **restoring file content** across Git's three areas — the working tree, the index (staging area), and the repository (commits/HEAD).
 
It was split out from `git checkout`, which previously handled branch switching, file restoration, and branch creation all in one command — causing persistent confusion.
 
> **Mental model:** `git restore` always moves content *backwards*: from a source (commit or index) into a target (index or working tree). `--staged` and `--worktree` sets tagert(s).

## The three-area model
 
```
Repository (HEAD/commits)
        │
        │  git restore --staged        (index ← HEAD)
        ▼
    Index (staging area)
        │
        │  git restore --worktree      (working tree ← index)
        ▼
  Working Tree (files on disk)
 
  git restore --staged --worktree      (working tree ← HEAD, via both)
```
 
Normal forward flow:
```
Working Tree  ──git add──▶  Index  ──git commit──▶  Repository
```
 
`git restore` reverses this flow, per flag.

## Why does it exist?

Before Git 2.23, `git checkout` served three unrelated roles:
 
```bash
git checkout main              # switch branch
git checkout -b feature        # create and switch branch
git checkout -- file.txt       # discard changes in a file
```
 
The `--` separator was easy to forget, and a typo in a branch name could silently restore files (or vice versa). Git 2.23 split this into:
 
| Command | Purpose |
|---|---|
| `git switch` | Branch switching |
| `git restore` | File content restoration |
| `git checkout` | Legacy (still works, now discouraged) |

## Key Commands & Usage

### Discard unstaged working tree changes
 
```bash
# Single file — restores from index
git restore file.txt
 
# All files
git restore .
 
# Specific directory
git restore src/
```
 
**⚠ Destructive.** Replaces working tree content with the index version. Cannot be undone. If the file hasn't been staged, the index holds the HEAD version.
 
---
 
### Unstage a file (safe)
 
```bash
# Single file
git restore --staged file.txt
 
# All staged files
git restore --staged .
```
 
Replaces the index entry with HEAD's version. **Working tree is untouched.** This is the safe complement to `git add` — no work is lost.

### Full undo — staged and working tree
 
```bash
git restore --staged --worktree file.txt
```
 
Resets both the index and working tree to HEAD. Equivalent to the old `git checkout HEAD -- file.txt`. **⚠ Destructive.**

### Restore from a specific commit or branch
 
```bash
# From a commit hash
git restore --source=abc1234 file.txt
 
# From HEAD explicitly
git restore --source=HEAD file.txt
 
# From a branch
git restore --source=main src/api.py
 
# From two commits back
git restore --source=HEAD~2 config.json
```
 
Replaces the working tree file with the version from the given source. **Does not auto-stage** — run `git add` afterwards if needed.

### Restore and stage in one step
 
```bash
git restore --source=HEAD~1 --staged --worktree file.txt
```
 
Restores from a source into both the index and working tree. Useful for "rolling back" a file to a prior version and having it ready to commit.

### Restore a deleted file
 
```bash
# Deleted but not staged
git restore deleted-file.txt
 
# Deleted and staged
git restore --staged --worktree deleted-file.txt
```
 
Git re-creates the file from the index or HEAD.

### Interactive (patch) mode
 
```bash
git restore -p file.txt
git restore --patch file.txt
```
 
Opens hunk-by-hunk interactive selector — identical UX to `git add -p`. Pick exactly which changes to discard.

## Comparison: restore vs reset vs checkout
 
| Goal | `git restore` (modern) | Old equivalent |
|---|---|---|
| Discard working tree changes | `git restore file.txt` | `git checkout -- file.txt` |
| Unstage a file | `git restore --staged file.txt` | `git reset HEAD file.txt` |
| Full undo (staged + working) | `git restore --staged --worktree file.txt` | `git checkout HEAD -- file.txt` |
| Restore from commit | `git restore --source=<ref> file.txt` | `git checkout <ref> -- file.txt` |
 
**Key distinction from `git reset`:**
- `git reset` moves the branch pointer (HEAD) — it operates at the commit level.
- `git restore` never moves HEAD — it only restores file content.

## What problems does it bring?

### 1. Working tree changes are gone forever
 
```bash
git restore file.txt  # ⚠ no undo, no reflog for this
```
 
Reflog only tracks commits. Uncommitted/unstaged changes discarded by `git restore` are permanently lost.
 
**Mitigation:** Stage anything you're unsure about before restoring. Or use `git stash` to temporarily save work.
 
---
 
### 2. Default source is the index, not HEAD
 
```bash
# These two are NOT equivalent if the file is staged:
git restore file.txt            # source = index (default)
git restore --source=HEAD file.txt  # source = HEAD (explicit)
```
 
If you have staged changes and unstaged edits on the same file:
- `git restore file.txt` — only discards the unstaged edits
- `git restore --source=HEAD file.txt` — overwrites with HEAD, losing staged changes too
 
---
 
### 3. `--staged` does not touch the working tree
 
```bash
git restore --staged file.txt
# Index is now HEAD's version, but working tree still has your edits
```
 
To undo both: `git restore --staged --worktree file.txt`.
 
---
 
### 4. `--source` without `--staged` doesn't stage the result
 
```bash
git restore --source=HEAD~2 file.txt
# File is now HEAD~2's content in working tree, but index is unchanged
# You still need: git add file.txt
```
 
---
 
### 5. Globbing and `.` gotchas
 
```bash
git restore .   # restores ALL files in working tree — double check before running
```
 
There is no "dry run" flag for `git restore`. Use `git status` and `git diff` first to understand what you'd be discarding.

## Safe workflow patterns
 
### Before discarding anything, check what you'd lose
 
```bash
git diff file.txt          # see unstaged changes
git diff --staged file.txt # see staged changes
git status                 # overview of both
```
 
### Stash first if unsure
 
```bash
git stash push file.txt    # save to stash
git restore file.txt       # now safe to discard
# If you need it back:
git stash pop
```
 
### Use patch mode for surgical restores
 
```bash
git restore -p file.txt    # pick individual hunks to discard
```

## 🔗 Further reading

- [Official git-restore docs](https://git-scm.com/docs/git-restore)
- [Pro Git Book — Recording Changes](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)