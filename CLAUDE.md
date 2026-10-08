# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Finding code: check source-map/ before you grep

Before looking for where a feature, screen, endpoint or piece of logic is handled:

1. Read [source-map/README.md](source-map/README.md). It has a quick-lookup table and an index of the detailed map files.
2. Open the matching map file (request-flow, models, settings-subsystems, data-codegen or frontend). Go straight to the file and method it names, and search for the method name inside that file.
3. **Only Grep/Glob or start an Explore agent when the map does not cover what you need.**
4. When you find a location the map is missing, or a path in the map has moved, update the matching file in `source-map/`.

## Overview

Pleasanter is a no-code/low-code business application platform: ASP.NET Core (.NET 10, SDK pinned in `global.json`) backend, a TypeScript/jQuery frontend, and SQL Server, PostgreSQL or MySQL as the database.

## Commands

### Backend

```bash
# Build the web app. This also builds the frontend: Implem.PleasanterFrontend is an MSBuild
# NoTargets project that runs `npm install` + `npm run build` in its wwwroot.
dotnet build Implem.Pleasanter/Implem.Pleasanter.csproj
# Skip the npm step (this is what the Dockerfiles do)
dotnet build Implem.Pleasanter/Implem.Pleasanter.csproj -p:SkipBuildTask=true

# Run (launch profile "Implem.Pleasanter.NetCore" -> http://localhost:59803, login Administrator / pleasanter)
dotnet run --project Implem.Pleasanter
```

The database connection comes from environment variables `Implem.Pleasanter_Rds_<Dbms>_SaConnectionString`, `..._OwnerConnectionString` and `..._UserConnectionString`, where `<Dbms>` is `SQLServer`, `PostgreSQL` or `MySQL`. The engine is selected by `Dbms` in `Implem.Pleasanter/App_Data/Parameters/Rds.json` (default is PostgreSQL). CONTRIBUTING.md has example values.

### CodeDefiner (database setup and code generation)

```bash
dotnet run --project Implem.CodeDefiner -- _rds                      # create/migrate DB schema from definitions
dotnet run --project Implem.CodeDefiner -- _rds /l "ja" /z "Asia/Tokyo" /y   # with language/timezone, no prompts
dotnet run --project Implem.CodeDefiner -- def                       # regenerate Def.cs + all generated MVC code
dotnet run --project Implem.CodeDefiner -- mvc /t Displays           # regenerate a single code definition by Id
```

Other actions are listed in `Implem.CodeDefiner/Starter.cs`: `rds` (= `_rds` + `def`), `_def`, `migrate`, `merge`, `ConvertTime`, `trial` and `backup`. Definition `OutputPath` values use `\` separators, so `def`/`mvc` generation appears to be Windows-oriented. On Linux, check the output paths before trusting it.

### Frontend (`Implem.PleasanterFrontend/wwwroot`)

```bash
npm run dev          # watch builds + browser-sync live reload
npm run build        # tsc + vite build, output to Implem.Pleasanter/wwwroot/assets (gitignored)
npm run typecheck
npm test             # vitest, "unit" project (happy-dom)
npx vitest run --project unit src/scripts/modules/richTextEditor/richTextEditor.test.ts   # single file
npx vitest run --project unit -t "test name"                                             # single test
npm run test:browser # vitest "browser" project (Playwright/Chromium), runs *.browser.test.ts
```

There is no lint script. ESLint, Stylelint and Prettier are configured, so run them directly (`npx eslint <file>`, `npx stylelint "src/**/*.scss"`, `npx prettier --check <file>`).

The Svelte web components live in `Implem.PleasanterFrontend/components` (`npm run build`, `npm run check`). They build to `Implem.Pleasanter/wwwroot/components`, which is committed and is **not** built by MSBuild.

### Tests

The repo contains no .NET unit test project (`InternalsVisibleTo Implem.PleasanterTest` refers to an external one). `Implem.TestAutomation` is a Selenium E2E runner driven by `App_Data/Parameters/AutoTestSettings.json` and `ExtendedAutoTest/`.

## Architecture

### Definition-driven code generation (read this before editing models)

A large share of the C# is **generated** by `Implem.CodeDefiner` from JSON definitions under `Implem.Pleasanter/App_Data/Definitions/`:

- `Definition_Column/<Table>_<Column>.json`: schema for every table, covering column types, indexes, grid/editor/filter defaults, labels and API mapping. `_rds` creates or migrates DB tables from these files.
- `Definition_Code/<Id>.json` + `<Id>_Body.txt`: C# templates.
  - `<!--OtherId-->` includes another definition.
  - `#TableName#` / `#ModelName#` are substituted.
  - `RepeatType: "Table"` emits one file per table.
- Generated outputs include:
  - `Models/<Table>/*` (`*Model`, `*Utilities`, `*Collection`, `*Validators`, `*ApiModel`, `*ExportModel`)
  - `Controllers/<Table>Controller.cs`
  - `Libraries/DataSources/Rds.cs` (~160k lines)
  - `Libraries/Responses/Displays.cs` and `Messages.cs`
  - `Implem.DefinitionAccessor/Def.cs`
  - frontend `src/scripts/generals/display.ts`

When you change generated code, **make the same change in the matching `Definition_Code/*_Body.txt` template**. Otherwise the next `def` run overwrites it. Generated files and template changes are committed together.

- Members whose XML doc contains `/// Fixed:` are hand-written and survive regeneration.
- Definitions with `MergeToExisting: "1"` (e.g. `Settings/Column.cs`, `DefinitionAccessor/Initializer.cs`, `HtmlGrids.cs`) merge generated members into a mostly hand-written file.
- Because models are generated per table, `Issue*` and `Result*` (and the other tables) contain near-identical code. A fix usually belongs in the shared template, not in one table's file.

### UI strings and parameters

- **UI strings:** each string is one file, `App_Data/Displays/<Id>.json`, with bodies for en (default), zh, ja, de, ko, es and vn. Setting `ClientScript: true` also exposes the string to the frontend. After regeneration, C# reads it via `Displays.<Id>(context: context)`.
- **Parameters:**
  - `App_Data/Parameters/*.json` deserialize into classes in `Implem.ParameterAccessor/Parts` through `Initializer.Read<T>()` and are used as `Parameters.<Name>`.
  - A new parameter file needs all three pieces: the class, the JSON file and the `Read` call.
  - The `Extended*` directories are runtime customization points for scripts, server scripts, SQL, styles, HTML and column definitions. Samples are `*.json.txt`.
  - `docs/dev/` is the developer guide for building features: site Scripts/Styles/Htmls, server scripts, ExtendedSqls and the other `Extended*` kinds, core-change conventions and a review checklist. Keep it in sync when you change how these mechanisms behave.

### Data access

- `Implem.Libraries/DataSources/SqlServer` is the DBMS-agnostic SQL builder (`SqlSelect`, `SqlWhere`, `SqlStatement`, …), despite its name. Engine-specific behavior goes through `ISqlObjectFactory` (`Rds/Implem.IRds`), implemented in `Rds/Implem.{SqlServer,PostgreSql,MySql}` and chosen by `Implem.Factory.RdsFactory` from `Parameters.Rds.Dbms`.
- The generated `Rds.cs` provides per-table fluent builders (`Rds.SelectIssues(...)`, `Rds.IssuesColumn()`, `Rds.IssuesWhere()`, …).
- Each table has `_deleted` and `_history` companion tables (`Sqls.TableTypes`).
- Raw SQL lives per engine in `App_Data/Definitions/Sqls/{SQLServer,PostgreSQL,MySQL}/`. Keep all three in sync.

### Request flow and rendering

- Controllers construct `new Context()` (`Libraries/Requests/Context.cs`: request, user, tenant and permissions), then call `*Model` / `*Utilities`, which query through `Rds`.
- HTML is rendered server-side in C# with `HtmlBuilder` (`Libraries/Html`, `Libraries/HtmlParts`). Most Razor views only output `ViewBag.HtmlBody`.
- Ajax endpoints return a `ResponseCollection` (`Libraries/Responses`): a JSON list of `{Method, Target, Value}` DOM commands that the client applies in `$p.setByJson` (`src/scripts/generals/_dispatch.js`).
- User-built apps are **Sites** with a `ReferenceType`: `Sites` (folder), `Issues` (period table), `Results` (record table), `Wikis` or `Dashboards`.
  - The `Items` table maps every record ID to its type. `ItemsController` / `ItemModel` dispatch on `ReferenceType` to `IssueUtilities`, `ResultUtilities` and so on.
  - Per-site configuration (columns, views, scripts, notifications, processes, …) is `SiteSettings` (`Libraries/Settings/SiteSettings.cs`), stored as JSON.
- Other subsystems:
  - user server scripts run on ClearScript V8 (`Libraries/ServerScripts`)
  - MCP server in `Implem.Pleasanter/MCP`
  - SCIM in `Controllers/Scim` and `Libraries/Scim`
  - background jobs on Quartz (`Libraries/BackgroundServices`)

### Frontend

- `src/scripts/generals/` is the legacy global-`$p` jQuery code, bundled through `app.ts` as an IIFE.
- `src/scripts/modules/` holds newer TS modules (`modules.ts`).
- SCSS entries are `src/styles/*.scss`; files prefixed with `_` are partials.
- `src/plugins` and `src/clone` are copied verbatim.
- The server finds hashed bundles through `assets/manifest.json`.

## Conventions

- Source files are UTF-8 **with BOM** (`.editorconfig`: `utf-8-bom`, 4-space indent). Preserve the BOM when editing and add it to new files. Git normalizes line endings (`* text=auto`).
- C# calls use named arguments almost everywhere (`Foo(context: context, ss: ss)`). Follow that style.
- Prettier settings (frontend): 4 spaces, single quotes, no trailing commas, `printWidth` 120 (100 for `.js`).
- Code comments are sparse and written in Japanese, and so are commit messages. `Implem.PleasanterFrontend/wwwroot/.github/copilot-instructions.md` asks for Japanese in all responses, code reviews and comments.

## Keep CLAUDE.md and source-map/ up to date

Every change must leave this file and `source-map/` accurate. Before you finish a task, check whether it changed anything they describe, and update them in the same change (same commit/PR) as the code:

- **CLAUDE.md:** update it when you change commands, build/test/run steps, project structure, architecture, code generation, parameters, or conventions.
- **source-map/:** update the matching map file when you add, move, rename or delete a file, class, method, controller action, endpoint, setting or frontend module that a map lists or should list. Fix any map entry you notice is wrong, even if your change did not cause it. If you add a new area, add it to the quick-lookup table and file index in `source-map/README.md`.
- Skip the update only when the change touches nothing these files describe (for example a bug fix inside an existing method). Say so in your summary when you skip it.
- Keep entries short and in the existing style; don't record line numbers as exact values.
