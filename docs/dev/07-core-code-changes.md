# 7. Changing Pleasanter core code

Change core code only when site settings and Extended* cannot do the job ([01](01-choosing-a-mechanism.md)). Every core change is a merge you will repeat on each upstream release, so keep it small. [CLAUDE.md](../../CLAUDE.md) has the full build and run commands, and [source-map/](../../source-map/README.md) shows where each feature lives.

## 7.1 Find before you change

1. Look the feature up in [source-map/README.md](../../source-map/README.md) and open the map file it points to. Grep only when the map has no answer.
2. **Decide whether the file is generated.** Large parts of `Models/*`, `Controllers/<Table>Controller.cs`, `Rds.cs`, `Displays.cs`, `Messages.cs` and `Def.cs` are produced by CodeDefiner from `App_Data/Definitions/Definition_Code/*_Body.txt`. To find the template, grep the `*_Body.txt` files for a distinctive line.

## 7.2 Generated code

- **Change the template and the generated output in the same commit.** If you change only the output, the next `def` run overwrites it. If you change only the template, the code you run differs from the code in the repo.
- **Fix the shared template, not one table.** `Issue*`, `Result*`, `Wiki*` and `Dashboard*` are near-identical. A fix in `IssueUtilities.cs` alone leaves the same bug in `ResultUtilities.cs`.
- **Hand-written members inside generated files** carry `/// Fixed:` in their XML doc and survive regeneration. Files with `MergeToExisting: "1"` (`Settings/Column.cs`, `Initializer.cs`, `HtmlGrids.cs`) merge generated members into hand-written code.
- **On Linux, check the output paths before running `def` / `mvc`.** `OutputPath` uses `\`.

## 7.3 Schema, strings and parameters

| Change | Do |
|---|---|
| New column on a table | Add `App_Data/Definitions/Definition_Column/<Table>_<Column>.json`, run `CodeDefiner _rds` (schema) and `def` (code) |
| New UI string | Add `App_Data/Displays/<Id>.json` with **all languages** (en, zh, ja, de, ko, es, vn). Set `ClientScript: true` if JS needs it. Regenerate, then use `Displays.<Id>(context: context)` |
| New parameter file | Add the class in `Implem.ParameterAccessor/Parts`, the JSON in `App_Data/Parameters`, and the `Read<T>()` call in `Initializer.cs`. **New keys need safe defaults,** because existing installations won't have them |
| Raw SQL | `App_Data/Definitions/Sqls/{SQLServer,PostgreSQL,MySQL}/`. **Change all three** |
| SiteSettings shape change | Add a migrator `Libraries/Migrators/SiteSettings/Version1_0NN.cs`, bump `SiteSettingsUtilities.Version` (also in its template), chain it in `SiteSettingsMigrator.Migrate`. Make sure `GetRecordingData` strips defaults |

## 7.4 Coding conventions

- **Files:** UTF-8 **with BOM**, 4-space indent. Keep the BOM when editing and add it to new files.
- **C#:**
  - Use **named arguments** everywhere: `Foo(context: context, ss: ss)`.
  - Pass `Context` explicitly. Never use static or request-global state.
  - Build SQL with the `Rds.*` builders (`Rds.SelectIssues(...)`, `Rds.IssuesWhere()…`) so values are parameterized and the code works on all three DBMSs. Use raw SQL only through the `Sqls` definitions.
  - **Check permissions** with `Libraries/Security/Permissions.cs` and the generated `*Validators` (`OnEntry`, `OnEditing`, `OnUpdating`…). Never trust a client-supplied ID without checking `context.CanRead/CanUpdate`.
  - **Respect the switches** that turn customization off: `context.ServerScriptDisabled`, `ContractSettings.*`, `Parameters.Script.*`.
- **HTML:** render with `HtmlBuilder` (`Libraries/Html`, `HtmlParts`). Use `.Text(...)` for user data. **`Raw` is reserved for trusted, admin-authored content.**
- **Ajax:** return a `ResponseCollection` (`Html`, `ReplaceAll`, `Invoke`, `Events`…). After re-rendering a screen, emit the matching `.Events("on_*_load")` so site scripts keep working ([03](03-client-scripts-styles-htmls.md#34-events-pevents)).
- **Frontend:**
  - New code goes in `src/scripts/modules/` (TypeScript). Legacy `$p` code is in `src/scripts/generals/`.
  - **Don't rename or remove public `$p` functions or events.** Customer site scripts depend on them. Update `src/types/pleasanter-script.d.ts` when you add to the public API.
  - Prettier: 4 spaces, single quotes, no trailing commas, width 120 (100 for `.js`).
- **Server-script API:**
  - When you add a host-object member, also add it to `src/types/pleasanter-server-script.d.ts`.
  - Remember that `Values` / `Columns` are rebuilt per V8 run, so keep new host objects cheap to construct.
- **Comments and commit messages are in Japanese** (repo convention).

## 7.5 Verify

```bash
dotnet build Implem.Pleasanter/Implem.Pleasanter.csproj          # also builds the frontend
cd Implem.PleasanterFrontend/wwwroot && npm run typecheck && npm test
npx eslint <changed files> && npx prettier --check <changed files>
dotnet run --project Implem.Pleasanter                            # http://localhost:59803
```

- **Test on each DBMS you support** if you touched SQL, `Rds.*` or definitions.
- **Run through the screens and entry points the change touches:** full editor, dialog editor, grid edit, API, import and export.
- **Update [CLAUDE.md](../../CLAUDE.md) and [source-map/](../../source-map/README.md) in the same change** when you add, move or rename files, methods, endpoints, settings or modules (rules in CLAUDE.md).
