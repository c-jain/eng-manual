**Status:** 🌳 Evergreen  
**Created:** 2026-03-27  
**Last Updated:** 2026-03-30

<br>

# Reset

It is a command that **moves the HEAD pointer** to a specified commit. Think of it as a way to "rewind" your repository's state.

It can affect **working directory** (files on your disk), **staging area** (index / `.git/index`) and **repository** (committed history).

3 modes:

| Mode              | HEAD moves? | Staging area                     | Working directory        | Safe?         |
|-------------------|------------|----------------------------------|--------------------------|---------------|
| `--soft`          | ✅ Yes      | Kept (changes staged)           | Kept                     | ✅ Safe       |
| `--mixed` (default)| ✅ Yes      | Cleared (changes unstaged)      | Kept                     | ✅ Safe       |
| `--hard`          | ✅ Yes      | Cleared                          | Cleared (deleted)        | ⚠️ Dangerous  |

## Why does it exist?

1. You staged the wrong files and want to unstage them
2. You committed too early and want to revise before pushing
3. You want to squash multiple messy commits into one clean one
4. You went down the wrong path entirely and want to throw away all local work

## Key Commands & Usage

- `git reset --soft <commit>`: Moves HEAD back but keeps all your changes staged. It's like uncommitting but keeping everything ready to re-commit.

    **Use case**: You committed too early and want to rewrite the commit message or add more files.

    ```bash
    # You made 3 commits but want to squash them into one
    git reset --soft HEAD~3
    # Now all 3 commits' changes are staged
    git commit -m "One clean commit"
    ```
- `git reset --mixed <commit>`**(default)**: Moves HEAD back and unstages your changes, but keeps the file edits in your working directory.

    **Use case**: You staged the wrong files, or you want to re-organize what goes into the next commit.

    ```bash
    # Unstage everything you just staged
    git reset HEAD

    # Undo last commit, keep files as modified (unstaged)
    git reset HEAD~1
    ```

- `git reset --hard <commit>`: Moves HEAD back and destroys all changes in the staging area and working directory. Files are reverted to exactly how they were at the target commit.

    **Use case**: You want to completely throw away your work and start fresh from a known state.


    ```bash
    # Throw away all local changes and go back to last commit
    git reset --hard HEAD

    # Throw away the last 2 commits AND all changes
    git reset --hard HEAD~2

    # Reset to a specific commit hash
    git reset --hard abc1234
    ```

    ⚠️ **Warning:** Uncommitted changes lost via --hard are **not in the trash** — they're gone unless you can find them via git reflog.

- `git reset` **on a single file**: You can reset just one file in the index without moving HEAD.

    ```bash
    # Unstage a specific file (keep the edit in working dir)
    git reset HEAD src/app.js

    # Reset a file to match a specific commit
    git reset abc1234 -- src/app.js
    ```

    This is equivalent to `git restore --staged <file>` in modern Git.

- `git reflog` **- your safety net**: tracks every movement of HEAD, including resets. If you accidentally do a `--hard` reset, you can often recover

    ```bash
    git reflog
    # Output:
    # abc1234 HEAD@{0}: reset: moving to HEAD~2
    # def5678 HEAD@{1}: commit: my important work  ← recover this!

    git reset --hard def5678   # restore to that commit
    ```

## What problems does it bring?

**Rewriting shared history is dangerous.** If you `git reset` commits that have already been pushed to a remote and others have pulled them, you create a divergent history. When you try to push:

```bash
git push origin main
# ERROR: Updates were rejected because the remote contains work
# you do not have locally.
```

You'd be forced to `git push --force`, which overwrites the remote's history — potentially destroying your teammates' work.

**Rules of thumb:**

- `git reset` is safe on **local-only** commits that no one else has pulled.
- Never reset commits that are on a **shared/public branch**,
- Use `git revert` instead when you need to undo a commit that's already been pushed — it adds a new commit rather than rewriting history.

## 🔗 Further reading

- [Git Tools - Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified)