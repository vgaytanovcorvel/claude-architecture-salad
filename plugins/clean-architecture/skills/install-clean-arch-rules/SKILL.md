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
│   ├── command-line.md
│   ├── database.md
│   ├── logging.md
│   ├── patterns.md
│   ├── security.md
│   └── testing.md
├── csharp/
│   ├── coding-style.md
│   ├── command-line.md
│   ├── domain.md
│   ├── hosting.md
│   ├── modularization.md
│   ├── persistence.md
│   ├── presentation.md
│   ├── scaffolding.md
│   ├── security.md
│   ├── services.md
│   └── testing.md
└── typescript/
    ├── angular.md
    ├── coding-style.md
    ├── css.md
    ├── frontend-arch.md
    ├── patterns.md
    ├── react.md
    ├── security.md
    └── testing.md
```

## Step 2 — Check Target Location

Check whether a `rules/` directory already exists at the repository root (excluding `rules/skills/`).

- **If `$ARGUMENTS` is `overwrite`**: Replace all rule files in `<repo-root>/rules/` with the bundled versions. Preserve the `rules/skills/` directory — only overwrite `rules/common/`, `rules/csharp/`, and `rules/typescript/`.
- **If `$ARGUMENTS` is `skip`**: Keep existing rules as-is. Report what was skipped.
- **If `$ARGUMENTS` is empty or unrecognized and rules already exist**: Ask the user whether to **overwrite** or **skip**.
- **If no rules exist at the repo root**: Copy everything without prompting.

## Step 3 — Deploy Rules via Shell Copy

**CRITICAL: Use the Bash tool with a single OS copy command. Do NOT read files and re-write them one by one — that is slow and error-prone.**

Resolve two absolute paths first:
- `$SKILL_DIR` — the directory containing this SKILL.md file
- `$REPO_ROOT` — the root of the target repository

Then run the appropriate command for the detected OS:

### Detect OS and copy

```bash
# Detect OS
OS=$(uname -s 2>/dev/null || echo "Windows")
SKILL_DIR="<absolute path to this skill's directory>"
REPO_ROOT="<absolute path to target repo root>"
SRC="$SKILL_DIR/rules"
DST="$REPO_ROOT/rules"
```

**macOS / Linux — use `cp -r`:**

```bash
mkdir -p "$DST"
cp -rf "$SRC/common"     "$DST/"
cp -rf "$SRC/csharp"     "$DST/"
cp -rf "$SRC/typescript" "$DST/"
```

**Windows — use `robocopy` (preferred) or `cp -rf` via Git Bash:**

```powershell
# robocopy — /E recurse, /IS overwrite same-size, /IT overwrite tweaked, /NP /NJH /NJS suppress noise
robocopy "$SRC\common"     "$DST\common"     /E /IS /IT /NP /NJH /NJS
robocopy "$SRC\csharp"     "$DST\csharp"     /E /IS /IT /NP /NJH /NJS
robocopy "$SRC\typescript" "$DST\typescript" /E /IS /IT /NP /NJH /NJS
```

Or via Git Bash on Windows:

```bash
mkdir -p "$DST"
cp -rf "$SRC/common"     "$DST/"
cp -rf "$SRC/csharp"     "$DST/"
cp -rf "$SRC/typescript" "$DST/"
```

**Skip mode:** Do not run the copy. Report existing files as skipped.

After the copy command completes:
1. Do NOT touch `$REPO_ROOT/rules/skills/` — that directory is managed separately
2. Verify by running `ls "$DST/common" "$DST/csharp" "$DST/typescript"` (or `dir` on Windows) and confirming file counts match the manifest in Step 1

## Step 4 — Report

Provide a summary using the file counts returned by the verification listing:

| Directory | Files Deployed | Status |
|---|---|---|
| rules/common/ | (count) | Created / Overwritten / Skipped |
| rules/csharp/ | (count) | Created / Overwritten / Skipped |
| rules/typescript/ | (count) | Created / Overwritten / Skipped |

## Constraints

- **Use shell copy commands (Bash tool) — never read + re-write files individually.**
- **Do NOT modify any rule file contents** during deployment — copy them exactly as-is.
- **Do NOT delete `rules/skills/`** or anything inside it.
- **Do NOT generate CLAUDE.md files** — that is a separate concern handled by `/bootstrap-clean-arch`.
- **Do NOT scaffold project directories** — that is also a separate concern.
