---
name: bootstrap-clean-arch
description: Scaffold one or more clean architecture projects and generate per-module CLAUDE.md files. Reads modularization rules from the deployed /rules folder to determine project structure, dependency boundaries, and applicable rules. Use when starting a new project, adding modules to an existing solution, or regenerating CLAUDE.md files.
argument-hint: "[ProjectNamespace] [--modules Module1,Module2,...] [--claude-only]"
---

# Bootstrap Clean Architecture Project

Scaffold a clean architecture .NET solution (or add modules to an existing one) and generate per-module CLAUDE.md files. All decisions are driven by the rules deployed in `<repo-root>/rules/`.

## Prerequisites

The `rules/` directory must exist at the repo root with at least `rules/csharp/modularization.md`. If it does not exist, tell the user to run `/install-clean-arch-rules` first and stop.

## Step 1 — Parse Arguments

Parse `$ARGUMENTS` to determine:

- **`ProjectNamespace`** (required unless `--claude-only`): The root namespace for the solution (e.g., `Corvel.ToDos`, `MyCompany.Billing`). This drives all assembly naming via the `[ProjectNamespace].[AssemblyType]` convention from modularization rules.
- **`--modules`** (optional): Comma-separated list of specific modules to create. Valid module names match the assembly types defined in `rules/csharp/modularization.md`:
  - `Common`, `Abstractions`, `Implementation`, `Repository`, `Client`, `Web.Core`, `Web.Server`, `Web.Api`, `Angular`, `Cli`
  - If omitted, ask the user which modules they need. Do NOT create all modules by default — follow the modularization rule that says "assemblies should only be created if applicable."
- **`--claude-only`**: Skip project scaffolding entirely. Only generate/update CLAUDE.md files for existing projects. When this flag is present, `ProjectNamespace` is optional — discover it from existing `.csproj` files.

If arguments are missing or ambiguous, ask the user.

## Step 2 — Read the Rules

1. Read `<repo-root>/rules/` recursively to catalog all available rule files.
2. Read the **full content** of every rule file, paying special attention to:
   - `csharp/modularization.md` — defines the assembly structure, dependency flow, folder layout, and naming conventions. **This file is used for scaffolding only — do NOT include it in CLAUDE.md `@` references.** Each module's CLAUDE.md already documents its purpose and dependency constraints.
   - `csharp/coding-style.md` — coding conventions that apply to generated code.
   - `csharp/domain.md` — domain model and contract patterns.
   - `csharp/services.md` — service layer, validation, and DI patterns.
   - `csharp/persistence.md` — repository, EF Core, and data access patterns.
   - `csharp/presentation.md` — controller, minimal API, and middleware patterns.
   - `csharp/hosting.md` — Program.cs pipeline, background services, and containerization.
   - `csharp/command-line.md` — System.CommandLine patterns, DI wiring, and CLI conventions.
   - `common/coding-style.md` — universal coding conventions.
3. Build a catalog of rules organized by:
   - **Language**: common (applies to all), csharp, typescript
   - **Concern**: coding-style, testing, security, domain, services, persistence, presentation, hosting, logging
   - **Applicability criteria**: What kind of project/module does each rule apply to? (e.g., domain libraries, service layers, repositories, web APIs, hosting apps, Angular frontends, test projects)

## Step 3 — Read Root CLAUDE.md

Read the root `CLAUDE.md` to understand the existing repository architecture, conventions, and any custom instructions. If a root CLAUDE.md does not exist, note this and generate one at the end.

## Step 4 — Scaffold Project Structure (skip if `--claude-only`)

For each requested module, create the project skeleton **following the rules from modularization.md exactly**:

### 4a. Solution-Level Layout

Create the standard directory structure from modularization rules:

```
src/
├── [ProjectNamespace].Common/
├── [ProjectNamespace].Abstractions/
├── [ProjectNamespace].Implementation/
├── [ProjectNamespace].Repository/
├── [ProjectNamespace].Client/
├── [ProjectNamespace].Web.Core/
├── [ProjectNamespace].Web.Server/
├── [ProjectNamespace].Web.Api/
└── [projectnamespace].client/          (Angular, lowercase)

tests/
├── [ProjectNamespace].Common.Tests/
├── [ProjectNamespace].Abstractions.Tests/
├── ... (one test project per src module)
```

Only create directories for the modules the user requested.

### 4b. .csproj Files

For each .NET module, generate a `.csproj` file with:
- Appropriate `<TargetFramework>` (use the latest stable .NET version, currently `net9.0`, or match existing projects in the repo)
- `<RootNamespace>` and `<AssemblyName>` matching the project name
- `<ProjectReference>` entries following the **Dependency Flow Guidelines** from modularization.md exactly:
  - Common: no project references
  - Abstractions: Common
  - Implementation: Abstractions, Common
  - Repository: Abstractions, Common
  - Client: Abstractions, Common
  - Web.Core: Abstractions, Implementation, Common
  - Web.Server: Web.Core, Implementation, Repository (+ Angular .esproj if applicable)
  - Web.Api: Web.Core, Implementation, Repository
  - Cli: Abstractions, Implementation, Repository, Common
- Appropriate NuGet package references based on module type (e.g., `Microsoft.EntityFrameworkCore` for Repository, `Microsoft.AspNetCore.*` for Web projects)
- `<ImplicitUsings>enable</ImplicitUsings>` and `<Nullable>enable</Nullable>`

### 4c. Angular .esproj (if Angular module requested)

Follow the `.esproj` template from modularization.md exactly:
- Use lowercase naming: `[projectnamespace].client`
- Create `angular.json`, `package.json`, `tsconfig.json`, `proxy.conf.js`
- Create the folder structure: `src/app/core/`, `src/app/features/`, `src/app/shared/`, `src/app/layout/`
- Create minimal bootstrap files: `main.ts`, `app.component.ts`, `app.config.ts`, `app.routes.ts`

### 4d. Test Projects

For each src module, create a corresponding test project:
- `[ProjectNamespace].[AssemblyType].Tests`
- Reference the module being tested
- Add xUnit, FluentAssertions, and NSubstitute NuGet references
- Include a placeholder test class

### 4e. Solution File

If a `.sln` file does not exist at the repo root, create one. If it exists, add the new projects to it using `dotnet sln add`.

Organize the solution into folders:
- `src` — all source projects
- `tests` — all test projects

### 4f. Starter Code

For each module, generate minimal starter files that demonstrate the module's role. Keep these minimal — just enough to compile and show the pattern:

| Module | Starter Files |
|---|---|
| Common | (empty — placeholder README only) |
| Abstractions | One example interface (e.g., `IExampleService.cs`), one example model |
| Implementation | One service implementing the example interface, DI registration extension |
| Repository | DbContext, one example repository, DI registration extension |
| Client | One example API client |
| Web.Core | One example controller |
| Web.Server | `Program.cs` with middleware pipeline |
| Web.Api | `Program.cs` with Swagger setup |
| Cli | `Program.cs` with RootCommand + one example subcommand using System.CommandLine + IHost DI wiring |

**All generated code MUST follow the rules** from `rules/csharp/coding-style.md` and `rules/csharp/patterns.md`. Specifically:
- No default parameters — use overloads
- All async methods must accept `CancellationToken` as the last parameter
- Follow naming conventions from the rules

## Step 5 — Generate CLAUDE.md Files

For each project (both existing and newly created), generate a `CLAUDE.md` following the same process as the bootstrap-claude-md skill:

### 5a. Determine Applicable Rules

Use the **minimum applicable set** principle. Only include rules where the project genuinely needs that guidance. Follow this matrix (adapt based on actual rule contents):

| Rule File | Domain Lib | App/Service Layer | Infrastructure | Web API (Web.Core) | Hosting (Web.Server/Api) | Cli | Angular Frontend | Test Project |
|---|---|---|---|---|---|---|---|---|
| common/coding-style.md | Y | Y | Y | Y | Y | Y | Y | Y |
| common/logging.md | — | Y | — | Y | Y | Y | — | — |
| common/patterns.md | Y | Y | Y | Y | Y | Y | Y | — |
| common/security.md | — | — | Y | Y | Y | Y | Y | — |
| common/testing.md | — | — | — | — | — | — | — | Y |
| common/command-line.md | — | — | — | — | — | Y | — | — |
| csharp/coding-style.md | Y (C#) | Y (C#) | Y (C#) | Y (C#) | Y (C#) | Y (C#) | — | Y (C#) |
| csharp/domain.md | Y (C#) | Y (C#) | — | — | — | — | — | — |
| csharp/services.md | — | Y (C#) | — | Y (C#) | — | Y (C#) | — | — |
| csharp/persistence.md | — | — | Y (C#) | — | Y (C#) | — | — | — |
| csharp/presentation.md | — | — | — | Y (C#) | Y (C#) | — | — | — |
| csharp/hosting.md | — | — | — | — | Y (C#) | Y (C#) | — | — |
| csharp/command-line.md | — | — | — | — | — | Y (C#) | — | — |
| csharp/security.md | — | — | Y (C#) | Y (C#) | Y (C#) | Y (C#) | — | — |
| csharp/testing.md | — | — | — | — | — | — | — | Y (C#) |
| typescript/coding-style.md | — | — | — | — | — | — | Y (TS) | Y (TS) |
| typescript/angular.md | — | — | — | — | — | — | Y | — |
| typescript/patterns.md | — | — | — | — | — | — | Y (TS) | — |
| typescript/security.md | — | — | — | — | — | — | Y (TS) | — |
| typescript/testing.md | — | — | — | — | — | — | — | Y (TS) |

### 5b. Generate CLAUDE.md Content

```markdown
# [Project Name]

[Brief description of the module's purpose]

## Rules

@[relative path to rule file]
@[relative path to next rule file]
...

## Module Purpose

[1-3 sentences describing what this module does]

## Key Contents

[Bullet list of main classes, services, or components]

## Dependency Constraints

[Allowed and forbidden dependencies per modularization rules]
```

### 5c. Handle Existing CLAUDE.md Files

- **No existing CLAUDE.md**: Create from scratch.
- **Existing CLAUDE.md**: Preserve custom content. Update only the `## Rules` section. Do not delete developer-written documentation.

### 5d. Root CLAUDE.md

If no root `CLAUDE.md` exists, generate one that:
- Describes the solution architecture
- Lists all modules and their purposes
- References the rules architecture
- Documents the deployed rules structure

If a root `CLAUDE.md` already exists, update its module listing to include any newly created projects while preserving all existing content.

## Step 6 — Verify

1. If projects were scaffolded, run `dotnet build` on the solution to verify everything compiles.
2. If the build fails, fix the issues before reporting success.
3. If only CLAUDE.md files were generated (`--claude-only`), verify that all `@` reference paths resolve to actual rule files.

## Step 7 — Report

Provide:

### Projects Scaffolded (if applicable)

| Project | Type | Directory | Status |
|---|---|---|---|
| [ProjectNamespace].Abstractions | Class Library | src/... | Created |
| ... | ... | ... | ... |

### CLAUDE.md Files

| Project | Rules Included | Status (Created/Updated) |
|---|---|---|
| ... | ... | ... |

### Dependency Graph

Show the dependency flow as a simple text diagram matching the modularization rules.

### Next Steps

Suggest what the user should do next (e.g., "Add your domain models to Abstractions", "Configure your DbContext in Repository", etc.)

## Constraints

- **Rules MUST be deployed first** — if `/rules` does not exist, stop and tell the user to run `/install-clean-arch-rules`.
- **Follow modularization rules exactly** — do not invent your own structure. The rules in `rules/csharp/modularization.md` are the source of truth.
- **Do not create modules the user didn't ask for** — always ask or use `--modules` to confirm which modules are needed.
- **All generated code must follow the deployed coding rules** — read them before generating any code.
- **Relative paths in CLAUDE.md must be accurate** — double-check directory depth.
- **Respect existing files** — never overwrite existing source code. Only overwrite CLAUDE.md rule sections.
- **Keep generated code minimal** — just enough to compile and demonstrate the pattern. Do not over-engineer starter code.
