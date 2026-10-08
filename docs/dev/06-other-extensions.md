# 6. Other extension points

All of these use the shared loader and condition model described in [02](02-extended-infrastructure.md).

## 6.1 ExtendedFields: custom filter and view inputs

- **Class:** `Implem.ParameterAccessor/Parts/ExtendedField.cs`.
- **Fields:**
  - `FieldType`
  - `TypeName` (default `nvarchar`)
  - `LabelText`, `ChoicesText`, `DefaultInput`, `ControlType`
  - `Validate*`, `Min` / `Max` / `Step`, `DecimalPlaces`
  - `FieldCss`, `ControlCss`, `After`
  - `SqlParam`
- **`FieldType` values:**
  - `"Filter"` adds an input to the filter panel. The value is stored in the view's `ColumnFilterHash`.
  - `"ViewExtensions"` adds inputs to a `#ViewExtensions` area (not on folder sites). The value is stored in `ViewExtensionsHash`.
- **With `SqlParam: true`,** the value is bound as `@<Name>` in grid queries. Pair it with `OnSelectingWhere` / `OnSelectingColumn` ExtendedSqls, as in the example in [05 §5.4](05-extended-sqls.md#54-view-filters-and-sorts-onselectingwhere--onselectingorderby).
- **Restrict with `SiteIdList`.** An unrestricted field appears on every site's filter panel.

## 6.2 ExtendedNavigationMenus

- **Format:** `{ "TargetId": "<MenuId>", "Action": "Append|Prepend|Remove|Replace|ReplaceAll", "NavigationMenus": [ … ] }`, applied in `HtmlParts/HtmlNavigationMenu.cs`.
- **Menu item fields:** `ContainerId`, `MenuId`, `Css`, `Icon`, `Url`, `Function`, `LinkParams`, `Target`, `ReferenceTypes`, `ChildMenus`.
- **Base menu IDs** are in `App_Data/Parameters/NavigationMenus.json`, e.g. `NewMenu`, `ViewModeMenu`, `SettingsMenu`, `HelpMenu`, `AccountMenu`, `AccountMenu_Logout`.

Best practices:
- **Prefer `Append` / `Prepend` / `Remove`** over `Replace` / `ReplaceAll`. They keep upstream menu additions working after an upgrade.
- **Hiding a menu item is not access control.** The URL behind it must still be protected by permissions.
- Use `ReferenceTypes` and the conditions (`SiteIdList`, `GroupIdList`…) to show items only where they make sense.

## 6.3 ExtendedStartGuides

- **Format:** `{ "TargetId", "Action", "StartGuides": [{ "Id", "Text", "Url", "ImgNameLight", "ImgNameDark", "Target" }] }`, with the same five actions as menus.
- **Default IDs:** `SgManual`, `SgGoHandsOn01`, `SgMaterials`, `SgExtensions`.
- **File-only:** there is no Extensions-table type for start guides.

## 6.4 ExtendedPlugins (PDF)

- **Registration:** `{ "Name": "Pdf", "PluginType": "Pdf", "LibraryPath": "Pdf/PdfPlugin.dll" }`. The DLL lives under `App_Data/UserPlugins/<LibraryPath>`. **Use `/` on Linux.** The sample file uses `\\`.
- **Contract** (`Implem.Plugins/`):

  ```csharp
  public interface IPdfPlugin { PdfData CreatePdf(IPdfPluginHost host); }
  // host: SiteTitle, Url, ReportId, GetGridData(string viewJson) -> List<Dictionary<string, object>>
  ```
- **Trigger:** `ItemsController.Pdf(id, reportId)` → `IssueUtilities.Pdf` / `ResultUtilities.Pdf`. The first plugin that matches `SiteIdList` wins.
- **Loading:** `PdfPluginCache` loads the DLL with `Assembly.LoadFrom` and caches it, so **replacing the DLL needs a restart**.
- **Target the same .NET version** as Pleasanter (see `global.json`).
- **Keep it stateless and thread-safe.** One instance is shared by all requests.
- **Fetch data only through `host.GetGridData(viewJson)`.** It applies the current user's permissions. Don't open your own DB connection.

## 6.5 ExtendedColumnDefinitions / ExtendedColumns: more columns

- **Defaults:** `Def.cs` `SetExtendedColumnDefinitions` generates **A–Z (26 each)** of `Class`, `Num`, `Date`, `Description`, `Check` and `Attachments` for Issues, Results, Users, Depts and Groups. The templates live in `App_Data/Parameters/ExtendedColumnDefinitions/{Class,Num,Date,…}.json`.
- **More columns** (`Class001…`) or **disabling columns** need `App_Data/Parameters/ExtendedColumns/*.json` (`TableName`, counts per type, `DisabledColumns`) **and a commercial or trial license**.
- **Any change** to these definitions changes the physical schema. Run `dotnet run --project Implem.CodeDefiner -- _rds` against every environment, then restart. **Plan this like a DB migration:** take a backup and test on a copy.
- **Before adding columns, consider** a linked child site, or splitting the site. Very wide tables hurt grid, export and server-script performance (every column is marshalled into every V8 run).

## 6.6 ExtendedAutoTest (E2E)

- **Settings:** `App_Data/Parameters/AutoTestSettings.json`.
- **Scenarios:** `ExtendedAutoTest/TestCases/*.json`, which reference parts in `TestParts/{Items,Other,Users}/*.json`. Rename a file to `*.json.ignore` to disable it.
- **Runner:** `Implem.TestAutomation` (Selenium). Paths use `\\`, so it is Windows-oriented.
- **Add a scenario for each critical customization:** a validation that must block a save, a button added by script, and so on. That gives upgrades a regression net.

## 6.7 Adding a new kind of Extended* (core change)

Do this only when no existing kind fits. It requires four changes:
1. a class in `Implem.ParameterAccessor/Parts` (inheriting `ExtendedBase`)
2. a loader in `Implem.DefinitionAccessor/Initializer.cs`, **with a `Directory.Exists` check**
3. a case in `PatchParameters`, if the list should be patchable
4. a case in `Libraries/Initializers/ExtensionInitializer.cs`, if it should be DB-backed

Then update `source-map/settings-subsystems.md` (see [07](07-core-code-changes.md)).
