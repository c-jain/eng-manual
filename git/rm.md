**Status:** 🌳 Evergreen  
**Created:** 2026-04-07  
**Last Updated:** 2026-04-07

<br>

# RM

`git rm` removes files from **both the working tree (disk) and the index (staging area)** in one tracked operation. It is the staged-deletion counterpart to `git add`.

When you delete a file with the shell (`rm file.txt`), Git notices the absence but the deletion is **not staged**. You still have to run `git add -u` to record it. `git rm` collapses both steps atomically.

### The Three-Tree Model

Git maintains three trees. `git rm` operates on the **Index** and optionally the **Working Tree**:

```
┌──────────────────────────────────────────────────────────────────┐
│                        THREE-TREE MODEL                          │
│                                                                  │
│  ┌───────────┐        ┌───────────┐        ┌───────────────┐     │
│  │   HEAD    │        │   Index   │        │ Working Tree  │     │
│  │           │        │ (Staging) │        │   (Disk)      │     │
│  │ Last      │        │ Next      │        │ Files you     │     │
│  │ committed │        │ proposed  │        │ actually see  │     │
│  │ snapshot  │        │ commit    │        │ and edit      │     │
│  └───────────┘        └───────────┘        └───────────────┘     │
│                              ▲                    ▲              │
│                              │                    │              │
│                    git rm --cached           git rm              │
│                    (index only)          (index + disk)          │
└──────────────────────────────────────────────────────────────────┘
```

| Tree | Role |
|---|---|
| HEAD | Last committed snapshot |
| Index (staging area) | What the next commit will look like |
| Working tree | Files on disk |

## Why Does It Exist?

Removing a file has two separate concerns:

1. **Disk concern** — the bytes are gone from your filesystem (`rm`)
2. **Index concern** — Git's staging area no longer lists the file (`git add -u`)

Plain `rm` handles only concern #1. `git rm` handles both simultaneously and adds safety rails — it refuses to delete a file with staged, uncommitted changes unless you explicitly pass `--force`.

A second critical use case is **`git rm --cached`**: removing a file *from the index only*, leaving the disk copy intact. This is the standard fix for accidentally committed secrets or build artifacts.

## Key Commands & Usage

### Basic Remove — Disk + Index

```bash
git rm file.txt
```

Deletes `file.txt` from disk and stages the deletion. The file is gone from the repo on next commit.

```
Before:                          After git rm file.txt:
┌─────────────┐                  ┌─────────────┐
│   HEAD      │  file.txt ✓     │   HEAD      │  file.txt ✓  (unchanged)
├─────────────┤                  ├─────────────┤
│   Index     │  file.txt ✓     │   Index     │  file.txt ✗  (deletion staged)
├─────────────┤                  ├─────────────┤
│ Working tree│  file.txt ✓     │ Working tree│  file.txt ✗  (deleted from disk)
└─────────────┘                  └─────────────┘
```

### Remove From Index Only — The Un-tracking Pattern

```bash
git rm --cached file.txt
```

Removes the file from Git's tracking but **leaves it on disk**. After commit, Git treats it as untracked.

```
Before:                          After git rm --cached .env:
┌─────────────┐                  ┌─────────────┐
│   HEAD      │  .env ✓         │   HEAD      │  .env ✓  (still in history)
├─────────────┤                  ├─────────────┤
│   Index     │  .env ✓         │   Index     │  .env ✗  (untracked)
├─────────────┤                  ├─────────────┤
│ Working tree│  .env ✓         │ Working tree│  .env ✓  (file safe on disk)
└─────────────┘                  └─────────────┘
```

**Classic use case — accidentally committed `.env`:**

```bash
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "chore: stop tracking .env"
```

### Remove a Directory Recursively

```bash
git rm -r build/
git rm -r --cached node_modules/   # un-track only, keep on disk
```

Without `-r`, Git refuses with a fatal error.

### Force Removal

```bash
git rm -f file.txt
git rm --force file.txt
```

Bypasses the safety check that prevents deleting files with staged, uncommitted changes. Use with extreme caution — you can lose staged work.

### Dry Run — Preview Before Deleting

```bash
git rm --dry-run '*.log'
git rm -n '*.log'
```

Prints what *would* be deleted without touching anything. Always use with glob patterns before committing.

```
rm 'debug.log'
rm 'error.log'
```

### Remove Multiple Files With Globs

```bash
git rm '*.log'          # all .log files (Git expands the glob)
git rm src/*.tmp        # all .tmp under src/
```

> Wrap globs in single quotes so your shell doesn't expand them. Git handles its own glob expansion and matches across subdirectories.

### Bulk Un-track After Adding `.gitignore`

Use this when you've set up `.gitignore` retroactively on a repo that was committed without one:

```bash
# 1. Remove everything from the index (NOT from disk)
git rm -r --cached .

# 2. Re-add everything — .gitignore is now respected
git add .

# 3. Commit the cleaned state
git commit -m "chore: apply .gitignore to tracked files"
```

⚠️ This is a broad operation. Review `git status` carefully before committing.

### Operation Comparison Table

| Operation | Disk | Index | Notes |
|---|---|---|---|
| `rm file` (shell) | deleted | **unchanged** | Need `git add -u` to stage deletion |
| `git rm file` | deleted | deletion staged | Standard tracked remove |
| `git rm --cached file` | **unchanged** | deletion staged | Un-track without touching disk |
| `git rm -f file` | deleted | deletion staged | Forces past safety checks |
| `git rm -r dir/` | deleted recursively | deletion staged | Required for directories |
| `git rm -n '*.log'` | **unchanged** | **unchanged** | Dry run — preview only |

### `git rm` vs. Related Commands

| Command | Purpose |
|---|---|
| `git rm` | Remove from index + disk |
| `git rm --cached` | Remove from index only (un-track) |
| `git mv` | Rename/move (= `git rm` + `git add` under the hood) |
| `git restore --staged` | Un-stage a change (doesn't delete anything) |
| `git clean -fd` | Delete untracked files/dirs (not in index at all) |

## Problems & Pitfalls

### `git rm` Does Not Erase History

Removing a file only stops it appearing in *future* commits. Every past commit still contains the file. If you committed a secret, use `git filter-repo` to scrub history — `git rm` alone is insufficient.

```
past commits:  A──B──C (file.txt exists in all)
                        \
                         D  ← git rm file.txt + commit
                             (file gone going forward, but A/B/C unchanged)
```

### The `--cached` + `.gitignore` Two-Step Is Often Forgotten

`.gitignore` only prevents **untracked** files from being added. It has no effect on already-tracked files.

```bash
# ❌ WRONG — .env is still tracked even after this
echo ".env" >> .gitignore
git commit -m "ignore .env"

# ✅ CORRECT
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "chore: stop tracking .env"
```

### `--force` Bypasses Safety Checks

`git rm -f` deletes a file even if it has staged changes that haven't been committed. You can permanently lose work this way. Always do a `--dry-run` first if unsure.

### Directories Require `-r`

```bash
git rm build/       # ❌ fatal: not removing 'build/' recursively without -r
git rm -r build/    # ✅
```

### Removal Is Branch-Local

After `git rm` + commit on `feature`, the file still exists on `main` until that branch also removes it (or is merged).

### Safety Model at a Glance

```
Scenario                              git rm behavior
─────────────────────────────────────────────────────────────────
File unmodified                       ✅ removes cleanly
File has unstaged working-tree changes ✅ removes (warns you)
File has staged (indexed) changes     ❌ fails — need --force
File already deleted from disk        ✅ removes from index only
```

## 🔗 Further reading

- [Pro Git Book — 2.2 Git Basics: Recording Changes](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)
- [Git Official Docs — git-rm](https://git-scm.com/docs/git-rm)
- [Atlassian Git Tutorial — git rm](https://www.atlassian.com/git/tutorials/undoing-changes/git-rm)
- [GitHub Docs — Removing files from a repository](https://docs.github.com/en/repositories/working-with-files/managing-files/removing-files-from-a-repositorys-entire-history)