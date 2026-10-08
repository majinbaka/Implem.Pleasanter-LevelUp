# Data access, DBMS, CodeDefiner, definitions and parameters

Paths are relative to the repo root.

## SQL builder (`Implem.Libraries/DataSources/SqlServer/`)

The builder is DBMS-agnostic despite the folder name.

- **Base class:** `SqlStatement.cs` holds the Column/Join/Where/OrderBy/Param/GroupBy/Having collections and the `BuildCommandText` method. Params get a `#CommandCount#` suffix.
- **Subclasses:**
  - `SqlSelect` (Top/PageSize/Offset/Union)
  - `SqlInsert`, `SqlUpdate`, `SqlUpdateOrInsert`
  - `SqlDelete` (logical: moves the row to `_deleted`), `SqlPhysicalDelete`, `SqlRestore`
  - `SqlExists`
- **Parts:** `SqlColumn*`, `SqlWhere*` (Operator, Sub, Raw, And/Or), `SqlJoin*`, `SqlOrderBy*`, `SqlGroupBy*`, `SqlHaving*`, `SqlParam*`.
- **`Sqls.cs`:** the `TableTypes` enum (Normal, History, Deleted, NormalAndDeleted, All…) and the `Functions` enum.
- **Execution:**
  - `SqlIo.cs` (ExecuteNonQuery / Scalar / Table / DataSet) and `SqlContainer.cs` (statement list, transaction).
  - `SqlDebugs.cs` handles SQL logging.
  - Connections: `Def.SqlIoBySa/ByAdmin/ByUser` in `Implem.DefinitionAccessor/Def.cs` (~169–265).
- **App wrapper:** `Implem.Pleasanter/Libraries/DataSources/Repository.cs` (`Repository.ExecuteTable(context, statements: ...)` etc.).
- **Generated fluent API:** `Implem.Pleasanter/Libraries/DataSources/Rds.cs` (~160k lines).
  - Statements: `Rds.Select/Insert/Update/UpdateOrInsert/Delete/PhysicalDelete/Restore/Exists<Table>`.
  - Builders: `Rds.<Table>Column()/Where()/Join()/OrderBy()/Param()/GroupBy()`.
  - Templates: `Definition_Code/Rds_*`.
  - **Do not grep the whole of Rds.cs.** Search for one specific method name.
- **`Implem.Libraries/Utilities/`:** extension helpers.
  - `Strings`, `Types` (ToInt/ToBool…)
  - `Jsons` (ToJson/Deserialize/ApplyPatch)
  - `Files`, `Environments` (global state), `DateTimes`, `CryptographyAes`
- **`Implem.Libraries/Classes/XlsIo.cs`:** reads the definition folders.

## DBMS abstraction (`Rds/`)

- **Engine selection:** `Implem.Factory/RdsFactory.Create(dbms)` returns SqlServer/PostgreSql/MySql `ObjectFactory`. The choice comes from `Parameters.Rds.Dbms`.
- **Factory access:** `Implem.Pleasanter/Libraries/Requests/Context.cs` itself implements `ISqlObjectFactory`. App code calls `context.Sqls.X`, `context.SqlCommandText.X`.
- **Interfaces in `Rds/Implem.IRds/`:**

  | Interface | Covers |
  |---|---|
  | `ISqlCommandText` | TOP/LIMIT, select identity, upsert (MERGE / ON CONFLICT / ON DUPLICATE KEY), delete/restore, TRY_CAST, full-text where |
  | `ISqls` | true/false literals, now(), LIKE/escape, date add/grouping, **permission SQL** (`GetPermissions`, `PermissionsWhere`, `Site*Where`), UpsertBinary |
  | `ISqlDataType` | Definition type → engine type |
  | `ISqlDefinitionSetting` | Identifier length, varchar reduction, PG schema |
  | `ISqlErrors` | Duplicate key / deadlock / timeout codes |
  | ADO wrappers | Command, Connection, Parameter… |

- **Implementations:** `Rds/Implem.{SqlServer,PostgreSql,MySql}/<Prefix>{ObjectFactory,CommandText,Sqls,DataTypes,DefinitionSetting,Errors,...}.cs`.
- **Per-DBMS differences live in four places.** Keep all of them in sync:
  1. `Implem.Pleasanter/App_Data/Definitions/Sqls/{SQLServer,PostgreSQL,MySQL}/<Name>.sql` (55 files per engine, same names). At runtime they are `Def.Sql.<Name>`.
  2. `Rds/Implem.*/*Sqls.cs` and `*CommandText.cs`.
  3. CodeDefiner schema code: `Implem.CodeDefiner/Functions/Rds/Parts/{Columns,Constraints,Indexes}.cs`, plus `Parts/MySql/*` and `Parts/PostgreSql/*`.
  4. `Parameters.Rds.Dbms` checks in `Startup.cs`, `Context.cs`, `Libraries/Server/DistributedLock.cs`, `BackgroundServices/CustomQuartzHostedService.cs`, `DataTypes/Attachment.cs`.

## CodeDefiner (`Implem.CodeDefiner/`)

- **`Starter.cs`:**
  - Actions:

    | Action | What it does |
    |---|---|
    | `_rds` | Database (create/migrate) → `Functions/Rds/Configurator.Configure` |
    | `rds` | `_rds` + `def` |
    | `_def` | `DefinitionAccessorCreator` (Def.cs, Initializer.cs) |
    | `def` | `_def` + `MvcCreator.Create` |
    | `mvc /t <Id>` | Regenerates one definition |
    | `splits_rds` | Splits `Rds.cs` |
    | `migrate` | SQL Server → PG/MySQL, `Functions/AspNetMvc/CSharp/Migrator.cs` |
    | `ConvertTime` | `Functions/Rds/TimeConverter.cs` |
    | `merge` | `Functions/Patch/PatchParameters.cs` |
    | `backup`, `trial` | |

  - Args: `/p` path, `/t` target, `/l` language, `/z` timezone, `/f` force, `/y` no prompt, `/c` check only.
- **Schema (`Functions/Rds/`):**
  - `Configurator.Configure`:
    1. When Provider=Local: `RdsConfigurator` (DB), `UsersConfigurator` (logins), `SchemaConfigurator` (PG schema).
    2. `CheckColumnsShrinkage`: aborts if Issues/Results columns were removed, unless `/f`.
    3. `TablesConfigurator`.
    4. `PrivilegeConfigurator`.
  - `TablesConfigurator`: for every table it handles normal + `_deleted` + `_history`. A missing table goes to `Parts/Tables.CreateTable`. When `HasChanges`, `MigrateTable` copies the data into a new table and swaps it in. Then the full-text index per engine.
  - `Parts/`: `Tables`, `Columns`, `ColumnSize`, `Indexes`, `Constraints`, plus `MySql/*` and `PostgreSql/*`.
- **Code generation (`Functions/AspNetMvc/CSharp/`):**
  - `MvcCreator.cs`: for each table and each definition with RepeatType=Table, it calls `Creators.Create`. It then replaces `#TableName#`, `#ModelName#`, `#tableName#`, `#ServiceName#` and calls `Merger.Merge`.
  - `Creators.cs` expands `<!--ChildId ( Opt = v )-->` (regex in `Utilities/CodePatterns.cs`). It dispatches on the child's RepeatType: Table, Column, Join, Form, Display, DefinitionRow, BaseModel…
  - Placeholders: `Parts/Column.cs` handles `#ColumnName#`, `#Type#`, `#ByForm#`… `Parts/Display.cs` and `Parts/Join.cs` handle theirs.
  - `Parts/Table.cs` `CheckExclude` handles the `Include`/`Exclude`/`ItemOnly`/`NotItem`/`GenericUi` options.
  - `Merger.cs` + `Parser.cs`:
    - Members marked `/// Fixed:` are kept; `/// NotMerge:` is honored.
    - `MergeToExisting=1` merges the generated members into the existing file.
  - `Utilities/CodeWriter.cs` writes only when the content changed. OutputPath uses `\`.
  - Valid options: `Definition_Code/__ColumnSettings.json`, handled by `Def.SetCodeDefinitionOption`.

### Top-level code definitions (`Implem.Pleasanter/App_Data/Definitions/Definition_Code/<Id>.json`)

| Id | Output (relative to Implem.Pleasanter/) |
|---|---|
| Base | Models/Shared/_BaseModel.cs |
| Model, Model_Utilities, Model_Validator, Model_Api, Model_Export, Collection | Models/<Table>/<Model>{Model,Utilities,Validators,ApiModel,ExportModel,Collection}.cs |
| Controller | Controllers/<Table>Controller.cs |
| Rds | Libraries/DataSources/Rds.cs |
| Displays, Messages, ResponseCollection | Libraries/Responses/Displays.cs, Messages.cs, ResponseSpecials.cs |
| ClientDisplays | ../Implem.PleasanterFrontend/wwwroot/src/scripts/generals/display.ts |
| Column (merge), SiteSettings, Lookup, LinkAction, Reminder (merge) | Libraries/Settings/Column.cs, SiteSettingsUtilities.cs, Lookup.cs, LinkAction.cs, Reminder.cs |
| GridData (merge), Summaries, FormulaUtilities | Libraries/Models/* |
| HtmlGrid (merge), HtmlLinks | Libraries/HtmlParts/HtmlGrids.cs, HtmlLinks.cs |
| Indexes (merge) | Libraries/Search/Indexes.cs |
| ItemsInitializer, Error | Libraries/Initializers/ItemsInitializer.cs, Libraries/General/Error.cs |
| Def, Initializer (merge) | ../Implem.DefinitionAccessor/Def.cs, Initializer.cs |

- **Child snippets:** named `<Parent>_<Part>.json` (options only) + `<Parent>_<Part>_Body.txt` (code).
- **Snippet counts by prefix:** Model_* about 677, Rds_* 82, Summaries_* 20, Def_* 17, SiteSettings_* 13, Collection_* 11, HtmlGrid_* 10.
- **Finding the template for a piece of generated code:** search for a distinctive line in `Definition_Code/*_Body.txt`.

## DefinitionAccessor and ParameterAccessor

- **`Implem.DefinitionAccessor/Initializer.cs`** (merged, mostly hand-written):
  - `Initialize()` runs these steps in order:
    1. `SetParameters`
    2. `SetRdsParameters`
    3. `SetDefinitions`
    4. `SetSqls`
    5. `PatchParameters` (patches from the DB `Parameters` table)
    6. `ApplyPatchedEnvironmentDefaults`
  - `SetParameters` (~93–200): `Parameters.X = Read<X>()`. **The file name is the class name + `.json`.** Connection strings come from env vars, then the Extended* folder loaders run.
  - `SetDefinitions` (~970): the Displays hash and the Code/Column/Template/ViewMode/Demo/Sql definitions. `SetDisplayAccessor` auto-adds a Display for each column label.
- **`Implem.DefinitionAccessor/Def.cs`** (generated):
  - `Def.ColumnTable.<Table>_<Column>` (full definition) and `Def.Column.<Id>`.
  - `Def.Sql.<Name>` (class `SqlColumn2nd` ~14684, `/// Fixed:`).
  - `TableNameCollection()`.
- **`Implem.DefinitionAccessor/Directories.cs`:** path helpers.
- **`Implem.ParameterAccessor/Parameters.cs`:** static `Parameters.<Name>` plus license helpers. `Parts/<Name>.cs` ↔ `App_Data/Parameters/<Name>.json`.
  - Exceptions: `CustomApps.json` → `Parameters.UserTemplate`, `NavigationMenus.json` → a list, `AutoTestSettings`.
- **`Implem.DisplayAccessor/`:** Display/DisplayElement models and the `Types` enum (Normal 110, Success 210, Information 220, Warning 230, Error 240, Confirmation 310…).
- **`Implem.Plugins/`:** `IPdfPlugin` contract.
- **`Implem.TestAutomation/`:** Selenium E2E, driven by `AutoTestSettings.json` + `ExtendedAutoTest/`.

## App_Data (`Implem.Pleasanter/App_Data/`)

| Folder | Content |
|---|---|
| Definitions/Definition_Code/ | C# templates (about 1790 files) |
| Definitions/Definition_Column/ | `<Table>_<Column>.json`: schema + UI metadata. `_Bases_*` / `_BaseItems_*` are the columns shared by all tables (`Base: "1"`). Field list: `__ColumnSettings.json` |
| Definitions/Definition_Template/ | Site template gallery (`TemplateNNN.json` + `_SiteSettingsTemplate.json`) |
| Definitions/Definition_Demo/ | Demo data |
| Definitions/Definition_ViewMode/Definition.json | View modes per ReferenceType |
| Definitions/Definition_Sql/ | Legacy Id list (SQL Server bodies). Runtime SQL comes from `Sqls/` |
| Definitions/Sqls/{SQLServer,PostgreSQL,MySQL}/ | Runtime `.sql` (DDL, permissions, group members…). Placeholders `#TableName#`, `#CommandCount#`, `@_T/@_D/@_U` |
| Displays/ | One file per UI string (about 1350): `{"Id","Type","ClientScript"?,"Languages":[{Body (en)},{"Language":"zh",...},ja,de,ko,es,vn]}` |
| Parameters/*.json | Runtime config (list below) |
| Parameters/Extended*/ | Runtime customization (`Sample*.txt` only) |
| Parameters/ExtendedColumnDefinitions/ | Default definitions for ClassA…/NumA… columns |
| Parameters/CustomDefinitions/ (optional) | Overrides definition rows without editing the JSON |

Parameter files:
- `Rds` (**Dbms**, connection strings), `Service` (name, timezone, language), `Authentication` (provider, LDAP/SAML/Passkey/TrustedProxy)
- `Security` (IP, lockout, password policy, CSP), `Session`, `Kvs` (Redis)
- `Mail`, `Notification`, `Reminder`
- `Api`, `RateLimit`, `Script` (V8 limits)
- `Search`, `SitePackage`, `BinaryStorage`
- `BackgroundService`, `BackgroundJobs`, `Quartz`
- `AiConnect`, `McpServer`, `Scim`
- `General`, `Site`, `User`, `Permissions`, `Validation`, `Form`, `Registration`
- `Deleted`, `History`, `Dashboard`, `Mobile`, `TextEditorUI`, `SysLog`, `SummarySync`, `Migration`
- `NavigationMenus`, `CustomApps`, `Locations`, `Version`, `ParameterSetting`, `OutputCache`, `Parameter` (SqlParameterPrefix)

## Recipes

### Add a column to a built-in table

1. Copy a sibling `Definition_Column/<Table>_<Col>.json` and fill in its fields, including LabelText for every language.
2. Run `dotnet run --project Implem.CodeDefiner -- _rds`. This migrates the table and its `_deleted`/`_history` companions.
3. Run `def` to regenerate `Def.cs`, the Models, Rds.cs, …
4. Commit the definition and the generated files together.
5. For user-extendable columns (ClassA…), use `Parameters/ExtendedColumnDefinitions` instead.

### Add a UI string

1. Create `App_Data/Displays/<Id>.json` with a BOM and all 7 languages. Set `ClientScript: true` if JS needs it.
2. Run `mvc /t Displays`. Also run `/t Messages` if Type is 210–240, and `/t ClientDisplays` for JS.
3. Use it as `Displays.<Id>(context: context)` or `Messages.<Id>(context: context)`.

### Add a parameter

- **New field in an existing file:** edit `Parts/<X>.cs` + `<X>.json`.
- **New file:**
  1. Create `Parts/<X>.cs` and `App_Data/Parameters/<X>.json` (BOM).
  2. Add `public static <X> <X>;` in `Parameters.cs`.
  3. Add `Parameters.<X> = Read<<X>>();` in `Initializer.SetParameters`.
  4. Add a `case "<X>"` in `Initializer.PatchParameters`.

### Raw SQL for all 3 DBMS

- **File-based:**
  1. Create `Sqls/{SQLServer,PostgreSQL,MySQL}/<Name>.sql`.
  2. Add `public string <Name>;` to `SqlColumn2nd` in `Def.cs`.
  3. Use it as `new SqlStatement(commandText: Def.Sql.<Name>)` through `Repository`. Example: `Models/Groups/GroupModel.cs`.
- **Fragment:** add the member to `Rds/Implem.IRds/ISqls.cs` (or `ISqlCommandText.cs`), implement it in all 3 `*Sqls.cs` / `*CommandText.cs`, and use it as `context.Sqls.<Name>`.

### Change generated code

1. Find the template by searching for a distinctive line in `Definition_Code/*_Body.txt`.
2. Edit the template and apply the same edit to the generated file. Shared templates mean a fix applies to all tables. Limit it with `Include`/`Exclude` if needed.
3. Alternatively, mark the member `/// Fixed:` in the generated file.
4. If you run `mvc /t <Id>`, check the output paths on Linux.
