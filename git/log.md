**Status:** 🌳 Evergreen  
**Created:** 2026-04-06  
**Last Updated:** 2026-04-06

<br>

## Log

`git log` is Git's **commit history browser**. It walks the commit graph — starting from a given ref (default: `HEAD`) and following parent pointers — and prints information about each reachable commit.

Every Git commit stores:
- A **SHA-1 hash** (40-hex unique identifier)
- **Author** name, email, timestamp
- **Committer** name, email, timestamp (may differ from author, e.g. after a rebase)
- **Parent SHA(s)** (0 for the root commit, 1 for a regular commit, 2+ for a merge commit)
- A **tree SHA** pointing to the snapshot of tracked files
- A **commit message**

`git log` reads and presents this metadata, optionally combined with diff information.

### The Commit Graph (ASCII)

```
              HEAD (main)
                  │
                  ▼
  A ── B ── C ── D ── E        ← linear history on main
                  │
                  └── F ── G   ← feature branch
```

`git log` traverses this graph in **reverse-chronological order** (newest first) by default, following parent edges.

## Why Does It Exist?

| Need | How `git log` Addresses It |
|---|---|
| **Audit trail** | Every commit is a timestamped, author-attributed snapshot |
| **Debugging** | Pinpoint when a bug was introduced (feeds into `git bisect`) |
| **Code archaeology** | Understand *why* code was written by reading commit messages |
| **Release preparation** | Generate changelogs between two tags |
| **Team awareness** | See what teammates committed and when |
| **Revert/cherry-pick prep** | Identify the exact SHA to undo or transplant |

Without a history browser you'd have raw `.git/objects` files — binary pack files with no practical navigation. `git log` is the human-facing window into the object database.

## Key Commands & Usage

### Basic Usage

```bash
# Show full log from HEAD
git log

# Show last N commits
git log -n 5
git log -5        # shorthand

# Show a specific branch
git log feature/auth

# Show a specific commit
git log abc1234

# Show all branches and tags
git log --all
```

**Sample output (default format):**
```
commit 3f9a2c8d1e4b5f7a0c3d6e9f2a5b8c1d4e7f0a3
Author: Chakshu Jain <chakshu@example.com>
Date:   Fri Apr 03 10:30:00 2026 +0530

    feat: add JWT authentication middleware

    - Validates Bearer tokens on protected routes
    - Returns 401 with error code on failure
    - Logs auth events to structured logger
```

### Formatting Output

This is where `git log` becomes genuinely powerful. The `--format` / `--pretty` flag accepts many placeholders.

#### Built-in formats

```bash
git log --oneline                 # short SHA + subject line
git log --pretty=short            # adds Author
git log / git log --medium      # default (medium format)
git log --pretty=full             # adds Author + Committer
git log --pretty=fuller           # adds Author + Committer + both dates
git log --format=raw              # raw object format (tree, parent SHAs, metadata)
```

**`--oneline` output:**
```
3f9a2c8 feat: add JWT authentication middleware
a1b2c3d fix: correct null check in user service
9e8f7g6 chore: update dependencies
```

#### Custom `--format` / `--pretty=format:` placeholders

| Placeholder | Description |
|---|---|
| `%H` | Full commit SHA |
| `%h` | Abbreviated commit SHA |
| `%T` | Full tree SHA |
| `%an` | Author name |
| `%ae` | Author email |
| `%ad` | Author date (respects `--date=`) |
| `%ar` | Author date, relative ("2 hours ago") |
| `%cn` | Committer name |
| `%cd` | Committer date |
| `%s` | Subject (first line of message) |
| `%b` | Body (rest of message) |
| `%D` | Ref names (branch, tag) |
| `%Cred` | Switch color to red |
| `%Cgreen` | Switch color to green |
| `%Cblue` | Switch color to blue |
| `%Creset` | Reset color |
| `%n` | Newline |

```bash
# Compact colored log
git log --pretty=format:"%Cgreen%h%Creset %Cblue%an%Creset %ar — %s"

# Full audit log with date, author, SHA, subject
git log --pretty=format:"%ad | %h | %an | %s" --date=short

# JSON-ish output (useful for scripting)
git log --pretty=format:'{"sha":"%H","author":"%an","date":"%ad","msg":"%s"}' --date=iso
```

#### `--date` formats

```bash
git log --date=short        # 2026-04-03
git log --date=relative     # 2 hours ago
git log --date=iso          # 2026-04-03 10:30:00 +0530
git log --date=rfc           # Thu, 3 Apr 2026 10:30:00 +0530
git log --date=unix         # Unix timestamp: 1743659400
git log --date=format:"%d %b %Y"   # custom strftime: 03 Apr 2026
```

### Filtering Commits

#### By author / committer

```bash
git log --author="Chakshu"             # partial name match (regex)
git log --author="chakshu@example.com" # by email
git log --committer="GitHub"           # e.g. web-flow commits
```

#### By date

```bash
git log --since="2026-01-01"
git log --after="2 weeks ago"
git log --until="2026-03-31"
git log --before="yesterday"

# Range: commits in March 2026
git log --after="2026-02-28" --before="2026-04-01"
```

#### By commit message

```bash
git log --grep="fix"              # commits whose message contains "fix"
git log --grep="JIRA-123"         # find a specific ticket
git log --grep="fix" -i           # case-insensitive
git log --grep="auth" --grep="fix" --all-match  # must match BOTH
```

#### Limiting count

```bash
git log -n 10          # last 10 commits
git log --skip=20 -10  # commits 21–30
```

---

### Graph & Topology Views

```bash
# ASCII branch graph
git log --oneline --graph --all

# With decorations (branch/tag names)
git log --oneline --graph --decorate --all
```

#### Topology ordering options

```bash
git log --topo-order     # keep commits on same branch together (default for --graph)
git log --date-order     # strict chronological regardless of topology
git log --author-date-order  # use author date for ordering
git log --reverse        # oldest commit first
```

### Diff & Patch Output

```bash
# Show full diff (patch) for each commit
git log -p
git log --patch           # same

# Show only which files changed + summary stats
git log --stat

# Ultra-compact: changed files only, no stats
git log --name-only

# Files + whether Added/Modified/Deleted
git log --name-status

# Condensed stats (insertions/deletions per file)
git log --shortstat

# Show word-level diff instead of line-level
git log -p --word-diff
```

**`--stat` sample output:**
```
commit 3f9a2c8
Author: Chakshu Jain <chakshu@example.com>
Date:   Fri Apr 03 10:30:00 2026

    feat: add JWT authentication middleware

 src/middleware/auth.js   | 42 +++++++++++++++++++++
 src/routes/protected.js  |  8 ++--
 tests/auth.test.js       | 31 +++++++++++++++
 3 files changed, 79 insertions(+), 2 deletions(-)
```

### File & Path Filtering

```bash
# History of a specific file
git log -- path/to/file.js

# History of a directory
git log -- src/

# Commits that changed a specific function (requires --pickaxe or -L)
git log -S "functionName"         # commits that added/removed this string
git log -G "regex.*pattern"       # commits whose diff matches this regex

# Follow a file through renames
git log --follow -- src/utils/helpers.js

# Line-range history: who changed lines 10-25 of auth.js?
git log -L 10,25:src/middleware/auth.js
```

**`-L` (line log) is extremely powerful for code archaeology:**
```bash
git log -L :functionName:src/service.js   # history of an entire function by name
```

### Range & Ancestry Queries

Range syntax controls *which* commits are shown.

```
Notation         Meaning
──────────────────────────────────────────────────────────
A..B             Commits reachable from B but NOT from A
                 (i.e. commits added since A, up to B)

A...B            Symmetric difference: commits reachable
                 from A OR B, but NOT both
                 (useful for comparing branches)

^A B             Same as A..B (explicit exclusion)

HEAD~3..HEAD     Last 3 commits

origin/main..HEAD  Local commits not yet pushed
```

```bash
# What's on feature but not on main?
git log main..feature/auth

# What diverged between two branches?
git log main...feature/auth --left-right --oneline

# Commits in last 3 hops from HEAD
git log HEAD~3..HEAD --oneline

# Commits not yet pushed to origin
git log origin/main..HEAD

# All commits that touched this file in a range
git log v1.0..v2.0 -- src/auth.js
```

**`--left-right` with `...`:**
```
< 3f9a2c8 (main) fix: correct null check
< a1b2c3d (main) chore: update CI config
> 9e8f7g6 (feature) feat: add dashboard
> 4d5e6f7 (feature) feat: stub route
```
`<` = reachable from left ref (main), `>` = reachable from right ref (feature).

#### Ancestry operators

| Notation | Meaning |
|---|---|
| `HEAD^` | First parent of HEAD (same as `HEAD~1`) |
| `HEAD^2` | Second parent of HEAD (for merge commits) |
| `HEAD~3` | Three generations back, always first parent |
| `HEAD^^` | Two generations back (first parent each time) |

```bash
# For a merge commit, show both parents
git log HEAD^1 -1   # first parent (mainline)
git log HEAD^2 -1   # second parent (merged branch)
```

### Searching Commit Content

```bash
# -S ("pickaxe"): commits that added or removed this exact string
git log -S "SECRET_KEY" --all    # find when a secret was introduced

# -G: commits whose patch matches a regex
git log -G "password\s*=" --all -p

# --grep on message body
git log --grep="Fixes #" --oneline

# Search across all refs, including stash
git log --all --grep="hotfix"
```

**Pickaxe**
- `-S <string>` matches commits that *changed the count* of that string (added or removed it).
- `-G <regex>` matches commits where *any line in the diff* matches the regex (even if net count didn't change).

## Anatomy of a Commit Log Entry

```
commit 3f9a2c8d1e4b5f7a0c3d6e9f2a5b8c1d4e7f0a3   ← Full SHA-1
Merge: a1b2c3d 9e8f7g6                             ← Only for merge commits: parent SHAs
Author: Chakshu Jain <chakshu@example.com>          ← Who wrote the change
Date:   Fri Apr 03 10:30:00 2026 +0530             ← Author date + timezone

    feat: add JWT authentication middleware          ← Subject (72-char convention)
                                                     ← Blank line separates subject/body
    - Validates Bearer tokens on protected routes    ← Body: the "why"
    - Returns 401 with error code on failure
    - Logs auth events to structured logger
                                                     ← Blank line before trailers
    Reviewed-by: Alice <alice@example.com>           ← Git trailers (structured metadata)
    Fixes: #142
```

**Key fields explained:**

| Field | Notes |
|---|---|
| SHA-1 | Cryptographic hash of tree + parent(s) + author + message. Change any field → new SHA. |
| Merge: | Shows 2 (or more) parents for a merge commit. Absent for regular commits. |
| Author vs. Committer | After `rebase`, `cherry-pick`, or `am`, these differ. Always check both. |
| Timezone offset | `+0530` = IST. Stored in the commit object; affects sort order. |
| Subject | First line; shown in `--oneline`. Should be ≤72 chars, imperative mood. |
| Body | After blank line. The "why", not the "what" (the diff shows "what"). |
| Trailers | `Key: Value` pairs after a blank line. Parsed by tools like `git interpret-trailers`. |

## Revision Selection Cheat Sheet

```
Expression          What it means
─────────────────────────────────────────────────────────────
abc1234             Exact (abbreviated) SHA
HEAD                Current commit
HEAD~               Parent of HEAD
HEAD~3              3 hops back (first-parent chain)
HEAD^               Same as HEAD~
HEAD^2              Second parent (merge commits only)
main                Tip of main branch
origin/main         Remote-tracking branch tip
v1.2.0              Tag
@{yesterday}        Where HEAD was yesterday (uses reflog)
@{-1}               Previous branch (before last checkout)
stash@{0}           Top of stash stack
main@{2.weeks.ago}  Where main was 2 weeks ago (reflog)
A..B                Commits in B not in A
A...B               Symmetric difference of A and B
```

## Comparison Table: Common Output Formats

| Flag | Output Detail | Best For |
|---|---|---|
| *(default)* | SHA, author, date, full message | Reading carefully |
| `--oneline` | Short SHA + subject | Quick scan, scripts |
| `--short` | SHA + author + subject | Light review |
| `--stat` | Files changed + insertions/deletions | PR review prep |
| `-p` / `--patch` | Full unified diff | Debugging, archaeology |
| `--name-only` | File paths only | CI/CD changed-file detection |
| `--name-status` | File paths + A/M/D status | Rename detection |
| `--graph --oneline` | ASCII branch tree | Understanding topology |
| `--format=raw` | Raw Git object fields | Scripting, tooling |
| `-L :fn:file` | Line/function range history | Deep code archaeology |

## Real-World Workflows

### Generate a Changelog Between Tags

```bash
git log v1.0.0..v2.0.0 \
  --pretty=format:"- %s (%h)" \
  --no-merges \
  | sort
```

### Find Who Last Touched a File

```bash
git log --oneline --follow -1 -- src/auth/jwt.js
# For full blame-style, use git blame instead
```

### See What's Not Yet Pushed

```bash
git log origin/main..HEAD --oneline
```

### Find When a Bug Was Introduced (feeds into bisect)

```bash
# Find the commit that added "eval(" in any JS file
git log -S "eval(" --all -- "*.js" --oneline
```

### Audit Recent Activity Across All Branches

```bash
git log --all --since="1 week ago" \
  --pretty=format:"%Cgreen%h%Creset %Cblue%an%Creset %ar %s" \
  --no-merges
```

### Custom Alias (add to `~/.gitconfig`)

```ini
[alias]
  lg = log --oneline --graph --decorate --all
  ll = log --stat --abbrev-commit
  la = log --all --since="1 week ago" --pretty=format:"%h %an %ar %s" --no-merges
  lf = log --follow -p --   # usage: git lf src/file.js
```

### Commits That Changed a Specific Function

```bash
# Show full history of the `authenticate` function in auth.js
git log -L :authenticate:src/middleware/auth.js --no-patch
```

## Problems & Pitfalls

### Overwhelming Output on Large Repos
`git log` with no filters on a repo with thousands of commits is noise. Always filter by `--since`, `--author`, `-n`, or path.

### Confusing Author vs. Committer Dates
- **Author date**: when the patch was originally written.
- **Committer date**: when it was applied to the repo (e.g. after a rebase or am-patch).

`git log` shows **author date** by default. Use `--format="%cd"` or `--committer` variants when committer date matters (e.g. in CI logs).

```
# A rebased commit: author date stays, committer date changes
Author Date:   Mon Jan 13 10:00:00 2025
Committer Date: Fri Apr 03 09:15:00 2026   ← updated by rebase
```

### Merge Commits Obscure Linear History
In repos with lots of merges, the default output interleaves commits from many branches, making the log hard to read.
**Fix:** Use `--first-parent` to follow only the mainline, or `--no-merges` to hide merge commits entirely.

### SHA Truncation Collisions
`--abbrev-commit` shows short SHAs (default 7 chars). Rarely, these collide in large repos.
**Fix:** `--abbrev=12` for more characters, or avoid short SHAs in scripts.

### Rewriting History Changes SHAs
After a `rebase` or `commit --amend`, the SHA of a commit changes. Saved log output with old SHAs becomes stale — they still exist in the object store but are no longer reachable via branch refs.

### Sensitive Data in Commit Messages
`git log` exposes every commit message ever written. Never log passwords, tokens, or PII in commit messages — `git log` (and GitHub's UI) makes them permanently public.

### `--all` vs. Default Scope
By default, `git log` only shows commits reachable from `HEAD`. Commits on other local branches or tags are hidden. Use `--all` to see everything, `--remotes` for remote-tracking branches.

## 🔗 Further reading

- [Pro Git Book — Viewing Commit History](https://book.git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History)
- [Pro Git Book — Revision Selection](https://book.git-scm.com/book/en/v2/Git-Tools-Revision-Selection)
- [Official `git-log` man page](https://git-scm.com/docs/git-log)
- [Official `git-log` pretty formats](https://git-scm.com/docs/pretty-formats)
- [Atlassian Git Log Tutorial](https://www.atlassian.com/git/tutorials/git-log)
- [Git SCM — `--pickaxe` / `-S`](https://git-scm.com/docs/git-log#Documentation/git-log.txt--Sltstringgt)