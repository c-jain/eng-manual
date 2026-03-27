**Status:** 🌳 Evergreen  
**Created:** 2026-03-27  
**Last Updated:** 2026-03-27

<br>

# Revert

It undoes the changes introduced by a specific commit by creating a **new commit** that applies the inverse of the original. It never deletes history — it appends to it.

```
C1 → C2 (bug) → C3 → C4 (revert of C2)
                              ↑
                    New commit that undoes C2's changes.
                    C2 still exists in history.
```

## Why does it exist?

| Problem | Without revert |
|---|---|
| You pushed a bad commit | `git reset` rewrites history → breaks teammates' branches |
| Bad code is on `main` | You need to undo it without disrupting others |
| Audit trail matters | Revert preserves what happened and when |
 
`git revert` is **safe on shared/public branches** because history only moves forward.

## Key Commands & Usage
 
### Revert the latest commit
```bash
git revert HEAD
```
 
### Revert a specific commit
```bash
git revert a1b2c3d
```
 
### Revert without opening the editor
```bash
git revert --no-edit a1b2c3d
```
 
### Stage revert changes without committing (batch them)
```bash
# Stage the revert changes but don't commit yet — lets you amend the message or squash
git revert --no-commit a1b2c3d
# or shorthand:
git revert -n a1b2c3d
```
 
### Revert a range of commits (creates one revert commit per original)
```bash
# Revert commits from older..newer (exclusive on the left, inclusive on right)
git revert HEAD~3..HEAD
```
 
### Revert a range as a single commit
```bash
# Revert the same range but stage everything as one pending change
git revert -n HEAD~3..HEAD
git commit -m "revert: undo last 3 commits"
```
 
### Revert a merge commit
```bash
# -m 1 means "treat parent 1 (the main branch) as the mainline"
git revert -m 1 <merge-commit-hash>
```
The `-m` flag is mandatory for merge commits because Git needs to know which "side" of the merge to walk back to.
 
### Abort a revert in progress
```bash
git revert --abort
```
 
### Continue after resolving conflicts
```bash
# Fix conflicts → stage them → then:
git revert --continue
```
 
## git revert vs git reset
 
| | `git revert` | `git reset --hard` |
|---|---|---|
| History | Preserved, new commit added | Commits deleted |
| Safe on shared branch | ✅ Yes | ❌ No |
| Recoverable | Yes (commit still exists) | Harder (via reflog) |
| Creates conflicts | Possible | No |
| When to use | Pushed commits, shared branches | Local-only commits |
 
## What problems does it bring?
 
### 1. Merge conflicts
If later commits changed the same lines that the reverted commit introduced, Git can't cleanly apply the inverse diff. Resolve manually, then `git revert --continue`.
 
### 2. The re-merge trap ⚠️
You revert a feature branch merge. Later you fix the bugs and try to merge the branch again.
Git sees the feature commits as already present in history and **silently drops them**.
 
**Fix:**
```bash
# Step 1: revert the revert commit
git revert <hash-of-the-revert-commit>
 
# Step 2: now merge the fixed branch
git merge feature
```
 
### 3. Noisy history
Each `git revert` adds a commit. Over time this clutters `git log`.
Use `git log --oneline` or squash reverts with `-n` + a single commit.
 
### 4. Wrong tool for local-only commits
If the commit hasn't been pushed, `git reset` is cleaner and leaves no trace:
```bash
git reset --hard HEAD~1  # discard last local commit entirely
```
 
### 5. Reverting a merge commit requires `-m`
Without `-m`, Git doesn't know which parent to treat as the mainline and will error.
 
## 🔗 Further reading
 
- [Git official docs — git-revert](https://git-scm.com/docs/git-revert)