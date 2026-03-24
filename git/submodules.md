**Status:** 🌿 Sapling  
**Created:** 2026-03-23  
**Last Updated:** 2026-03-24

<br>

# Submodules

Let's say we are working on project A and need to use project B within project A. At the same time, we want to be able to treat the two projects as separate. In such scenario, Submodules let you clone another repository (project B) as a subdirectory into your project (project A) while keeping the commits separate.

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

Let's say we made some changes in project B and want to pull those changes inside project A.

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

### 🔗 Initial Resources

-[Git Tools - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)