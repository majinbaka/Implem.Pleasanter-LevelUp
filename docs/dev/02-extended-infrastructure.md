# 2. Extended* infrastructure: loading, conditions, reload, deploy

Every `App_Data/Parameters/Extended*` kind shares the same loader, the same condition model and the same deploy rules. Read this page before you add any extension.

## 2.1 How files are loaded

The loader is `Implem.DefinitionAccessor/Initializer.cs`. `SetParameters()` assigns each `Parameters.Extended*` list, using one loader per directory.

| Directory | Files read | Recursive | Notes |
|---|---|---|---|
| `ExtendedSqls` | `*.json` + optional `<file>.json.sql` | yes | The `.json.sql` content replaces `CommandText` |
| `ExtendedServerScripts` | `*.json` + optional `<file>.json.js` | yes | The `.json.js` content replaces `Body` |
| `ExtendedScripts` | `*.js` | yes | Files sorted by name, then sub-folders sorted by name. `Name` = file name |
| `ExtendedStyles` | `*.css` | yes | Same order as scripts |
| `ExtendedHtmls` | `*.html` | yes | File name = HTML position id (see [03](03-client-scripts-styles-htmls.md#361-extendedhtmls-positions)) |
| `ExtendedFields`, `ExtendedNavigationMenus`, `ExtendedPlugins`, `ExtendedStartGuides` | `*.json` | yes | |
| `ExtendedColumnDefinitions` | `*.json` | no | Column templates, see [06](06-other-extensions.md) |
| `ExtendedTags` | `*.html` | no | Only `Header.html` is used. The directory does not exist by default |

### Best practices

- **Keep SQL and JS in companion files.** Use `Foo.json` + `Foo.json.sql` / `Foo.json.js` instead of escaping a long body inside a JSON string. Then the SQL and JS get syntax highlighting, linting and readable diffs.
- **The `Sample.*.txt` files are ignored** because of their extension. To disable a file temporarily, rename it to `*.txt` / `*.ignore`, or set `"Disabled": true`.
- **Never delete an `Extended*` directory.** Most loaders call `GetFiles` without an `Exists` check, so a missing folder crashes startup. Leave the folder in place, even if it is empty.
- **Validate the JSON before deploying.** Invalid JSON throws `ParametersIllegalSyntaxException`, and the app does not start (or the reload fails). Use `jq . file.json` in CI.
- **Use sub-folders per feature or customer** (`ExtendedSqls/Sales/…`). Load order is by file name. It matters for scripts and styles, so use number prefixes (`10_common.js`, `20_sales.js`) when order matters.
- **Avoid `_` in ExtendedHtml file names.** The text after the last `_` is read as a language code.

## 2.2 The Extensions table (DB-backed extensions)

- **Columns:** `ExtensionType`, `ExtensionName`, `ExtensionSettings` (JSON of the matching Extended* class), `Body`, `Description`, `Disabled`.
- **Supported types:** `Sql`, `ServerScript`, `Script`, `Style`, `Html`, `Fields`, `NavigationMenu`, `Plugin`. There is no StartGuide type, and unknown types are silently ignored.
- **Loading:** `Libraries/Initializers/ExtensionInitializer.cs` appends the rows to the file-loaded lists, ordered by `ExtensionName`.
  - `ExtensionName` overrides `Name`.
  - `Body` overrides `CommandText` / `Body` / `Script` / `Style` / `Html`.
- **API:** `Controllers/Api/ExtensionsController.cs` serves `POST /api/extensions/Get|{id}/Get|Create|{id}/Update|{id}/Delete`. It requires all of:
  - the tenant flag `AllowExtensionsApi`
  - an authenticated **privileged** user (`Security.json` `PrivilegedUsers`)
  - an allowed license environment

Gotchas:
- **The query has no `TenantId` filter.** In a multi-tenant instance, a row applies to **every tenant** unless its conditions restrict it.
- **Create/Update through the API do not reload.** Call `/Admins/ReloadParameters` afterwards, or restart.
- **An invalid `ExtensionSettings` row is skipped silently**, while a bad file crashes startup. After a reload, check that your row actually took effect.

## 2.3 Conditions (who / where an extension applies)

Every Extended* item inherits `Implem.ParameterAccessor/Parts/ExtendedBase.cs`:

```json
{
    "Name": "SalesSummary",
    "SpecifyByName": false,
    "Description": "Owner: sales team. Ticket #123",
    "Disabled": false,
    "SiteIdList": [], "IdList": [],
    "DeptIdList": [], "GroupIdList": [], "UserIdList": [],
    "Controllers": [], "Actions": [], "ColumnList": []
}
```

Matching is done by `Models/Extensions/ExtensionUtilities.cs` `ExtensionWhere` / `MeetConditions`:

- **An empty or omitted list means "all".** A null or empty *request value* also passes. For example, `ColumnList` is ignored when no column is involved.
- **Positive entries:** the value must equal one of the entries.
- **Negative entries:** `[-5]` means "everything except 5".
  - **Do not mix positive and negative entries.** As soon as a negative entry is present, anything not excluded matches, so `[1, -2]` also matches site 3.
- **There is no `*` wildcard.**
- **`Controllers` / `Actions` must be lowercase route names** (`"items"`, `"index"`, `"edit"`, `"new"`), because `Context` lower-cases them.
- `SiteId`/`Id` come from the current request and are `0` on non-site pages and on `/api/extended/sql`. A positive `SiteIdList` never matches there.
- **`GroupIdList`** matches if any of the user's groups matches.
- **`SpecifyByName: true`** makes the item apply only when the caller asks for it by `Name`, e.g. a view's `OnSelectingWhere` name. Use it for every reusable filter or API SQL so it cannot attach itself everywhere by accident.

**Best practice: always restrict.** An extension with all lists empty runs on every page or every save of every site. Set at least `SiteIdList` (or `SpecifyByName`) unless the extension really is global, and write the reason in `Description`.

## 2.4 When changes take effect

- **Files and Extensions rows are read at startup.**
- **`GET /Admins/ReloadParameters`** (privileged users only) re-reads all parameter files and Extensions rows **in the current process only**. On a multi-node (load-balanced) deployment, reload or restart every node.
- **There is no file watcher.**
- **Plugins** (PDF DLLs) are cached statically, so replacing a DLL needs a restart.
- **The DB `Parameters`-table patch mechanism** can replace some `Extended*` lists wholesale (`Initializer.PatchParameters`). If a file change "does nothing", check that table.

## 2.5 Promoting from dev to prod

| What | Travels in a site package? | How to promote |
|---|---|---|
| Site Scripts / Styles / Htmls / ServerScripts / Views / Processes… | **yes**: the whole SiteSettings | Export and import the site package |
| Notifications / reminders | optional (`SitePackage.json`) | |
| Extended* files | **no** | Copy the files (keep them in git), then restart or reload |
| Extensions rows | **no** | Call `/api/extensions`, then reload |

- **IDs inside script and SQL bodies are not rewritten** when a site package is imported. Site IDs and record IDs differ between environments, so do not hard-code them:
  - **Client JS:** `$p.siteId('Site title')` resolves a joined site by title.
  - **Server scripts:** `items.GetSiteByName` / `GetSiteByTitle` / `GetClosestSite`, or `siteSettings.SiteId(title)`.
  - **Use site `SiteName`** (a stable identifier you set) rather than IDs wherever the API supports it.
- **Server scripts are disabled during site-package import**, so `AfterCreate`-style logic does not run for imported data.
- **Keep every customization in git:** Extended* files directly, and the site package JSON (or at least each script body) for site settings. Review them in PRs like code.
