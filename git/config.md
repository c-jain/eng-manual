**Status:** 🌳 Evergreen  
**Created:** 2026-04-06  
**Last Updated:** 2026-04-06

<br>

# Config

It is Git's built-in configuration system. It controls **everything** about how Git behaves — your identity, editor, merge strategy, color output, credential handling, aliases, hooks paths, and hundreds of other options.

Configuration is stored as plain `.ini`-style text files, read at runtime. There is no "global state" baked into the Git binary — all behavior is derived from these files at the time a command runs.

```
git config [--scope] [--type=<type>] <key> [<value>]
```

Conceptually, `git config` is two things at once:

| Role | Description |
|---|---|
| **Storage** | Key-value pairs persisted in `.ini` files on disk |
| **CLI tool** | `git config` command to read/write/list those pairs |

## Why does it Exist?

Git was designed to be a distributed, cross-platform tool used by millions of people with wildly different needs. A single hardcoded behavior is impossible. Config solves several real problems:

### Identity per commit
Every commit records an author name and email. Without config, Git wouldn't know who you are. The global `user.name` / `user.email` settings are the most fundamental config values — every serious Git workflow depends on them.

### Environment portability
Different operating systems handle line endings, file permissions, and credential storage differently. Config lets Git adapt — `core.autocrlf`, `core.fileMode`, `credential.helper` — without requiring different builds of Git.

### Personal preferences vs project standards
You may prefer `nvim` as your editor, but your team's repo may require signed commits and a specific commit-message format. The **three-scope model** (system → global → local) lets both coexist cleanly.

### Repo-level overrides
Open-source contributors often work across dozens of repos with different email addresses, different signing keys, and different remote conventions. Local config lets each repo carry its own settings without touching the global identity.

## The Three-Scope Model

This is the most important concept in Git config. Every setting lives in exactly one scope. When Git resolves a setting, it walks the scope chain from most-specific to least-specific and takes the **first match**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Config Resolution Order                      │
│                  (higher = wins, lower = fallback)              │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  LOCAL  ·  .git/config  ·  repo-specific                 │  │  ← Highest priority
│  │  e.g. user.email = work@company.com                      │  │
│  └────────────────────┬─────────────────────────────────────┘  │
│                       │ falls through if key not found          │
│  ┌────────────────────▼─────────────────────────────────────┐  │
│  │  GLOBAL  ·  ~/.gitconfig  ·  per-user                    │  │
│  │  e.g. user.email = personal@gmail.com                    │  │
│  └────────────────────┬─────────────────────────────────────┘  │
│                       │ falls through if key not found          │
│  ┌────────────────────▼─────────────────────────────────────┐  │
│  │  SYSTEM  ·  $(prefix)/etc/gitconfig  ·  machine-wide     │  │  ← Lowest priority
│  │  e.g. core.autocrlf = input                              │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Scope details

| Scope | Flag | File location (Linux/macOS) | File location (Windows) | Who sets it |
|---|---|---|---|---|
| **system** | `--system` | `/etc/gitconfig` or `$(git --exec-path)/../../etc/gitconfig` | `C:\Program Files\Git\etc\gitconfig` | Admin / installer |
| **global** | `--global` | `~/.gitconfig` or `~/.config/git/config` | `C:\Users\<You>\.gitconfig` | You, once |
| **local** | `--local` _(default)_ | `<repo>/.git/config` | Same | You, per repo |
| **worktree** | `--worktree` | `<repo>/.git/worktrees/<id>/config.worktree` | Same | You, per worktree |

> **Rule of thumb:** Set identity + personal preferences globally. Override per-repo with local. Never commit `.git/config` (it's in `.gitignore` by default). Never touch system config unless you're the machine's admin.

## Key Commands & Usage

### Reading config

```bash
# Read a single key (searches all scopes, local wins)
git config user.name

# Read from a specific scope
git config --global user.name
git config --local user.email

# List ALL config with file origins
git config --list --show-origin

# List all config with scope labels
git config --list --show-scope

# List only global config
git config --global --list

# Check where a specific key comes from
git config --show-origin user.email
```

### Writing config

```bash
# Set in local scope (default when inside a repo)
git config user.email "chakshu@corp.com"

# Set globally
git config --global user.name "Chakshu Jain"
git config --global core.editor "code --wait"

# Set system-wide (needs sudo on Linux/macOS)
sudo git config --system core.autocrlf input
```

### Unsetting / removing config

```bash
# Remove a key
git config --unset user.email

# Remove a key from global scope
git config --global --unset core.editor

# Remove an entire section
git config --remove-section alias

# Remove an entire section from global
git config --global --remove-section alias
```

### Editing config directly

```bash
# Open the local .git/config in your $EDITOR
git config --edit

# Open ~/.gitconfig
git config --global --edit

# Open /etc/gitconfig
git config --system --edit
```

### Type coercion

Git can interpret config values as typed — useful for scripting:

```bash
# Store a boolean
git config --global --type=bool core.bare

# Store an integer
git config --global --type=int pack.threads 4

# Store a color
git config --global --type=color color.status.changed "yellow bold"

# Store a path (expands ~ and environment variables)
git config --global --type=path core.hooksPath "~/.config/git/hooks"
```

### Multi-value keys

Some keys can have multiple values (e.g., `remote.origin.fetch`):

```bash
# Add a value (doesn't replace existing)
git config --add remote.origin.fetch "+refs/pull/*/head:refs/remotes/origin/pr/*"

# Get all values for a key
git config --get-all remote.origin.fetch

# Replace all values for a key
git config --replace-all remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
```

## Config File Anatomy

Git config files use an `.ini`-like format. Understanding the raw file format helps when you need to hand-edit or script against it.

```ini
# ~/.gitconfig — example annotated file

# ── Section: core settings ──────────────────────────────────────
[core]
    editor = code --wait          # editor for commit messages, rebase, etc.
    autocrlf = input              # normalize line endings on commit
    excludesFile = ~/.gitignore   # global gitignore
    hooksPath = ~/.config/git/hooks  # global hooks directory

# ── Section: user identity ──────────────────────────────────────
[user]
    name = Chakshu Jain
    email = chakshu@personal.dev
    signingKey = ABCDEF1234567890   # GPG key fingerprint

# ── Section: commit signing ──────────────────────────────────────
[commit]
    gpgSign = true                # sign all commits by default

# ── Section: aliases ────────────────────────────────────────────
[alias]
    st = status
    co = checkout
    br = branch
    lg = log --oneline --graph --decorate --all
    undo = reset --soft HEAD~1
    wip = !git add -A && git commit -m "WIP"
    unstage = restore --staged

# ── Section: pull behavior ──────────────────────────────────────
[pull]
    rebase = true                 # rebase instead of merge on pull

# ── Section: push behavior ──────────────────────────────────────
[push]
    default = current             # push to same-name branch on remote
    autoSetupRemote = true        # auto set upstream on first push

# ── Section: diff & merge tools ─────────────────────────────────
[diff]
    tool = vscode
    colorMoved = default          # color moved lines differently

[difftool "vscode"]
    cmd = code --wait --diff $LOCAL $REMOTE

[merge]
    tool = vscode
    conflictstyle = diff3         # show base + both sides in conflict markers

[mergetool "vscode"]
    cmd = code --wait $MERGED

# ── Section: colors ─────────────────────────────────────────────
[color]
    ui = auto

[color "status"]
    added = green bold
    changed = yellow bold
    untracked = red bold

# ── Section: remote defaults ────────────────────────────────────
[remote "origin"]
    url = git@github.com:c-jain/eng-manual.git
    fetch = +refs/heads/*:refs/remotes/origin/*

# ── Section: conditional includes (see §8.1) ────────────────────
[includeIf "gitdir:~/work/"]
    path = ~/.config/git/work.gitconfig
```

### Key format rules

| Rule | Example |
|---|---|
| Section header in `[brackets]` | `[user]` |
| Subsection with quoted name | `[remote "origin"]` |
| Keys are lowercase, hyphen-separated | `signing-key` |
| Values are after `=`, optional whitespace | `name = Chakshu` |
| Comments start with `#` or `;` | `# this is a comment` |
| Boolean: `true/false`, `yes/no`, `on/off`, `1/0` | `gpgSign = true` |

## Essential Settings Cheatsheet

A minimal but production-ready global config setup:

```bash
# ── Identity ────────────────────────────────────────────────────
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# ── Editor ──────────────────────────────────────────────────────
git config --global core.editor "code --wait"         # VS Code
# git config --global core.editor "nvim"              # Neovim
# git config --global core.editor "nano"              # Nano

# ── Default branch name ─────────────────────────────────────────
git config --global init.defaultBranch main

# ── Pull behavior ───────────────────────────────────────────────
git config --global pull.rebase true

# ── Push behavior ───────────────────────────────────────────────
git config --global push.default current
git config --global push.autoSetupRemote true

# ── Line endings ────────────────────────────────────────────────
git config --global core.autocrlf input              # macOS/Linux
# git config --global core.autocrlf true             # Windows

# ── Global gitignore ────────────────────────────────────────────
git config --global core.excludesFile ~/.gitignore
# Then create ~/.gitignore with: .DS_Store, *.swp, .env, etc.

# ── Credentials ─────────────────────────────────────────────────
git config --global credential.helper osxkeychain   # macOS
# git config --global credential.helper manager     # Windows

# ── Diff improvements ───────────────────────────────────────────
git config --global diff.colorMoved default
git config --global merge.conflictstyle diff3

# ── Useful aliases ──────────────────────────────────────────────
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.unstage "restore --staged"
```

## Advanced Patterns

### `includeIf` — Multiple identities on one machine

The single most powerful config feature for developers who work across personal and professional repos.

```
┌──────────────────────────────────────────────────────────────────┐
│                   includeIf Directory Layout                     │
│                                                                  │
│  ~/                                                              │
│  ├── .gitconfig              ← global base config               │
│  │     [user]                                                    │
│  │       name = Chakshu Jain                                     │
│  │       email = chakshu@personal.dev                           │
│  │     [includeIf "gitdir:~/work/"]                              │
│  │       path = ~/.config/git/work.gitconfig                    │
│  │     [includeIf "gitdir:~/oss/"]                               │
│  │       path = ~/.config/git/oss.gitconfig                     │
│  │                                                              │
│  ├── work/                   ← all work repos live here         │
│  │   └── api-service/        → uses work.gitconfig              │
│  │                                                              │
│  └── oss/                    ← all open source repos            │
│      └── react-fork/         → uses oss.gitconfig               │
└──────────────────────────────────────────────────────────────────┘
```

```ini
# ~/.config/git/work.gitconfig
[user]
    email = chakshu@company.com
    signingKey = WORKKEYFINGERPRINT
[commit]
    gpgSign = true
```

```ini
# ~/.config/git/oss.gitconfig
[user]
    email = chakshu@personal.dev
    signingKey = OSSKEYFINGERPRINT
```

```bash
# Verify: which email will be used in ~/work/api-service?
cd ~/work/api-service
git config user.email
# → chakshu@company.com ✓
```

> **Note:** The `gitdir:` path in `includeIf` must end with `/` to match all repos under that directory. The trailing slash is mandatory.

### Aliases — Shell commands with `!`

Git aliases starting with `!` execute as shell commands, not Git subcommands:

```bash
# Shell alias: interactive add + commit in one step
git config --global alias.ac '!git add -A && git commit'

# Usage
git ac -m "feat: add login form"

# List all current aliases
git config --global --get-regexp alias
```

**More useful shell aliases:**
```ini
[alias]
    # Pretty log
    lg = log --oneline --graph --decorate --all --color

    # Show files changed in last commit
    last = diff HEAD~1 HEAD --name-only

    # Undo last commit keeping changes staged
    undo = reset --soft HEAD~1

    # Delete all local branches that were merged to main
    sweep = !git branch --merged main | grep -v '^\* main' | xargs git branch -d

    # Open repo in browser (macOS)
    browse = !open $(git remote get-url origin | sed 's/git@github.com:/https:\\/\\/github.com\\//')
```

### Global hooks with `core.hooksPath`

Instead of per-repo hooks, point Git at a global hooks directory:

```bash
mkdir -p ~/.config/git/hooks
git config --global core.hooksPath ~/.config/git/hooks

# Example: global commit-msg hook enforcing Conventional Commits
cat > ~/.config/git/hooks/commit-msg << 'EOF'
#!/bin/sh
PATTERN="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,72}"
if ! grep -qE "$PATTERN" "$1"; then
  echo "ERROR: Commit message must follow Conventional Commits."
  echo "  Format: <type>(<scope>): <subject>"
  exit 1
fi
EOF
chmod +x ~/.config/git/hooks/commit-msg
```

> **Note:** Per-repo `core.hooksPath` in `.git/config` overrides the global setting. This is the intended hierarchy.

### `url` rewriting — Force SSH over HTTPS

Useful when you have SSH keys set up and want to avoid HTTPS authentication prompts:

```ini
[url "git@github.com:"]
    insteadOf = https://github.com/
```

```bash
# Now `git clone https://github.com/user/repo.git`
# automatically becomes `git clone git@github.com:user/repo.git`
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

### Rerere — Reuse Recorded Resolution

`rerere` (Reuse Recorded Resolution) remembers how you resolved a merge conflict and reapplies the fix automatically next time the same conflict appears:

```bash
git config --global rerere.enabled true
```

Useful when you rebase long-running feature branches frequently against a fast-moving main.

### `safe.directory` — Shared repos and containers

In Docker containers or when mounting repos owned by another user, Git refuses to operate for security reasons. Fix with:

```bash
# Allow a specific directory
git config --global --add safe.directory /app/repo

# Allow all directories (use only in controlled environments)
git config --global --add safe.directory "*"
```
## Problems & Pitfalls

### Wrong identity on commits ⚠️

**Problem:** You forget to set `user.email` locally in a work repo. All commits go out with your personal email.

**Fix:** Always set local config immediately after cloning a work repo, or use `includeIf`.

```bash
# After cloning a work repo
git clone git@github.com:corp/api-service.git
cd api-service
git config user.email "chakshu@corp.com"
git config user.name "Chakshu Jain"
```

**Amend if you caught it early:**
```bash
# Fix author on the most recent commit
git commit --amend --reset-author
```

### Line-ending chaos (`core.autocrlf`) ⚠️

**Problem:** Windows developers push CRLF line endings. Linux/macOS devs push LF. Diffs become noise; `git blame` breaks.

```
# Windows setting
core.autocrlf = true    # checkout CRLF, commit LF

# Linux/macOS setting
core.autocrlf = input   # checkout as-is, commit LF

# Team recommendation: use .gitattributes instead
# and set core.autocrlf = false everywhere
```

**Best practice:** Use a `.gitattributes` file in the repo root to enforce line endings at the repo level regardless of developer config:

```
# .gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.sh text eol=lf
```

### Typos silently create new keys

**Problem:** `git config --global usr.name "Chakshu"` creates `usr.name`, not `user.name`. Git won't warn you.

**Detection:**
```bash
git config --list --show-origin | grep usr
# Shows nothing for user.name, all commits are authorless
```

**Fix:** Always use `--list` to verify after setting important keys.

### Scope confusion — "Why isn't my setting taking effect?"

```bash
# You set it globally...
git config --global core.editor "code --wait"

# ...but local config overrides it silently
git config --local core.editor "vim"

# To debug: show where each key comes from
git config --list --show-origin
```

### Credentials stored in plaintext

On Linux, the default credential helper stores credentials in `~/.git-credentials` in **plaintext**.

```bash
# BAD (default on many Linux systems)
credential.helper=store   # writes plaintext to ~/.git-credentials

# BETTER: use OS keychain
# macOS
git config --global credential.helper osxkeychain
# Linux (requires libsecret)
git config --global credential.helper /usr/share/doc/git/contrib/credential/libsecret/git-credential-libsecret
# Windows
git config --global credential.helper manager   # Git Credential Manager
```

### Config included from untrusted paths

**Problem:** If you clone a repo that somehow writes a `.git/config` with a `[core] hooksPath` or `[include]` pointing to attacker-controlled scripts, it could execute code.

**Mitigation:** Git introduced `safe.directory` for this reason. Never run `git config --local` on a repo you don't trust.

## 🔗 Further reading

- [Pro Git Book — Git Configuration](https://book.git-scm.com/book/en/v2/Customizing-Git-Git-Configuration)
- [Official `git-config` man page](https://git-scm.com/docs/git-config)
- [Atlassian — git config](https://www.atlassian.com/git/tutorials/setting-up-a-repository/git-config)
- [GitHub — Configuring Git](https://docs.github.com/en/get-started/getting-started-with-git/setting-your-username-in-git)
- [includeIf docs](https://git-scm.com/docs/git-config#_includes)
- [Git Credential Storage](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage)