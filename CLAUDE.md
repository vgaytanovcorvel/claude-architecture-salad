# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Claude Architecture Salad** is a marketplace of Claude Code plugins (skills) for enterprise architecture. Each plugin provides opinionated scaffolding, rules, and automation for a specific architectural pattern.

## Repository Structure

```
plugins/
  <plugin-name>/
    skills/      # Claude Code skill definitions
    rules/       # Coding rules referenced by skills and generated CLAUDE.md files
    docs/        # Plugin documentation
```

Each plugin is self-contained under `plugins/<plugin-name>/`.

### Plugins

| Plugin | Path | Status | Description |
|--------|------|--------|-------------|
| **Clean Architecture** | `plugins/clean-architecture/` | First plugin | Scaffolds clean architecture projects, installs coding rules, and generates per-module CLAUDE.md files |

## Available Skills

### Clean Architecture

- `/bootstrap-clean-arch` — Scaffold clean architecture projects and generate per-module CLAUDE.md files. Reads modularization rules from the plugin's `rules/` folder to determine project structure and dependency boundaries.
- `/install-clean-arch-rules` — Install clean architecture coding rules into the target repo's `/rules` folder (common, csharp, typescript rule sets).

## Development Workflow

This is a skills/plugin repository — the primary artifacts are skill definitions and rule files, not compiled application code. There is no build step or test suite yet.

### Adding a New Plugin

1. Create a new directory under `plugins/<plugin-name>/` with `skills/`, `rules/`, and `docs/` subdirectories
2. Define skill files under `skills/`
3. Create corresponding rule files under `rules/`
4. Add documentation under `docs/`
5. Update this CLAUDE.md with the new plugin entry in the Plugins table and Available Skills section

## Conventions

- Main branch: `main`
- Plugin directories use kebab-case (e.g. `clean-architecture`)
- Skills follow the Claude Code skill format and are registered in `.claude/settings.json` or equivalent
- Rule files are plain text/markdown and are meant to be machine-readable by Claude Code
