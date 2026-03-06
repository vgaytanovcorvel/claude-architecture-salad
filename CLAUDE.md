# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Claude Architecture Salad** is a marketplace of Claude Code plugins (skills) for enterprise architecture. Each plugin provides opinionated scaffolding, rules, and automation for a specific architectural pattern.

## Repository Structure

```
plugins/
  <plugin-name>/
    skills/
      <skill-name>/
        SKILL.md       # Skill definition with frontmatter (name, description, argument-hint)
        rules/         # Bundled rule files deployed by the skill (if applicable)
```

Each plugin is self-contained under `plugins/<plugin-name>/`. Rules are bundled inside the skill that deploys them (e.g., `install-clean-arch-rules/rules/`).

### Plugins

| Plugin | Path | Status | Description |
|--------|------|--------|-------------|
| **Clean Architecture** | `plugins/clean-architecture/` | First plugin | Scaffolds .NET clean architecture projects, installs coding rules, and generates per-module CLAUDE.md files |

## Available Skills

### Clean Architecture

- `/install-clean-arch-rules` — Deploy bundled coding rules (common, csharp, typescript) to target repo's `/rules` folder. Run this first.
- `/bootstrap-clean-arch` — Scaffold a .NET clean architecture solution and generate per-module CLAUDE.md files. Requires rules to be deployed first.

## Development Workflow

This is a skills/plugin repository — the primary artifacts are SKILL.md definitions and rule files, not compiled application code. There is no build step or test suite.

### Adding a New Plugin

1. Create `plugins/<plugin-name>/skills/<skill-name>/SKILL.md` for each skill
2. Bundle any rule files alongside the skill that deploys them
3. Update this CLAUDE.md with the new plugin entry in the Plugins table and Available Skills section
4. Update README.md with usage instructions

## Conventions

- Main branch: `main`
- Plugin directories use kebab-case (e.g. `clean-architecture`)
- Each skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`, `argument-hint`)
- Rule files are plain text/markdown, machine-readable by Claude Code
- Skills reference each other by their `/skill-name` invocation name
