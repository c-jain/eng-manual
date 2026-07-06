# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`eng-manual` is a personal engineering knowledge base — a curated TIL log and reference of Markdown notes on DSA, system design (HLD/LLD), and Git. There is no application code, no build system, no linter, and no test suite. "Contributing" here means writing or editing Markdown notes, not shipping software.

## Repository structure

```
dsa/
├── ROADMAP.md            interview-prep roadmap; source of truth for what each note file should cover
├── fundamentals/          complexity analysis, math for DSA
├── data-structures/       arrays, strings, linked lists, trees, heaps, graphs, tries, etc.
├── algorithms/            named techniques: sorting, binary search, Dijkstra, DP, KMP, etc.
└── patterns/              how to frame a problem into a known shape: sliding window, two pointers, union-find, etc.

system-design/
├── ROADMAP.md            HLD + LLD interview-prep roadmap and module breakdown
├── hld/                   architecture-scale topics (caching, sharding, CAP, load balancing) and case studies (design-*.md)
└── lld/
    ├── patterns/{creational,structural,behavioral}/   GoF design patterns, one file per pattern
    └── design-*.md        LLD case studies (parking lot, etc.)

git/                       one topic per Git command/concept (rebase, worktree, cherry-pick, etc.)
web/                       misc web topics (e.g. atom feeds)
```

`dsa/ROADMAP.md` and `system-design/ROADMAP.md` are the canonical plans — they list every intended file with its expected content and study order. When adding a new note, check the relevant ROADMAP first: the filename, section, and scope are usually already specified there.

## Note conventions

Every note follows a consistent shape — match it when adding or editing files:

1. **Frontmatter header** at the very top of the file:
   ```
   ---
   Status: 🌱 Seed | 🌿 Sapling | 🌳 Evergreen
   Created: YYYY-MM-DD
   Last Updated: YYYY-MM-DD
   ---
   ```
   Update `Last Updated` whenever a note is meaningfully edited. Advance `Status` as a note matures (see maturity scale in README.md): 🌱 Seed (raw/WIP) → 🌿 Sapling (structured, verified examples) → 🌳 Evergreen (polished, long-term reference).

2. **Title + Table of Contents** — an H1 title followed by a "Table of Contents" (or "Table Of Contents") section linking to each H2.

3. **DSA problem entries** use a fixed pattern (see `dsa/ROADMAP.md` "Problem Format"): a LeetCode link with difficulty, two `<details>`-collapsed hints, then a `<details>`-collapsed Go solution with a short "Why this works" explanation. Discipline is deliberate — don't reveal the solution or skip straight to hints; the intended usage order is attempt → hint 1 → attempt → hint 2 → solution.

4. **Primary language is Go** for DSA code (JavaScript only where the idiom meaningfully differs); LLD pattern examples vary by file (Go, Java, Rust) — match the existing file's language, and check for language-suffixed siblings (e.g. `factory-method-go.md` vs `factory-method-java.md`, `iterator.md` vs `iterator-rust.md`) before assuming which language a topic uses.

5. **HLD case studies** (`hld/design-*.md`) and **LLD case studies** (`lld/design-*.md`) follow an interview-framework structure: Requirements → Estimation → High-Level Design → Deep Dives → Trade-offs (see `hld/design-rate-limiter.md` or `hld/interview-framework.md` for the canonical shape).

## Commit conventions

Follow Conventional Commits: `<type>(<scope>): <subject>`, imperative mood, one topic per commit.

- **Types:** `feat` (new topic/guide), `docs` (refine/clarify/typo), `refactor` (reorganize structure), `fix` (correct a broken snippet or wrong technical detail), `wip` (raw in-progress capture), `chore` (README/metadata maintenance).
- **Common scopes:** `dsa`, `hld`, `lld`, `git`, `rust`, `go`, `kafka`, `docker`, `linux`, `arch`, `meta`.
- Direct pushes to `main` are the norm for this repo — there's no PR/review flow.

Full details: `CONTRIBUTING.md`.
