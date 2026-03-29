**Status:** 🌳 Evergreen  
**Created:** 2026-03-29  
**Last Updated:** 2026-03-29

<br>

# Diff

It is  built-in comparison engine. It shows the **line-by-line differences** between two states of your repository — working directory vs. staging area, staging area vs. a commit, or any two commits/branches against each other.
 
Under the hood, it uses the **unified diff format** (the same format as the Unix `diff` command) to produce a **patch** — a structured representation of what was added, removed, or left unchanged.

## The three trees (mental model)
 
Git has three storage areas. Every `git diff` variant compares a pair of them:
 
```
┌─────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│  Working        │        │  Index           │        │  Commit          │
│  directory      │        │  (staging area)  │        │  history         │
│                 │        │                  │        │                  │
│  Files on disk  │──add──▶│  After git add,  │──cmt─▶│  Permanent       │
│  (unsaved/      │        │  before commit   │        │  snapshots       │
│   uncommitted)  │        │                  │        │  HEAD, SHAs      │
└─────────────────┘        └──────────────────┘        └──────────────────┘
        │                           │                           │
        │◀─────── git diff ───────▶│                           │
        │                           │◀─── git diff --staged ──▶│
        │                           │                           │
        │◀────────────────── git diff HEAD ───────────────────▶│
```
 
This is the most important mental model. Every command variant maps to one of these comparisons.

## Anatomy of a unified diff
 
```diff
diff --git a/main.js b/main.js        ← diff header (files being compared)
index 8a3f1c2..b7d4e90 100644         ← blob hashes + file mode (100644 = regular file)
--- a/main.js                         ← old file marker
+++ b/main.js                         ← new file marker
@@ -12,7 +12,8 @@ function greet(name) {   ← hunk header
   const msg = "Hello, " + name;     ← context line (unchanged, 3 lines shown by default)
-  return msg;                        ← removed line (was in old file)
+  console.log(msg);                  ← added line (new in new file)
+  return msg.trim();                 ← added line
   }                                  ← context line
```
 
### Reading the hunk header `@@ -12,7 +12,8 @@`
 
| Part | Meaning |
|------|---------|
| `-12,7` | Old file: starts at line 12, shows 7 lines |
| `+12,8` | New file: starts at line 12, shows 8 lines |
| `+1 line` | Net addition of 1 line in this hunk |
| Text after `@@` | Nearest enclosing function/class (context hint) |
 
### Line prefixes
 
| Prefix | Meaning |
|--------|---------|
| `-` | Removed (only in old file) |
| `+` | Added (only in new file) |
| ` ` (space) | Context — unchanged, shown for orientation |

## Why does it exist?

Before `git diff`, answering "what exactly changed?" required manual comparison or external tools. `git diff` solves several real problems:

- **Review before you commit** - The most dangerous moment in version control is committing code you didn't intend to change. `git diff` lets you inspect every line before it becomes part of history.
- **Understanding a codebase's evolution** - When debugging, the question *"what changed between when this worked and now?"* is answered by diffing two commits or tags.
- **Code review** - Pull Requests on GitHub/GitLab are fundamentally `git diff` output rendered nicely. Understanding the raw diff makes you a better code reviewer.
- **Generating patches** - The output of `git diff` is a valid patch file that can be emailed, applied on another machine, or stored as a changeset.

## Common commands
 
### Working directory vs index (unstaged changes)
 
```bash
git diff                     # All unstaged changes
git diff src/utils.js        # Unstaged changes in one file
git diff -w                  # Ignore all whitespace differences
git diff --stat              # Summary only (no line-by-line)
git diff --name-only         # Just filenames
git diff --name-status       # Filenames + M/A/D status
```
 
> **Gotcha:** After `git add`, a file disappears from `git diff`. It's now in the index. Use `git diff --staged` to see it.
 
---
 
### Index vs last commit (staged changes)
 
```bash
git diff --staged            # What's about to be committed
git diff --cached            # Identical to --staged (older alias)
git diff --staged --stat     # Summary of what's staged
git diff --staged src/api.js # Staged changes in one file
```
 
> Run this **before every `git commit`** to verify exactly what's going into the snapshot.
 
---
 
### Working directory vs last commit (all local changes)
 
```bash
git diff HEAD                # All changes: staged + unstaged vs last commit
git diff HEAD -- src/        # All local changes in a directory
```
 
---
 
### Comparing commits
 
```bash
# Two specific commits
git diff abc1234 def5678
 
# What changed IN a specific commit (vs its parent)
git diff abc1234^ abc1234
git show abc1234             # Equivalent shorthand
 
# N commits ago vs HEAD
git diff HEAD~3 HEAD
git diff HEAD~3              # Shorthand (HEAD is implicit)
 
# Specific file between commits
git diff HEAD~5 HEAD -- path/to/file.py
```
 
---
 
### Comparing branches
 
```bash
# Two-dot: diff of tips of both branches
git diff main..feature-branch
 
# Three-dot: changes on feature-branch SINCE it diverged from main
# (this is what pull requests use — ignores commits made to main since diverge)
git diff main...feature-branch
 
# Specific paths only
git diff main feature-branch -- src/api/
```
 
> **Two-dot vs three-dot:** Almost always use `...` (three-dot) for branch comparisons.
> Two-dot includes unrelated commits on `main` since the branch was created.
> Three-dot shows only "what did this branch add?"
 
```
main:     A──B──C──D
               \
feature:        E──F──G
 
git diff main..feature    → D vs G  (all differences between tips)
git diff main...feature   → B vs G  (only what feature added since diverging)
```
 
---
 
### Word-level diffs
 
```bash
# Changes shown inline, word by word (great for prose/docs)
git diff --word-diff
 
# Character-level highlighting within a line
git diff --color-words
```
 
Instead of replacing whole lines, shows: `[-old word-]` `{+new word+}` inline.
 
---
 
### Generating and applying patch files
 
```bash
# Save diff as a patch
git diff > my-changes.patch
 
# Apply a patch
git apply my-changes.patch
 
# Apply with 3-way merge fallback (safer, handles conflicts)
git apply --3way my-changes.patch
 
# Check if a patch applies cleanly without applying it
git apply --check my-changes.patch
```
 
---
 
### Diffing across tags and remotes
 
```bash
# Since a tag
git diff v1.2.0 HEAD
 
# Vs a remote branch (requires fetch first)
git fetch origin
git diff origin/main HEAD
```
 
---
 
### Filtering output
 
```bash
# Exclude a file
git diff -- . ':(exclude)package-lock.json'
 
# Only show files matching a pattern
git diff -- '*.ts'
 
# Show N lines of context instead of default 3
git diff -U5
```
 
---
 
## All flags reference
 
| Flag | Effect |
|------|--------|
| `--staged` / `--cached` | Index vs last commit |
| `--stat` | File-level summary: insertions, deletions |
| `--name-only` | Only filenames, no content |
| `--name-status` | Filenames + M/A/D status |
| `-U<n>` | Show `n` context lines (default: 3) |
| `-w` | Ignore all whitespace |
| `-b` / `--ignore-space-change` | Ignore whitespace amount |
| `--ignore-blank-lines` | Ignore blank-line-only changes |
| `--word-diff` | Inline word-level diff |
| `--color-words` | Inline character-level coloring |
| `-M` / `--find-renames` | Detect renamed files |
| `-C` / `--find-copies` | Detect copied files |
| `--no-color` | Plain text (for piping) |
| `--output=<file>` | Write output to file |
| `-p` / `--patch` | Force patch format |
| `--diff-algorithm=<algo>` | Use histogram/minimal/patience |
| `--binary` | Output binary patch |
 
---
 
## Configuring diff output
 
### Use a visual diff tool
 
```bash
# Launch configured difftool (e.g. VS Code, vimdiff, meld)
git difftool
 
# Set VS Code as the default
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'
```
 
### Diff algorithm
 
```bash
# patience: often produces more human-readable diffs for refactors
git diff --diff-algorithm=patience
 
# histogram: improved patience, good for code moves
git diff --diff-algorithm=histogram
 
# Set globally
git config --global diff.algorithm histogram
```
 
### Diffing non-text files
 
Configure textconv drivers in `.gitattributes`:
 
```
# .gitattributes
*.pdf diff=pdf
*.docx diff=docx
*.png diff=exif
```
 
Then register the converters:
 
```bash
git config diff.pdf.textconv pdftotext
git config diff.docx.textconv "docx2txt"
git config diff.exif.textconv "exiftool"
```

## Common gotchas
 
### 1. Staged vs unstaged confusion
After `git add`, `git diff` shows nothing for that file. The change moved to the index. Use `git diff --staged` to see it. Run both routinely:
 
```bash
git diff          # unstaged
git diff --staged # staged
```
 
### 2. Two-dot vs three-dot on branches
```bash
git diff main..feature    # tips of both branches (includes main's new commits)
git diff main...feature   # only what feature added since diverging (use this for PRs)
```
 
### 3. Renamed files appear as delete + add
Use `-M` to detect renames:
```bash
git diff -M           # show renamed files with similarity score
git diff -M90%        # only flag as rename if 90%+ similar
```
 
### 4. Untracked files are invisible
`git diff` never shows files that haven't been `git add`-ed at least once. Use `git status` to find them.
 
### 5. Lock files and generated code pollute diffs
```bash
git diff -- . ':(exclude)package-lock.json' ':(exclude)*.min.js'
```
Or suppress globally via `.gitattributes`:
```
package-lock.json -diff
dist/*.js -diff
```
 
### 6. Line-ending noise on cross-platform teams (CRLF vs LF)
```bash
git diff --ignore-cr-at-eol
git diff -w               # nuclear option: ignore all whitespace
```
 
### 7. `git diff` on a merge commit shows nothing useful
Merge commits have two parents. Use `git show -m <sha>` to see the diff against each parent.
 
---
 
## Practical workflows
 
### Pre-commit review ritual
```bash
git status              # What's modified?
git diff                # Review unstaged changes
git add -p              # Stage interactively (review each hunk)
git diff --staged       # Final check of what's about to be committed
git commit -m "..."
```
 
### Investigating a bug (bisect companion)
```bash
# What changed between the last working release and HEAD?
git diff v2.1.0 HEAD -- src/
 
# What changed IN the suspicious commit?
git show abc1234
 
# Find when a line was introduced
git log -S "suspiciousFunction" --diff-filter=A
```
 
### Pre-merge sanity check
```bash
# Fetch latest main
git fetch origin main
 
# See what your branch adds
git diff origin/main...HEAD
 
# See what you'd be merging IN
git diff HEAD...origin/main
```

## 🔗 Further reading

- [Official `git diff` documentation](https://git-scm.com/docs/git-diff)
- [Git Book — Recording Changes](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)
- [Git Book — Diff tools](https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration#_external_merge_tools)