**Status:** 🌳 Evergreen  
**Created:** 2026-03-23  
**Last Updated:** 2026-03-25

<br>

# Submodules

Let's say we are working on project A and need to use project B within project A. At the same time, we want to be able to treat the two projects as separate. In such scenario, Submodules let you clone another repository (project B) as a subdirectory into your project (project A) while keeping the commits separate.

## Why this existed (and still does)

1. **Vendoring with commit pinning.** You want to include a dependency at a specific commit, not just a version tag — and you want that choice tracked in your repo's history. Package managers (npm, pip) pin versions, but not always to exact commits.
2. **Monorepo alternatives.** Before monorepos became fashionable, many organizations split code into multiple repositories but still needed to compose them. Submodules were the composing glue.
3. **Shared components across teams.** If team A owns a shared library and teams B and C use it, submodules let B and C pin to whatever commit of A's repo they've tested against — and update deliberately, not automatically.
4. **Embedded firmware / platform code.** In embedded/systems work, you might include a vendor's SDK or an RTOS as a submodule at the exact commit your hardware was validated against. Drifting to a newer commit could break things. This use case is very much alive.
5. **C/C++ projects** that can't use a package manager (CMake's FetchContent is an alternative, but submodules are still widely used).
6. Simple cases of **including a small utility repo** that you don't want to publish to a registry.

## What problems submodules bring

1. **Clone complexity.** A fresh `git clone` of a repo with submodules gives you empty directories unless you also run `git submodule update --init --recursive`. New contributors regularly hit this.
2. **Detached HEAD everywhere.** When you `cd` into a submodule directory, you are in a detached HEAD state by default — not on any branch. Making changes there is easy to accidentally lose.
3. **Two-step updates.** To update a submodule to a newer commit, you `cd` into it, check out the new commit, then `cd` back to the parent and `git add` the changed pointer, then commit that. Two separate git operations in two places. Many people find this unintuitive.
4. **Pull doesn't cascade.** `git pull` on the parent does not automatically update submodules. You need `git pull --recurse-submodules` or a separate `git submodule update`. This trips people up constantly.
5. **Merge conflicts on the gitlink.** If two branches update a submodule to different commits, merging them produces a conflict on the pointer (the gitlink), which is a raw commit SHA in a conflict — not very human-readable.
6. **Tooling inconsistency.** Many GUI tools, CI systems, and hosting platforms have partial or inconsistent submodule support. GitHub's web UI renders them but some CI systems need explicit flags.

## Common commands

**Add Submodule**

```bash
git submodule add <repository-url> [<path>]
```

- **Notes:**
    
    - while adding a local repo as submodule in another local repo, git might give following error

        ```bash
        fatal: transport 'file' not allowed
        ```

        It happens because Git blocks `file://` protocol by default for **security reasons**. One possible solution is to allow file protocol by running following command:
        ```bash
        git config --global protocol.file.allow always
        ```

    - To verify whether Submodule pointer stored correctly and what exactly is committed, use following command:
    
        ```bash
        git ls-tree -r HEAD
        ```

        It shows what Git has stored in the latest commit.

**Update Submodule**

Let's say project B got some new changes, and we want to pull those changes inside project A.

Step 1: Inside project A, go inside submodule project B and run following command:

```bash
git pull origin main # or master
```

Step 2: Go back inside project A, stage changes and commit.

**Cloning a Project with Submodules**

When some clone project A, by default, with `git clone` command, they will only get empty directories containing submodules.

To populate the submodule directories, we must run two commands from the project A:

1. `git submodule init`: it will initialize/populate local configuration file (`.git/config`) with the content of `.gitmodules` file.

2. `git submodule update`: it will fetch all the data from project B and checkout the appropriate commit listed in project A.

    **Note:** We can also use `git submodule update --remote` command to get the latest commit from project B. Don't forget to stage and commit these changes else others won't get updated version.

**Note:** We can combine above two command into one `git submodule update --init` and to include nested submodules we can use `git submodule update --init --recursive` command.

To make it simple we can use `git clone --recurse-submodules` command. It will initialize and update each submodules including nested submodules.

**Remove Submodule**

Removing a submodule is tricky as it exists in multiple places.

**Step 1:** `git submodule deinit -f <path>` deinitialize submodule or remove local config.

**Step 2:** `git rm -f <path>` remove from index (staging). This removes submodule entry and updates `.gitmodules` automatically.

**Step 3:** `rm -rf .git/modules/<path>` delete leftover internal data.

**Step 4:** `git commit -m "remove submodule <name>"` persist changes.

**Step 5:** `git submodule status # should be empty` verify removal.

### 🔗 Further reading

- [Git Tools - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)