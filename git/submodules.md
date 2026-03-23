**Status:** 🌱 Seed  
**Created:** 2026-03-23  
**Last Updated:** 2026-03-23

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



### 🔗 Initial Resources

-[Git Tools - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)