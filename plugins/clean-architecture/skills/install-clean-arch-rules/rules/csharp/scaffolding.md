# C# Scaffolding

Defines the mechanics for scaffolding a .NET clean architecture solution: file formats, tooling commands, NuGet package assignments, and starter code conventions. Used by the `/bootstrap-clean-arch` skill — do not reference this file from module CLAUDE.md files.

---

## Solution Setup

### Directory Structure

```
<solution-root>/
├── src/                     ← source projects
├── tests/                   ← test projects
├── <SolutionName>.sln
└── Directory.Packages.props
```

### Solution File

Create `<SolutionName>.sln` at the repo root if one does not exist. Add projects with solution folders:

```bash
dotnet new sln -n <SolutionName>
dotnet sln add src/<ProjectName>/<ProjectName>.csproj --solution-folder src
dotnet sln add tests/<ProjectName>.Tests/<ProjectName>.Tests.csproj --solution-folder tests
```

If a `.sln` already exists, add only the new projects to it.

---

## Central Package Management (CRITICAL)

Every solution MUST have a `Directory.Packages.props` at the solution root. Individual `.csproj` files declare `<PackageReference>` **without a `Version` attribute** — versions are managed exclusively in this file.

### Directory.Packages.props Template

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- DI / Extensions -->
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Abstractions" Version="9.0.0" />

    <!-- Entity Framework Core -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Tools" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0" />

    <!-- Web / API -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="9.0.0" />
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="7.0.0" />

    <!-- CLI -->
    <PackageVersion Include="System.CommandLine" Version="2.0.0-beta4.22272.1" />

    <!-- Validation -->
    <PackageVersion Include="FluentValidation" Version="11.0.0" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="11.0.0" />

    <!-- Testing -->
    <PackageVersion Include="xunit" Version="2.9.0" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.11.0" />
    <PackageVersion Include="FluentAssertions" Version="6.12.0" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
    <PackageVersion Include="coverlet.collector" Version="6.0.2" />
  </ItemGroup>
</Project>
```

Update package versions to latest stable at time of scaffolding. Only declare packages that are actually used — add entries as modules are added.

---

## Project File (.csproj) Templates

### Base Properties (all .NET projects)

```xml
<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

Match the `<TargetFramework>` of existing projects in the repo if any exist.

### Per-Module .csproj

**Common** — class library, no project or package dependencies:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

**Abstractions** — class library:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>...</PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  </ItemGroup>
</Project>
```

**Implementation** — class library:
```xml
<ItemGroup>
  <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
  <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  <PackageReference Include="FluentValidation" />
  <PackageReference Include="FluentValidation.DependencyInjectionExtensions" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
</ItemGroup>
```

**Repository** — class library:
```xml
<ItemGroup>
  <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
  <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
</ItemGroup>
```

**Client** — class library:
```xml
<ItemGroup>
  <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
  <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
</ItemGroup>
```

**Web.Core** — class library (not a web app SDK; references ASP.NET Core packages implicitly via transitive deps from Web.Server/Web.Api):
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" />
  </ItemGroup>
</Project>
```

**Web.Server** — ASP.NET Core web application:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Web.Core\[ProjectNamespace].Web.Core.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Repository\[ProjectNamespace].Repository.csproj" />
    <!-- Angular SPA — build-only reference, no assembly output -->
    <ProjectReference Include="..\[projectnamespace].client\[projectnamespace].client.esproj"
                      ReferenceOutputAssembly="false" />
  </ItemGroup>
</Project>
```

Omit the Angular `.esproj` reference if no Angular module was requested.

**Web.Api** — ASP.NET Core web application:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Web.Core\[ProjectNamespace].Web.Core.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Repository\[ProjectNamespace].Repository.csproj" />
    <PackageReference Include="Swashbuckle.AspNetCore" />
  </ItemGroup>
</Project>
```

**Cli** — console application:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Repository\[ProjectNamespace].Repository.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
    <PackageReference Include="System.CommandLine" />
    <PackageReference Include="Microsoft.Extensions.Hosting" />
  </ItemGroup>
</Project>
```

---

## Angular .esproj

Project name is lowercase: `[projectnamespace].client`. Place it under `src/`.

```xml
<!-- [projectnamespace].client.esproj -->
<Project Sdk="Microsoft.VisualStudio.JavaScript.SDK/1.0">
  <PropertyGroup>
    <StartupCommand>npm start</StartupCommand>
    <JavaScriptTestRoot>src/</JavaScriptTestRoot>
    <JavaScriptTestFramework>Jasmine</JavaScriptTestFramework>
    <GeneratePkgDefFile>false</GeneratePkgDefFile>
    <ShouldRunBuildScript>false</ShouldRunBuildScript>
    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
  </PropertyGroup>
</Project>
```

Additional files to create (see `typescript/angular.md` for package and config details):

| File | Notes |
|---|---|
| `angular.json` | Set `outputPath` to `../[ProjectNamespace].Web.Server/wwwroot` |
| `package.json` | Angular dependencies per `typescript/angular.md` |
| `tsconfig.json` | TypeScript config per `typescript/angular.md` |
| `proxy.conf.js` | Dev proxy: route `/api` to `https://localhost:<port>` |
| `src/app/core/` | Singleton services, guards, interceptors |
| `src/app/features/` | Feature-sliced lazy modules |
| `src/app/shared/` | Reusable components, pipes, directives |
| `src/app/layout/` | Shell (header, sidebar, footer) |
| `src/main.ts` | Bootstrap entry point |
| `src/app/app.component.ts` | Root component |
| `src/app/app.config.ts` | Application configuration |
| `src/app/app.routes.ts` | Root route definitions |

---

## Test Projects

For each source module, create `tests/[ProjectNamespace].[AssemblyType].Tests/`.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\[ProjectNamespace].[AssemblyType]\[ProjectNamespace].[AssemblyType].csproj" />
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="FluentAssertions" />
    <PackageReference Include="NSubstitute" />
    <PackageReference Include="coverlet.collector">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

Include a placeholder test class so the project compiles:

```csharp
namespace [ProjectNamespace].[AssemblyType].Tests;

public class PlaceholderTests
{
    [Fact]
    public void Placeholder() => Assert.True(true);
}
```

---

## Starter Code

Generate minimal files per module — just enough to compile and demonstrate the module's role. All generated code MUST follow `csharp/coding-style.md` and `common/patterns.md`.

| Module | Files to Create |
|---|---|
| Common | README placeholder only |
| Abstractions | `Interfaces/IExampleService.cs`, `Models/ExampleModel.cs` |
| Implementation | `Services/ExampleService.cs` (implements IExampleService), `Extensions/ServiceCollectionExtensions.cs` |
| Repository | `Contexts/AppDbContext.cs`, `Entities/ExampleEntity.cs`, `Repositories/ExampleRepository.cs`, `Extensions/ServiceCollectionExtensions.cs` |
| Client | `Clients/ExampleApiClient.cs`, `Extensions/ServiceCollectionExtensions.cs` |
| Web.Core | `Controllers/ExampleController.cs` (thin — delegates to IExampleService) |
| Web.Server | `Program.cs` (HTTPS, static files, SPA fallback, DI wiring) |
| Web.Api | `Program.cs` (HTTPS, Swagger UI, DI wiring) |
| Cli | `Program.cs` (RootCommand + IHost via `CommandLineBuilder.UseHost()`), `Commands/ExampleCommand.cs` |

### Starter Code Constraints

- No default parameters — use overloads
- All async methods accept `CancellationToken ct` as the last parameter
- `Program.cs` for Cli: follow patterns in `csharp/command-line.md`
- Keep files minimal — no over-engineering, no speculative abstractions

---

## Build Verification

After scaffolding, verify compilation:

```bash
dotnet build <SolutionName>.sln
```

If the build fails, diagnose and fix before reporting success. Common causes:
- Package not declared in `Directory.Packages.props`
- Project reference path incorrect (check relative depth `src/` vs `tests/`)
- Namespace mismatch between `.csproj` and generated code
