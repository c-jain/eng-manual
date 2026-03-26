**Status:** 🌳 Evergreen  
**Created:** 2026-03-26  
**Last Updated:** 2026-03-26

<br>

# Tagging

A tag in Git is a named reference that points to a specific commit — essentially a human-readable alias for a commit hash. Unlike branches, tags don't move; they stay permanently pinned to the commit they were created on.

Two types:
- **Lightweight**: just a pointer. `git tag v1.0.0`
- **Annotated**: full object with author, date, message. `git tag -a v1.0.0 -m "..."`
  → Prefer annotated tags for releases.

## Why does it exist?

1. **Release marking** — pinning `v1.0.0`, `v2.3.1` etc. to the exact commit that went to production. Any user or pipeline can check out that exact state anytime.
2. **Deployment triggers** — CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins) are commonly configured to trigger builds/deployments only when a tag is pushed. This makes releases intentional and controlled.
3. **Audit & compliance** — annotated tags record who created the tag and when, giving you a verifiable audit trail separate from branch history.
4. **Rollback reference** — if `v1.3.0` introduces a critical bug, you can instantly check out `v1.2.0` without digging through commit history.
5. **Semantic Versioning anchor** — tags are the mechanism that makes `git describe` and semantic versioning tooling work.

## What problems does it bring?

1. **Tags are not pushed automatically.** `git push` does not push tags. You must explicitly run `git push origin <tagname>` or `git push --tags`. Forgetting this means your local tags exist nowhere else — a silent footgun in team environments.
2. **Deleting and re-tagging causes confusion.** Tags are meant to be immutable. If you delete and re-create a tag at a different commit (a "re-tag"), anyone who already fetched the old tag won't automatically get the update. They'll have a stale pointer — this causes silent, hard-to-debug discrepancies.
3. **No protection by default.** Git doesn't prevent you from deleting a tag or force-pushing over one on a remote. If your team hasn't protected tags in GitHub/GitLab, a single command can destroy a release reference.
4. **Lightweight tags carry no context.** If you use lightweight tags heavily, `git log` and `git tag -v` give you nothing about why that tag was created or who did it.
5. **Tag flooding —** if your CI pipeline auto-creates tags on every commit (e.g. `build-20260325`), the tag list becomes noise, making it hard to find real release tags.
6. **Duplicate tags across forks —** when working with forks, `git fetch --tags` can pull in tags from the fork that conflict with tags in origin.

## Key Commands
| Goal                        | Command                                      |
|-----------------------------|----------------------------------------------|
| Annotated tag on HEAD       | `git tag -a v1.0.0 -m "message"`            |
| Lightweight tag             | `git tag v1.0.0`                             |
| Tag a past commit           | `git tag -a v1.0.0 <hash> -m "message"`     |
| List all tags               | `git tag` or `git tag -l "v1.*"`            |
| Inspect a tag               | `git show v1.0.0`                            |
| Push one tag                | `git push origin v1.0.0`                    |
| Push all tags               | `git push origin --tags`                    |
| Push commits + tags         | `git push origin --follow-tags`             |
| Check out a tag             | `git checkout v1.0.0`                       |
| Branch from a tag           | `git checkout -b hotfix v1.0.0`             |
| Delete local tag            | `git tag -d v1.0.0`                         |
| Delete remote tag           | `git push origin --delete v1.0.0`           |
| Auto-version from tags      | `git describe --tags --always`              |

## 🔗 Further reading

- [Git Basics - Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging)