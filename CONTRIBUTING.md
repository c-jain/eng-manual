# 📜 Contributing Guidelines

This repository follows professional engineering standards for version control to ensure the history remains clean, searchable, and meaningful.

## 🪴 Knowledge Growth (Maturity Scale)
Every note should include a **Frontmatter** block at the top to track its state:

1. **🌱 Seed**: Capture phase. Raw logs, error messages, or unformatted commands.
2. **🌿 Sapling**: Organization phase. Added context, structured headers, and verified snippets.
3. **🌳 Evergreen**: Authority phase. Polished, high-quality reference with "Best Practices."

### Frontmatter Template
Use the following block at the top of every new Markdown file:

**Status:** 🌱 Seed | 🌿 Sapling | 🌳 Evergreen  
**Topic:** Technology Name  
**Last Updated:** YYYY-MM-DD

## 🏗️ Version Control Standards
- **Direct Pushes:** All updates are pushed directly to the `main` branch.
- **Atomic Commits:** Each commit focuses on a specific topic or update to keep the history granular.
- **Imperative Mood:** Commit messages use the imperative mood (e.g., "Add" instead of "Added").

## 📝 Commit Message Convention
I follow the **Conventional Commits** standard using the format: `<type>(<scope>): <subject>`

### Types
- **feat**: Adding a new topic or a significant new guide.
- **docs**: Refining existing notes, fixing typos, or adding clarity.
- **refactor**: Reorganizing folder structures or splitting files.
- **fix**: Correcting a broken code snippet or an incorrect technical detail.
- **wip**: Raw insights captured during active work (Work In Progress).
- **chore**: Maintenance tasks like updating the README or metadata.

### Common Scopes
`rust`, `go`, `kafka`, `docker`, `git`, `linux`, `arch`, `meta`

### Examples
- `feat(rust): add module on ownership & borrowing`
- `docs(kafka): clarify consumer group rebalance triggers`
- `fix(git): correct the syntax for interactive rebase`
