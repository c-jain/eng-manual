**Status:** 🌳 Evergreen  
**Created:** 2026-04-05  
**Last Updated:** 2026-04-05

<br>

# Cherry-Pick

`git cherry-pick` applies the diff introduced by a specific commit (or a range of commits) onto your current branch, creating a new commit. It does not bring over the entire branch history — only the changes from the selected commit(s).

```
Before:
  main:    A - B - C
  feature:         X - Y   ← want just Y

After: git cherry-pick Y
  main:    A - B - C - Y'  ← new commit, same diff as Y
  feature:         X - Y   ← untouched
```

`Y'` is a *copy*, not a reference. It has a different commit hash and a different parent than `Y`. The original `Y` on `feature` is unchanged.

## Why does it exist?

Merging and rebasing operate at the branch level. Cherry-pick operates at the **commit level**, giving you surgical precision when you need to move specific changes without bringing everything else along.

**Primary use cases:**

- **Hotfix backporting** — apply a critical fix from `main` onto `release/1.x` and `release/2.x` without merging unrelated work.
- **Rescuing a commit from the wrong branch** — pick it to the right branch, then reset/revert where it landed erroneously.
- **Selective staging** — your feature branch has 10 commits, but only 3 are ready for staging.
- **Atomic bug fixes shared across branches** — apply the same fix independently to multiple long-lived branches.

## How it works internally

1. Git computes the diff between the picked commit and its parent (i.e., "what this commit changed").
2. It applies that diff to your current `HEAD`, just like a patch.
3. If the patch applies cleanly, it creates a new commit with you as committer and the original author preserved.
4. If it conflicts, it pauses for you to resolve.

> The diff is applied relative to the picked commit's parent — **not** relative to your current branch. If your branch has diverged significantly from that parent, conflicts are likely.

## Key Commands & Usage

### Pick a single commit

```bash
git cherry-pick <commit-hash>
```

### Pick multiple specific commits

```bash
git cherry-pick abc1234 def5678 ghi9012
# Applied and committed in order — 3 new commits on your branch
```

### Pick a range of commits

```bash
# A..B = everything after A up to and including B (A excluded)
git cherry-pick A..B

# A^..B = everything from A up to and including B (A included)
git cherry-pick A^..B
```

### Stage only, don't auto-commit (`-n` / `--no-commit`)

```bash
git cherry-pick --no-commit abc1234
# Changes land in your index, no commit is created yet
# Useful to squash multiple cherry-picks into one commit
git commit -m "chore: backport fixes for release"
```

### Edit the commit message (`-e` / `--edit`)

```bash
git cherry-pick --edit abc1234
# Opens your editor before committing — add "Backport:" prefix etc.
```

### Cherry-pick from a remote branch

```bash
git fetch origin
git cherry-pick origin/feature~2   # third-from-tip commit on that remote branch
```

### Cherry-pick a merge commit (`-m`)

Merge commits have two parents. You must specify which parent is the "mainline" so Git knows which side's diff to replay:

```bash
git cherry-pick -m 1 <merge-commit-hash>
# -m 1 = parent 1 (the branch you merged *into*) is the mainline
# This is almost always the right choice
```

### Add a sign-off line

```bash
git cherry-pick --signoff abc1234
# Appends: Signed-off-by: Your Name <you@example.com>
```

## Handling conflicts

Cherry-pick pauses on conflicts the same way a merge does.

```bash
# 1. Resolve conflicts manually in the file(s)
# 2. Stage the resolved files
git add <resolved-files>

# 3. Continue
git cherry-pick --continue

# Alternatively — abort entirely (restores pre-cherry-pick state)
git cherry-pick --abort

# Skip the current conflicting commit and move to the next (range only)
git cherry-pick --skip
```

---

## Worked example: hotfix backport

```bash
# Fix a critical bug on main
git checkout main
# ... make fix ...
git commit -m "fix: sanitize user input in auth handler"

# Find the commit hash
git log --oneline -5
# a3f9c2e fix: sanitize user input in auth handler

# Backport to release/2.x
git checkout release/2.x
git cherry-pick a3f9c2e

# Backport to release/1.x
git checkout release/1.x
git cherry-pick a3f9c2e
```

Result — the same fix lives independently on three branches. Neither release branch receives unrelated commits from `main`.

```
main:        A - B - FIX - C - D
release/2.x: P - Q - FIX'
release/1.x: M - N - FIX''
```

## What problems does it bring?

### 1. Duplicate commits when you later merge

Since `Y'` is a new commit, merging the original branch back into `main` after you've cherry-picked `Y` results in both `Y` and `Y'` being in history. Git usually handles the duplicate diff as a no-op, but:

- It clutters history — two commits doing the same thing.
- If surrounding lines changed on either branch, Git may surface it as a conflict.

**Rule of thumb:** cherry-pick for one-way transfers (e.g., backports that never get merged back). Avoid it if you plan to merge the source branch into the target later.

### 2. Context dependency — the most dangerous pitfall

Commit `Y` may depend on changes introduced in `X`. Cherry-picking `Y` without `X` onto a branch that lacks `X` can give you code that compiles or even passes tests but behaves incorrectly at runtime. Git only applies the diff — it has no understanding of semantic dependencies between commits.

**Always audit the commit you're picking:** check its parent diff and understand what it assumes is already in the codebase.

### 3. Diverged history = more conflicts

Cherry-pick applies the diff relative to the picked commit's parent. If your target branch has diverged significantly from that parent, a simple 5-line change can conflict heavily because the surrounding context has shifted. In those cases, a full merge or rebase is usually less painful.

### 4. Messy git log over time

Overusing cherry-pick makes `git log --graph` a spiderweb. It is often a signal that your branching strategy has friction. If you find yourself cherry-picking frequently, consider: long-lived release branches, feature flags, or a proper merge strategy instead.

### 5. Author vs committer

Cherry-pick preserves the original `Author:` field. You become the `Committer:`. The original timestamp is preserved too. This can cause confusion in `git blame` and code review when the committer, author, and timestamp don't align with when the work actually landed on the branch.

## Cherry-pick vs alternatives

| Situation | Prefer |
|---|---|
| Move a single specific fix to another branch | `cherry-pick` |
| Bring an entire feature branch into main | `merge` |
| Rewrite your branch on top of main | `rebase` |
| Undo a commit already pushed to shared branch | `revert` |
| Picking is needed very frequently | Reconsider your branching strategy |

## 🔗 Further reading

- [Pro Git Book — Chapter 5.3: Rebasing and Cherry-picking](https://git-scm.com/book/en/v2/Distributed-Git-Maintaining-a-Project#_rebase_cherry_pick)
- [git-cherry-pick official man page](https://git-scm.com/docs/git-cherry-pick)
- [Atlassian Git Tutorial — git cherry-pick](https://www.atlassian.com/git/tutorials/cherry-pick)