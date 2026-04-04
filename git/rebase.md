**Status:** 🌳 Evergreen  
**Created:** 2026-04-04  
**Last Updated:** 2026-04-04

<br>

# Rebase

Rebasing is Git's mechanism for **replanting a sequence of commits onto a different base commit**. The name is literal: you're changing the *base* of your branch.

Unlike `git merge`, which records the fact that two branches converged, rebase rewrites history to make it appear as though your work always started from a later point.

```
Before rebase:                    After: git rebase main

A ← B ← C        (main)          A ← B ← C          (main)
    ↑                                     ↑
    D ← E    (feature)                   D' ← E'  (feature)
```

D and E are not moved — they are **replayed** as new commits (D′, E′) with new SHAs. The originals are abandoned but remain accessible via `git reflog`.

## Why Does It Exist?

`git merge` is non-destructive but leaves a trace of every integration in the log. On a busy team this produces a tangle of merge commits that document *process* rather than *intent*.

Rebase keeps history **linear and readable** — the question "what did this feature branch actually change?" has a clean answer.

| Dimension        | `git merge`                        | `git rebase`                        |
|------------------|------------------------------------|-------------------------------------|
| History shape    | Branching DAG                      | Linear                              |
| Records process  | Yes (merge commits)                | No                                  |
| Destructive      | No                                 | Yes (rewrites SHAs)                 |
| Safe on shared   | Always                             | Only on private branches            |
| Conflict model   | One resolution session             | One session per commit               |
| Best for         | Long-lived branches, team merges   | Cleanup before PR, syncing upstream |

## The Golden Rule

> **Never rebase commits that have already been pushed to a branch others are working on.**

When you rebase, you produce new commits (D′, E′). If a colleague already pulled the originals (D, E), their local repo diverges from origin. When they pull again, Git sees two different representations of the same work — resulting in duplicate commits and a messy forced merge.

**Safe to rebase:** your own private branches, before a PR is opened.  
**Unsafe to rebase:** `main`, `develop`, any shared feature branch others are tracking.

## Key Commands & Usage

### Basic rebase — integrate upstream changes

```bash
git checkout feature
git rebase main
```

Replants all feature commits on top of the current tip of `main`. Equivalent to pulling `main` in, without the merge commit.

### Interactive rebase — rewrite local history

```bash
git rebase -i HEAD~4    # rewrite the last 4 commits interactively
```

Git opens your editor with a list of commits and a verb in front of each:

```
pick a1b2c3d Add login form
pick e4f5a6b Fix typo in login form
pick c7d8e9f Add password validation
pick f0a1b2c WIP checkpoint
```

**Available verbs:**

| Verb      | Effect                                                        |
|-----------|---------------------------------------------------------------|
| `pick`    | Keep commit as-is                                             |
| `reword`  | Keep commit, edit its message                                 |
| `edit`    | Pause here to amend the commit (files + message)              |
| `squash`  | Fold into previous commit, combine both messages              |
| `fixup`   | Fold into previous commit, discard this commit's message      |
| `drop`    | Delete this commit entirely                                   |
| `exec`    | Run a shell command at this point in the replay               |

**Example — clean up before a PR:**

```
pick a1b2c3d Add login form
fixup e4f5a6b Fix typo in login form    ← folded into above
pick c7d8e9f Add password validation
drop f0a1b2c WIP checkpoint             ← discarded
```

Result: 2 clean commits, no WIP noise, typo fix invisible.

### Rebase onto a different base (`--onto`)

```bash
git rebase --onto <newbase> <oldbase> <branch>
```

Takes commits on `<branch>` that came *after* `<oldbase>` and plants them on `<newbase>`.

**Use case — wrong branch parent:**

```bash
# You branched feature-b off feature-a by mistake.
# Move feature-b's commits to start from main instead:
git rebase --onto main feature-a feature-b
```

### Handling conflicts during rebase

```bash
# After resolving conflicts in your editor:
git add <conflicted-file>
git rebase --continue

# Skip this commit entirely (dangerous — changes are lost):
git rebase --skip

# Abort everything, return to pre-rebase state:
git rebase --abort
```

### Force-push after rebasing a private branch

```bash
# --force-with-lease fails if someone else pushed since your last fetch
git push --force-with-lease origin feature
```

Always prefer `--force-with-lease` over `--force`.

### Configure pull to rebase by default

```bash
git pull --rebase origin main
# Or configure permanently:
git config --global pull.rebase true
```

With this set, `git pull` becomes `git pull --rebase` globally — no more "Merge remote-tracking branch" commits cluttering your log.

This is a common team convention: instead of git pull creating a "Merge remote-tracking branch" commit every time, you get a clean linear log.

## Worked Examples

### Example 1: Sync a feature branch with main

```bash
# You're on feature, main has moved forward
git fetch origin
git rebase origin/main

# If conflicts:
# 1. Fix files
# 2. git add <file>
# 3. git rebase --continue
# Repeat until done

git push --force-with-lease origin feature
```

### Example 2: Squash all feature commits before merging

```bash
# On feature branch, squash everything since branching from main
git rebase -i $(git merge-base HEAD main)

# In the editor: keep first as 'pick', change rest to 'fixup'
# Write one clear commit message
# Then open your PR with a single clean commit
```

### Example 3: Move a commit to a different branch

```bash
# You committed to main by mistake, need it on feature
git log --oneline       # note the SHA, e.g. abc1234

git checkout feature
git cherry-pick abc1234  # copy the commit to feature

git checkout main
git reset --hard HEAD~1  # remove it from main (private branch only)
```

## Rebase vs Merge: When to Use Which

| Situation                                          | Use          |
|----------------------------------------------------|--------------|
| Syncing a private feature branch with main         | `rebase`     |
| Cleaning up commits before opening a PR            | `rebase -i`  |
| Merging a finished feature into main               | `merge`      |
| Integrating a long-lived release branch            | `merge`      |
| Undoing a pushed commit on a shared branch         | `revert`     |
| Pulling latest changes (solo or team config)       | `pull --rebase` |

## What problems does it bring?

### 1. Conflict-per-commit, not conflict-per-merge
Rebase replays commits one at a time. If 3 of your 10 commits conflict with upstream, you resolve conflicts 3 separate times. Merge would give you one conflict session.

### 2. Interactive rebase `drop` can lose work
If you `drop` a commit in `git rebase -i` and don't notice, those changes are gone from the branch. Recoverable via `git reflog` within ~30 days.

### 3. `--force-with-lease` is safer than `--force`, but still dangerous
After rebasing a branch, you must force-push. `--force-with-lease` checks that no one pushed to the remote since your last fetch, reducing accidental overwrites — but it doesn't eliminate the risk entirely.

### 4. Re-merge trap with `git revert` + rebase
If a revert commit is in the shared history and you rebase past it, the revert may be reapplied, silently undoing your feature again. Always check the log after rebasing near revert commits.

### 5. Long-running branches accumulate rebase debt
The longer a branch diverges from `main`, the more conflicts accumulate. Either rebase frequently (every day or two) or accept a messier session at the end.

## 🔗 Further reading

- [Pro Git Book — Chapter 3.6: Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)  
- [Atlassian Git Tutorial: Merging vs Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)  
- [git-rebase man page](https://git-scm.com/docs/git-rebase)  
- [Oh Shit, Git!](https://ohshitgit.com/) — recovery recipes when rebase goes wrong