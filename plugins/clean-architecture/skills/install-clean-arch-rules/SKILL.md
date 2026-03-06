---
name: install-clean-arch-rules
description: Install clean architecture coding rules into the repository's /rules folder. Copies bundled rule files (common, csharp, typescript) to the repo root so other skills and CLAUDE.md files can reference them.
argument-hint: "[overwrite | skip]"
---

# Install Clean Architecture Rules

Deploy the bundled coding rule files to `<repo-root>/rules/`. This skill only copies rules — it does not generate CLAUDE.md files or scaffold projects.

## Step 1 — Locate Bundled Rules

Find the `rules/` subdirectory that is a sibling of this `SKILL.md` file (inside the skill's own directory). It contains:

```
rules/
├── common/
│   ├── coding-style.md
│   ├── logging.md
│   ├── patterns.md
│   ├── performance.md
│   ├── security.md
│   └── testing.md
├── csharp/
│   ├── backend.md
│   ├── coding-style.md
│   ├── modularization.md
│   ├── patterns.md
│   ├── security.md
│   └── testing.md
└── typescript/
    ├── angular.md
    ├── coding-style.md
    ├── patterns.md
    ├── security.md
    └── testing.md
```

## Step 2 — Check Target Location

Check whether a `rules/` directory already exists at the repository root (excluding `rules/skills/`).

- **If `$ARGUMENTS` is `overwrite`**: Replace all rule files in `<repo-root>/rules/` with the bundled versions. Preserve the `rules/skills/` directory — only overwrite `rules/common/`, `rules/csharp/`, and `rules/typescript/`.
- **If `$ARGUMENTS` is `skip`**: Keep existing rules as-is. Report what was skipped.
- **If `$ARGUMENTS` is empty or unrecognized and rules already exist**: Ask the user whether to **overwrite** or **skip**.
- **If no rules exist at the repo root**: Copy everything without prompting.

## Step 3 — Deploy Rules

1. Copy the entire bundled `rules/` tree (common/, csharp/, typescript/) to `<repo-root>/rules/`.
2. Do NOT touch `<repo-root>/rules/skills/` — that directory is managed separately.
3. Verify each file was written correctly by spot-checking at least one file per subdirectory.

## Step 4 — Report

Provide a summary:

| Directory | Files Deployed | Status |
|---|---|---|
| rules/common/ | (count) | Created / Overwritten / Skipped |
| rules/csharp/ | (count) | Created / Overwritten / Skipped |
| rules/typescript/ | (count) | Created / Overwritten / Skipped |

## Constraints

- **Do NOT modify any rule file contents** during deployment — copy them exactly as-is.
- **Do NOT delete `rules/skills/`** or anything inside it.
- **Do NOT generate CLAUDE.md files** — that is a separate concern handled by `/bootstrap-clean-arch`.
- **Do NOT scaffold project directories** — that is also a separate concern.
