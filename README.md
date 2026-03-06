# Claude Architecture Salad

A marketplace of [Claude Code](https://claude.ai/code) plugins for enterprise architecture. Each plugin provides opinionated skills and coding rules for a specific architectural pattern.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [Clean Architecture](plugins/clean-architecture/) | Scaffold .NET clean architecture solutions with dependency boundaries, coding rules, and per-module CLAUDE.md generation |

## Installation

1. Clone this repository
2. Open your target project in Claude Code
3. Run the skills from this repo to bootstrap your architecture

### Clean Architecture — Quick Start

```
/install-clean-arch-rules          # Deploy coding rules to your repo
/bootstrap-clean-arch MyCompany.MyApp --modules Abstractions,Implementation,Web.Api
```

## Plugin Structure

Each plugin lives under `plugins/<plugin-name>/` and contains:

```
plugins/<plugin-name>/
  skills/       # Claude Code skill definitions (SKILL.md files)
  rules/        # Coding rules bundled with the plugin
  docs/         # Plugin documentation
```

## Skills

### `/install-clean-arch-rules`

Deploys bundled coding rules (common, C#, TypeScript) to `<repo-root>/rules/`. Supports `overwrite` or `skip` arguments for existing rules.

### `/bootstrap-clean-arch`

Scaffolds a .NET clean architecture solution with proper dependency flow, generates per-module CLAUDE.md files with rule references. Requires rules to be deployed first.

**Arguments:** `[ProjectNamespace] [--modules Module1,Module2,...] [--claude-only]`

**Modules:** Common, Abstractions, Implementation, Repository, Client, Web.Core, Web.Server, Web.Api, Angular

## License

[MIT](LICENSE)
