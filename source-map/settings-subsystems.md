# SiteSettings and subsystems

Paths are relative to `Implem.Pleasanter/` unless noted. Line numbers are approximate.

## SiteSettings (`Libraries/Settings/`)

Per-site configuration is stored as JSON in `Sites.SiteSettings`.

### SiteSettings.cs (~6400 lines)

- **Fields (~80–309):**
  - Columns: `Columns`, `ColumnHash`, `GridColumns`, `FilterColumns`, `EditorColumnHash`, `Tabs`, `Sections`, `LinkColumns`, `HistoryColumns`.
  - Feature lists: `Aggregations`, `Links`, `Summaries`, `Formulas`, `Processes`, `StatusControls`, `Views`, `Notifications`, `Reminders`, `AiProviders`, `Exports`, `Styles`, `Scripts`, `Htmls`, `ServerScripts`, `BulkUpdateColumns`, `RelatingColumns`, `DashboardParts`. Each has a `*AllDisabled` flag.
  - Switches: `Enable{Calendar,...,ImageLib}`, `Use*Filter`.
  - Column access control: `Create/Read/UpdateColumnAccessControls`.
  - Integration and forms: `IntegratedSites`, `Form*`.
- **Lifecycle and persistence:**
  - `Init(context)` (~335).
  - `RecordingJson` / `RecordingData` (~784) persist the settings with defaults stripped (each child has `GetRecordingData`).
- **Columns:**
  - `GetColumn` (~2410), `GetColumnOrExtendedFieldColumn`.
  - `GetGridColumns` (~2588), `GetFilterColumns` (~2641), `GetEditorColumns` (~2692).
  - `ReadableColumns` (~4169).
- **Settings editor setters:** `Set(context, propertyName, value)` (~4204) and `SetColumnProperty` (~4428), the generic setters used by the editor.
- **Dialog options:** the `*SelectableOptions` / view-mode option lists for dialogs are at ~2914–4133.
- **Links and choices:** `SetLinks` (~4640), `SetChoiceHash` (~4692–4871), `LinkHash` (~4986), `GetLinkedSiteSettings`.
- **Structure:** `AddTab`/`AddSection`/`AddView` (~5226–5313).
- **Exports and joins:** `GetExport`/`ExportColumns` (~5336), `Join`/`SqlJoinCollection` (~5594).
- **Injection of site scripts, styles and HTML:**
  - `EditorScripts`/`ViewModeScripts` (~5774)
  - `EditorStyles`/`ViewModeStyles` (~5697)
  - `ViewModeHtmls` (~5834)
- **Server scripts:** `GetServerScripts` (~6204) merges the site's ServerScripts with `Parameters.ExtendedServerScripts`.

### Load, deserialize and migrate

- **Load:** `SiteSettingsUtilities.cs` (generated).
  - `Get(...)` (~17–75) and `GetByReference` (~124).
  - Per-table defaults: `IssuesSiteSettings` (~744), `ResultsSiteSettings` (~818), `UsersSiteSettings` …
- **Deserialize:** `DeserializeSiteSettings` (~983) runs `Deserialize`, then `SiteSettingsMigrator.Migrate` if the version differs, then `Init`.
- **Version:** the `Version` constant is at ~15 (e.g. `1.017M`).
- **Migration:** `Libraries/Migrators/SiteSettingsMigrator.cs` (`Migrate` chains `Migrate1_001…1_0NN`) with `Libraries/Migrators/SiteSettings/Version1_0NN.cs`.
  - **To add a migration:** create `Version1_0NN.cs`, bump `SiteSettingsUtilities.Version` (in the template too), and add the call to `Migrate`.
  - Statuses have their own migrator: `Migrators/StatusesMigrator.cs`.

### Other Settings files

| File | Purpose / key members |
|---|---|
| Column.cs (gen, merged) | One column: choices (`SetChoiceHash`, `EditChoices`), Display/DisplayGrid/DisplayControl, Round, DateTimeFormat, defaults, CanCreate/CanRead/CanUpdate (column access control), `Update` from dialog |
| View.cs | Saved view: ColumnFilterHash, ColumnSorterHash, Search, Calendar*/Kamban* params, `OnSelectingWhere`/`OnSelectingOrderBy` (ExtendedSql names). `SetByForm` (~522), `Where` (~1844), `SetColumnsWhere` (~2145), typed filters (`CsNumericColumns`, `CsDateTimeColumns`, `CsStringColumns` ~2930), `SetByWhenViewProcessingServerScript` (~3312), `OrderBy` (~3330), `SetSearchWhere` (~3461), `MergeSession` (~3580) |
| Notification.cs / NotificationUtilities.cs / NotificationColumnFormat.cs | Notification rule. `Types` enum (Mail, Slack, ChatWork, Line, LineGroup, Teams, RocketChat, InCircle, HttpClient, LineWorks). `Send` (~210) switches on type. `MeetConditions` |
| Reminder.cs (gen, merged) / ReminderUtilities.cs | Scheduled reminders. `Remind` (~165) |
| Process.cs / ValidateInput.cs / DataChange(Utilities).cs | Process (workflow button) with input validation and value changes. `IsTarget`, `Accessable` |
| StatusControl.cs | Per-status read-only/hidden/required columns |
| Export.cs / ExportColumn.cs / ExportUtilities.cs | Export templates. `ExportUtilities.Export` (~26) builds CSV/JSON |
| ImportUtilities.cs / ImportColumn.cs | CSV import mapping. OnImporting/OnImported ExtendedSqls |
| Formula.cs / FormulaSet.cs / FormulaBuilder.cs | Classic formulas. `ParseFormulaScript` handles script-style formulas |
| Summary.cs | Summary definition. Execution is in `Libraries/Models/Summaries.cs` |
| Link.cs / LinkAction(s).cs / Join.cs / JoinStack.cs | Link columns. `CopyWithLinks`, `DeleteWithLinks`. Join paths |
| Lookup.cs (gen) / Lookups.cs | Copy values from a linked record |
| ServerScript.cs | Site server script settings (Body, conditions, Shared, TimeOut) |
| BackgroundServerScript(s).cs / BackgroundSchedule.cs / BackgroundServerScriptUtilities.cs | Tenant scheduled scripts. `Reschedule` → Quartz `BackgroundServerScriptJob` |
| Script.cs / Style.cs / Html.cs | Site JS / CSS / HTML injection |
| DashboardPart*.cs / DashboardTimeLineItem.cs | Dashboard parts (QuickAccess, TimeLine, Custom, CustomHtml, Calendar, Kamban, Index) |
| ColumnAccessControl.cs | `Allowed` |
| AiProvider.cs / AiProviderUtilities.cs | Per-site AI provider settings |
| ColumnUtilities.cs / ColumnNameInfo.cs | Extended columns, `~siteId,Column` joined names |
| Tab, Section, Aggregation, BulkUpdateColumn, RelatingColumn, Choice, AutoNumbering, Form | Small types |
| UserSettings, TenantSettings, ContractSettings | User prefs, tenant settings, contract limits (users, sites, storage, API, IP, deadline) |
| PasskeyData, PasswordHistory | Authentication data |
| MultilingualLabelExportImport.cs | Label translations |
| SettingList.cs | Generic `SettingList<T>` (Get, MoveUpOrDown, Copy, Delete) |

**Settings editor UI:**
- Dialog HTML: `Models/Sites/SiteUtilities.cs` `*SettingsEditor` / `*Dialog` methods.
- Button handling: `Models/Sites/SiteModel.cs` `SetSiteSettings(context, res)`, a `switch(controlId)` with about 110 cases.
- Frontend: `sitesettings.js`.
- See [models.md](models.md) for line numbers.

## Server scripts (`Libraries/ServerScripts/`, ClearScript V8)

- `ScriptEngine.cs`: wraps `V8ScriptEngine` (timeout callback, AddHostObject, Execute).
- `ServerScriptUtilities.cs`:
  - Data passing:
    - Model values into the script: `Values` (~191), `SavedValues`, `Columns`.
    - Results back onto the model: `SetIssueModelValues` (~860), `SetResultModelValues` (~978), `SetViewValues`.
  - **`Execute` (~1167)** sets up the engine:
    1. Runs `ServerScriptJsLibraries.ScriptInit()` (`$ps`).
    2. Adds host objects: `context`, `grid`, `model`, `saved`, `depts`, `groups`, `users`, `columns`, `siteSettings`, `view`, `items`, `hidden`, `responses`, `elements`, `extendedSql`, `notifications`, `httpClient` (unless disabled), `utilities`, `logs`, `_file_cs`, `_csv_cs`.
    3. Runs the shared scripts, then the condition-matching scripts.
  - Script-side CRUD helpers: Get, Create, Update, Upsert, Delete, BulkDelete, Aggregate, ImportItem, ExportItem (~1452–1857).
- `ServerScriptModel.cs`: the `ServerScriptConditions` enum (~40):
  - WhenViewProcessing, WhenloadingSiteSettings, BeforeOpeningPage, BeforeOpeningRow, WhenloadingRecord
  - Before/AfterFormula, Before/AfterCreate, Before/AfterUpdate, Before/AfterDelete, Before/AfterBulkDelete
  - BackgroundServerScript
- **One file per host object:**

  | Host object | File(s) |
  |---|---|
  | `context` | `ServerScriptModelContext.cs` |
  | `items` | `ServerScriptModelApiItems.cs`, `ServerScriptModelApiModel.cs` |
  | `httpClient` | `ServerScriptModelHttpClient.cs` |
  | `extendedSql` | `ServerScriptModelExtendedSql.cs` |
  | `notifications` | `ServerScriptModelNotification*.cs` |
  | `utilities` | `ServerScriptModelUtilities.cs` |
  | `logs` | `ServerScriptModelLogs.cs` |
  | `depts`, `groups`, `users` | `ServerScriptModel{Depts,Groups,Users}*.cs` |
  | `siteSettings` | `ServerScriptModelSiteSettings.cs` |
  | `view` | `ServerScriptModelView.cs` |
  | `grid` | `ServerScriptModelGrid.cs` |
  | `hidden` | `ServerScriptModelHidden.cs` |
  | `responses` | `ServerScriptModelResponse(s).cs` |
  | `elements` | `ServerScriptElements.cs` |
  | `_file_cs` | `ServerScriptFile.cs` |
  | `_csv_cs` | `ServerScriptCsv.cs` |

- `ServerScriptJsLibraries.cs` holds the JS prelude (`$ps`, `$p.JSON`).
- `FormulaServerScriptUtilities.cs` runs script-style formulas (Excel-like functions). It is called from `IssueModel`/`ResultModel`/`WikiModel`/`DashboardModel` `ExecFormulaExtended`.
- **Where scripts are triggered:**
  - `Models/Shared/_BaseModel.cs` `SetBy*ServerScript` (~758–1106)
  - `View.SetByWhenViewProcessingServerScript`
  - `Libraries/BackgroundServices/BackgroundServerScriptJob.cs`
- **Types for the script API:** `Implem.PleasanterFrontend/wwwroot/src/types/pleasanter-server-script.d.ts`.

## Notifications and mail

- **Trigger:**
  1. `IssueModel.Notice` (~4297) is called after Create/Update/Delete, and from `IssueUtilities.UpdateByGrid`.
  2. It calls `Settings/Notification.Send` (~210).
  3. Bulk update (`IssueUtilities.BulkUpdate`) sends one summary message per notification. Notifications with `NotifyPerRecordOnBulkUpdate` (`Notification.GetNotifyPerRecordOnBulkUpdate`) are skipped there and sent per record instead, through `Update(notice: true, noticeType: "BulkUpdated")` → `Notice` case `"BulkUpdated"`.
  4. `Notification.ReplaceRelatedUsers` returns a copy, so `[RelatedUsers]` resolves per record in bulk loops.
- **Clients in `Libraries/DataSources/`:** `Slack.cs`, `Teams.cs`, `ChatWork.cs`, `Line.cs`, `LineWorks.cs`, `RocketChat.cs`, `InCircle.cs`, `HttpClient.cs` (webhook), `NotificationHttpClient.cs`.
- **Mail:** `Models/OutgoingMails/OutgoingMailModel.cs` `Send` (~944) uses one of:
  - SMTP: `DataSources/Smtp.cs` (supports OAuth)
  - SendGrid: `SendGridMail.cs`
  - AWS SES: `AwsSesMail.cs`
- **Address parsing:** `Libraries/Mails/Addresses.cs`.
- **Config:** `App_Data/Parameters/Notification.json` and `Mail.json`. `Libraries/Initializers/NotificationInitializer.cs` sets up the notification parameters at startup.

## Background services (Quartz, `Libraries/BackgroundServices/`)

- **Startup:** `CustomQuartzHostedService.cs` → `TimerBackground.InitAsync` (~16) registers the timers:
  - `SyncByLdapExecutionTimer`
  - `DeleteSysLogsTimer`, `DeleteTemporaryFilesTimer`, `DeleteTrashBoxTimer`
  - `ReminderBackgroundTimer` (→ `ReminderScheduleUtilities.Remind`)
  - `DeleteUnusedRecordTimer`, `DeleteMcpLogsTimer`, `DeleteTenantsTimer`
  - `BackgroundJobDispatcher`, `BackgroundJobTimeoutMonitor`, `DeleteBackgroundJobsTimer`
- **Timer base classes:** `ExecutionTimerBase.cs`, `ClusterExecutionTimerBase.cs`, `TimerTriggerRegistrar.cs`.
- **Job queue:**
  - `BackgroundJobDispatcher.cs` registers the handlers: `ExportJobHandler`, `ImportJobHandler`, and the AiConnect sync/resync/indexcheck/delete handlers.
  - The queue itself is `Models/BackgroundJobs/BackgroundJobQueue.cs`.
- **`BackgroundServerScriptJob.cs`:** runs the scheduled tenant server scripts.
- **`ApplicationWarmupHostedService.cs`:** runs the initializers (Tenant, Extension, Users, Items, Statuses, Notification) and the migrators (~188).
- **Config:** `App_Data/Parameters/BackgroundService.json`, `BackgroundJobs.json`, `Quartz.json`.

## Other subsystems

| Subsystem | Where |
|---|---|
| AI Connect (push data to OpenAI / Dify knowledge bases) | `Libraries/AiConnect/`: `AiConnectUtilities.cs` (Sync, Delete, Output markdown), `AiConnectQueueUtilities.cs`, `AiConnectProviderFactory.cs`, `AiConnect{OpenAi,Dify,MarkdownFile}Provider.cs`, `AiConnectHttpClient.cs`; settings `Settings/AiProvider*.cs`; `Parameters/AiConnect.json` |
| MCP server | `MCP/Tools/{Items,Sites,Views,Users,OutgoingMails}Tool.cs`, `MCP/Resources/`, `MCP/Prompts/WorkflowPrompts.cs`, `MCP/Infrastructure/McpContextMiddleware.cs` (auth/context), `McpRateLimit*.cs`, `MCP/McpContext/`, `MCP/Translator/` (label ↔ column name), `MCP/Utilities/`; `Startup.ConfigureMcpServer`; `Parameters/McpServer.json` |
| SCIM | `Controllers/Scim/*`, `Libraries/Scim/ScimDirectoryService*.cs`, `ScimTokenValidator.cs`; `Parameters/Scim.json` |
| PDF | `Libraries/Pdf/PdfPluginCache.cs` (loads `IPdfPlugin` from `Parameters.ExtendedPlugins`), `PdfPluginHost.cs`; contract in `Implem.Plugins/` |
| Images / thumbnails | `Libraries/Images/ImageData.cs` |
| Binary storage | `Libraries/DataSources/BinaryStorages/` (Local, AzureBlob, factory); `Parameters/BinaryStorage.json` |
| Redis | `Libraries/Redis/` (session cache, data protection); `Parameters/Kvs.json` |
| License / trial | `Libraries/License/LicenseUtilities.cs`; `Libraries/TrialLicenses/TrialLicenseUtilities.cs`; checks in `Implem.ParameterAccessor/Parameters.cs` |
| Site packages (export/import site JSON) | `Libraries/SitePackages/Utilities.cs` (Import/ExportSitePackage), `SitePackage.cs`, `PackageSiteModel.cs` (ID rewrite), `IdConverter.cs`; `Parameters/SitePackage.json` |
| Settings visualizer (Excel/Mermaid) | `Libraries/SiteManagement/Utilities.VisualizeSettings`, `SettingsJsonConverter*.cs`, `Json2XlsxConvertor.cs`, `Json2MermaidConvertor.cs` |
| Initializers | `Libraries/Initializers/` (ExtensionInitializer loads the Extensions table into `Parameters.Extended*`; ItemsInitializer; Users/Tenant/Statuses/Notification) |
| Responses | `Libraries/Responses/ResponseCollection.cs` (client commands, see [frontend.md](frontend.md)), `ResponseSpecials.cs` (gen), `Displays.cs` (gen), `Messages.cs` (gen), `Locations.cs` (URL builders), `ApiResults.cs`/`ApiResponses.cs`, `FileContentResults.cs`, `ResponseFile.cs` |
| Search | `Libraries/Search/Indexes.cs` (gen), `WordBreaker.cs`; `Extensions/FullTextExtensions.cs`, `SearchIndexExtensions.cs` |
| DB execution | `Libraries/DataSources/Repository.cs`; `Rds.cs` (gen). See [data-codegen.md](data-codegen.md) |

## Extended* customization

- **Loading from files:** `Implem.DefinitionAccessor/Initializer.cs` (~113–131 assign, ~576–914 loaders) reads `App_Data/Parameters/Extended*/` recursively. ExtendedSqls/ExtendedServerScripts also read a companion `<file>.json.sql` / `<file>.json.js` into `CommandText` / `Body`.
- **Loading from the DB:** `Libraries/Initializers/ExtensionInitializer.cs` merges Extensions-table rows into the same lists.
- **Reload without restart:** `GET /Admins/ReloadParameters` (`AdminsController.ReloadParameters` → `Libraries/Initializers/ParametersInitializer.cs`, privileged users only, current process only).
- **Extensions CRUD API:** `Controllers/Api/ExtensionsController.cs` (`/api/extensions/...`, needs tenant `AllowExtensionsApi` + privileged user); does not reload.
- **Applicability filter:** `Models/Extensions/ExtensionUtilities.cs` `ExtensionWhere<T>` (~182) and `MeetConditions` (~236). It matches on site, reference, action, controller, user, dept and group.
- **Where each kind is consumed:**

  | Kind | Consumed in |
  |---|---|
  | ExtendedSqls (CRUD hooks) | `Rds.On{Creating,Created,Updating,...,Importing,Imported}ExtendedSqls` (Rds.cs ~5938–6179), called from Issue/Result/Wiki/Dashboard models Create/Update/Delete |
  | ExtendedSqls (view filter/sort) | `View.cs` (OnSelectingWhere ~2168, OnSelectingOrderBy ~3336) |
  | ExtendedSqls (column subquery) | `Rds.OnSelectingColumnSql` (OnSelectingColumn) |
  | ExtendedSqls (permission picker) | `PermissionUtilities` (OnSelectingWherePermissions{Depts,Groups,Users}) |
  | ExtendedSqls (2FA skip) | `UserModel` (OnUseSecondaryAuthentication) |
  | ExtendedSqls (import) | `ImportUtilities` |
  | ExtendedSqls (API) | `/api/extended/sql` → `ExtensionUtilities.Sql` |
  | ExtendedSqls (server script) | `extendedSql` host object |
  | ExtendedSqls (HTML) | `HtmlParts/HtmlSql.cs` |
  | ExtendedScripts | `HtmlParts/HtmlScripts.cs` |
  | ExtendedStyles | `HtmlStyles.cs` |
  | ExtendedHtmls | `HtmlHtmls.cs` (used in HtmlTemplates and HtmlFields) |
  | ExtendedServerScripts | `SiteSettings.GetServerScripts` |
  | ExtendedFields | `Context.cs`, `View.AddExtendedFieldParam` |
  | ExtendedNavigationMenus | `HtmlNavigationMenu.cs` |
  | ExtendedStartGuides | `SiteUtilities` |
  | ExtendedTags["Header"] | `HtmlTemplates.cs` |
  | ExtendedPlugins | PDF |
  | ExtendedColumnDefinitions / ExtendedColumns | Extra ClassA/NumA… columns through `Def.ColumnDefinition(no, ...)` |
