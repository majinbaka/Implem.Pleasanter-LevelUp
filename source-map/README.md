# Source map: where to look for each feature

Read this before searching the code. Pick the right file below, open the relevant section, and only then go to the source. **Fall back to grep only when the map does not cover what you need.** When you find a location the map is missing, or a path that has moved, update the matching file here.

Line numbers in the maps (`~1234`) are approximate and drift as the code changes. Open the file the map names, search for the method name inside it, and read the code around it.

## Files

| File | Covers |
|---|---|
| [request-flow.md](request-flow.md) | Startup and middleware order, routing, filters, `Context`, every controller and action → which model it calls, REST API, SCIM, authentication (Forms/LDAP/SAML/2FA/Passkey/API key), permissions, rate limiting |
| [models.md](models.md) | `Models/*`: generated vs hand-written code, method lists for `IssueUtilities`/`IssueModel` (Results has the same structure), Items dispatch, Sites (settings editor), Users/Depts/Groups/Tenants, Binaries, Permissions; HtmlParts, ViewModes, Search, Libraries/Models |
| [settings-subsystems.md](settings-subsystems.md) | `SiteSettings` and all of `Libraries/Settings` (View, Column, Notification, Process, StatusControl, Export, Formula…), SiteSettings migration, server scripts (V8), notification/mail sending, Quartz background services, AI Connect, MCP, SCIM, PDF, SitePackages, Extended* customization |
| [data-codegen.md](data-codegen.md) | SQL builder (`Implem.Libraries`), DBMS abstraction (`Rds/*`), CodeDefiner (schema + code generation), Def/Initializer, Parameters, `App_Data` (Definitions, Displays, Parameters), recipes: add a column, string, parameter, raw SQL; edit generated code |
| [frontend.md](frontend.md) | Build (Vite/MSBuild) and how C# includes the bundles, `$p` core (send/ajax/setByJson ↔ ResponseCollection), what each file in `generals/` does, `modules/` custom elements, SCSS, the public `$p` API, Svelte SmartDesign |

## Quick lookup (most common cases)

| Need | Go to |
|---|---|
| A button / URL `/items/{id}/xxx` → server handler | `Controllers/ItemsController.cs` action `Xxx` → `Models/Items/ItemModel.cs` `Xxx` → `Models/<Table>/<Model>Utilities.cs` |
| Record list (grid) | `IssueUtilities.Index/GridRows`, `Libraries/HtmlParts/HtmlGrids.cs`, `Libraries/Models/GridData.cs` |
| Filter / sort → SQL | `Libraries/Settings/View.cs` `Where` / `OrderBy` |
| Record editor screen | `IssueUtilities.Editor/EditorResponse/Field`, `Libraries/HtmlParts/HtmlFields.cs` |
| Saving a record (create/update) | `IssueUtilities.Create/Update` → `IssueModel.Create/Update` (server scripts, notifications, formulas, ExtendedSqls) |
| Site settings screen and dialogs | `SiteUtilities.*SettingsEditor/*Dialog`, `SiteModel.SetSiteSettings` (switch), `sitesettings.js` |
| SiteSettings JSON / migration | `Libraries/Settings/SiteSettings.cs`, `SiteSettingsUtilities.cs`, `Libraries/Migrators/` |
| Server script | `Libraries/ServerScripts/ServerScriptUtilities.Execute`, `Models/Shared/_BaseModel.cs` (hooks) |
| Notifications | `IssueModel.Notice` → `Libraries/Settings/Notification.Send` → `Libraries/DataSources/<Channel>.cs` |
| REST API | `Controllers/Api/ItemsController.cs` → `ItemModel.*ByApi` → `IssueUtilities.*ByApi`; envelope `Libraries/Requests/Api.cs` |
| Login / authentication | `UsersController.Authenticate` → `UserModel.Authenticate`; `Libraries/Security/` |
| Permissions | `Libraries/Security/Permissions.cs`, `Models/<T>/<T>Validators.cs`, `Models/Permissions/PermissionUtilities.cs` |
| Ajax response → DOM | `Libraries/Responses/ResponseCollection.cs` ↔ `src/scripts/generals/_dispatch.js` |
| Client JS for a feature | the "Feature → file" table in [frontend.md](frontend.md) |
| UI string / translations | `App_Data/Displays/<Id>.json` → `Displays.<Id>(context: context)` |
| Config | `App_Data/Parameters/<X>.json` ↔ `Implem.ParameterAccessor/Parts/<X>.cs` |
| DB schema of a table | `App_Data/Definitions/Definition_Column/<Table>_*.json` |
| SQL specific to one DBMS | `App_Data/Definitions/Sqls/<Dbms>/`, `Rds/Implem.<Dbms>/` |
| Generated code → template | search `Definition_Code/*_Body.txt` for a distinctive line |

## Important notes

- **Most of `Models/*` is generated, as are `Rds.cs`, `Displays.cs` and the others** (see the list in [models.md](models.md)). After editing generated code, update the template as well.
- **Results/Wikis/Dashboards mirror Issues.** If you know the Issues line, find the method of the same name in the other table's file.
- **Large files** (`Rds.cs` ~160k, `SiteUtilities.cs` ~20k, `IssueUtilities.cs` ~11k lines): search for a specific method name inside them. Do not read them whole.
