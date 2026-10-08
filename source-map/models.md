# Models, Html rendering, view modes

All paths are relative to `Implem.Pleasanter/`. Line numbers are approximate method-start lines and drift over time. Use them as a hint, then search for the method name inside that one file.

## Generated vs hand-written

- **Generated per table** from `App_Data/Definitions/Definition_Code/Model*.json` + `*_Body.txt` (about 1350 template files). Each table gets:
  - `Models/<Table>/<Model>{Model,Utilities,Collection,Validators,ApiModel,ExportModel}.cs`
  - `Controllers/<Table>Controller.cs`
  - `Models/Shared/_BaseModel.cs` (generated as a single file)
- **Other generated library files:**
  - `Libraries/DataSources/Rds.cs`
  - `Libraries/Responses/{Displays,Messages,ResponseSpecials}.cs`
  - `Libraries/Models/{GridData,Summaries,FormulaUtilities}.cs`
  - `Libraries/HtmlParts/{HtmlGrids,HtmlLinks}.cs`
  - `Libraries/Search/Indexes.cs`
  - `Libraries/General/Error.cs`
  - `Libraries/Settings/{Column,Lookup,LinkAction,Reminder,SiteSettingsUtilities}.cs`
  - `Libraries/Initializers/ItemsInitializer.cs`
- **Editing generated code:** also edit the matching `Definition_Code/Model_*_Body.txt` template, for example `Model_Utilities_Update_Body.txt` or `Model_Utilities_Calendar_Body.txt`. To find the template, search `Definition_Code/` for a distinctive line of the code you are changing.
  - Members tagged `/// Fixed:` are hand-written and survive regeneration.
- **Hand-written files inside `Models/`:**
  - `Issues/IssueBulkUpsertApiModel.cs`, `Results/ResultBulkUpsertApiModel.cs`
  - `Sites/ApiSiteSettings/*`, `Sites/DemoApiModel.cs`, `Sites/SitePackageApiModel.cs`
  - `Tenants/{CreateTenantApiModel,SuspendTenantApiModel,TenantDeleteService}.cs`
  - `Sessions/SessionExclusive.cs`
  - `BackgroundJobs/*` (except the generated model)
  - `SysLogs/SysLogLogModel.cs`, `McpLogs/McpLogLogModel.cs`
  - `Shared/_BaseApiModel.cs`, `Shared/_ImageApiModel.cs`
- `Models/Extensions` is the model for the `Extensions` DB table (extended SQL API). It is not a helper folder.

## Models/<Table> folders

| Folder | Role / key entry points |
|---|---|
| Issues | Period table (Start/Completion/Progress/Status/Manager/Owner). Has Gantt, BurnDown and Separate |
| Results | Record table. Same structure as Issues but without Gantt, BurnDown or Separate. Line numbers run about 100–1200 lower than Issues |
| Wikis | One wiki record per site. Editor, Update, Delete, Restore, History, UnlockRecord |
| Dashboards | Dashboard parts: QuickAccess, TimeLine, Calendar, Kamban, Index, Custom, CustomHtml |
| Sites | Folder/site CRUD, site settings editor, templates, site menu |
| Items | `ItemModel` = dispatcher for `/items/{id}/{action}` (switches on ReferenceType). `ItemUtilities`: title sync |
| Users | Users, login/Authenticate, password, 2FA/TOTP, API key, LDAP sync, SwitchUser |
| Depts / Groups / Tenants | Admin entities. Groups also covers members/children. Tenants also covers BG server scripts and images. `TenantDeleteService` |
| GroupMembers / GroupChildren | `SyncGroupMembers`, `RefreshAllChildMembers` |
| Binaries | Attachments, images, site/tenant images, temp files, storage provider |
| Permissions | Permission dialogs, record permission, column access control |
| Links | Links table (`DestinationId` → `SourceId`) |
| Statuses | System status table (assembly version, monitors). Not the record Status column |
| Orders | Stored sort orders (site menu and others) |
| Sessions | DB-backed session storage. `SessionExclusive` does table locking |
| Extensions | Extensions table: extended SQL API, CRUD by API |
| AutoNumberings | `ExecuteAutomaticNumbering` counters |
| BackgroundJobs | `BackgroundJobQueue`: Enqueue/EnqueueExport/EnqueueImport/Dequeue/Complete/Fail/Cancel |
| ReminderSchedules | `ReminderScheduleUtilities.Remind` |
| Registrations | User self-registration and approval |
| Passkeys | WebAuthn passkey UI and session helpers |
| ScimTokens, TenantQuotaUsages, SysLogs, McpLogs, MailAddresses, LoginKeys | Small helpers |
| OutgoingMails | Mail-to dialog: Editor, Send, SendByApi, AddressBook |
| Parameters | Parameter editor for DB-stored parameters |
| Demos | Demo registration |

### Models/Shared/_BaseModel.cs (generated)

- **`BaseModel`** holds the common columns and extended-column accessors for Class/Num/Date/Description/Check/Attachments (`GetX/SetX/GetSavedX/X_Updated`).
- **`BaseItemModel`** (~871) holds all server-script hooks. Each one calls `ServerScriptUtilities.Execute(condition: ServerScriptConditions.X)`.
  - Loading: `SetByWhenloadingSiteSettingsServerScript`, `SetByWhenloadingRecordServerScript`
  - Formula: `SetByBeforeFormulaServerScript`, `SetByAfterFormulaServerScript`
  - Create and update: `SetByBefore/AfterCreate`, `SetByBefore/AfterUpdate`
  - Delete: `SetByBefore/AfterDelete`, `SetByBefore/AfterBulkDelete`
  - Opening: `SetByBeforeOpeningRow`, `SetByBeforeOpeningPage`

`_BaseApiModel.cs` is the base for `*ApiModel`, with the extended-column dictionaries (ClassHash etc.).

## IssueUtilities.cs (Results has the same methods)

- **Grid:**
  - Index 31, ViewModeTemplate 83, IndexJson 207, GetGridData 239, Grid 250
  - GridRows 311/497 (BeforeOpeningRow SS ~399)
  - GridNewRows 564, ReloadRow 681
  - TrashBox 726, TdValue 782, TdCustomValue 1314
- **Editor:**
  - EditorNew 1435, Editor 1479/1613, EditorInDialog 1569, EditorTabs 1810
  - Fields: FieldSetGeneral 1889, Field 1962
  - Values: ControlValue 2313, ControlRawValue 2452
  - Response: EditorJson 2622, EditorResponse 2653 (BeforeOpeningPage SS, `Process.GetProcess`)
  - EditorMainCommands 2756, GetSwitchTargets 2788 (previous/next record)
  - FieldResponse 2840, NewOnGrid 3121, SelectedIds 3211
- **CRUD:**
  - GetByApi 3267
  - Create 3444 (processes ~3480), CreateByApi 3605, CreateByServerScript 3724
  - Update 3826 (processes ~3846), ResponseByUpdate 3954
  - UpdateByApi 4877, UpsertByApi 5119, BulkUpsertByApi 5428
  - Delete 5848, DeleteByApi 5881
- **Bulk:**
  - BulkUpdate dialog: OpenBulkUpdateSelectorDialog 4113, BulkUpdate 4196/4304 (summary notice; per-record notice when `Notification.NotifyPerRecordOnBulkUpdate`)
  - Inline grid edit: UpdateByGrid 4366 (lock check, SS, notices)
  - BulkProcess 4714
  - BulkMove 6573, BulkDelete 6705/6793, BulkDeleteByApi 6897
  - PhysicalBulkDelete 7165
- **Copy, move, restore, history:**
  - Copy 5674, Move 5772
  - Restore 5973/6020, RestoreFromHistory 6167
  - Histories 6279, History 6401, DeleteHistory 7072
  - Separate 6461
- **Import/Export:**
  - Import 7449, ImportCore 7490, ImportByApi 7937
  - OpenExportSelectorDialog 8679, Export 8705
  - ExportAndMailNotify 8744 (background), ExportByApi 8840, ExportCrosstab 8953
- **View modes:**
  - Calendar 9016 / UpdateByCalendar 9138 / CalendarJson 9188
  - Crosstab 9473, Gantt 9881, BurnDown 10119, TimeSeries 10307
  - Analy 10581, Kamban 10785 / UpdateByKamban 11060, ImageLib 11230
- **Other:**
  - SmartDesignJson 11098, Pdf 11111, UnlockRecord 11185
  - SetLinks 11346, ExistsLockedRecord 11394

## IssueModel.cs (ResultModel has the same methods)

- **Get and display:**
  - Get 864, GetByApi 917
  - ToValue/ToDisplay 961/987, ToApiValue 1299
  - FullText 1465 (search index text)
- **Create** 1622:
  - BeforeCreate SS, AutoNumbering ~1694, Notice ~1709, AfterCreate SS
  - CreateStatements 1819
- **Update** 1959:
  - BeforeUpdate SS, Notice ~2053, UpdateRelatedRecords ~2102, AfterUpdate SS
  - UpdateStatements 2131
- **Attachments and links:**
  - WriteAttachments 2252
  - InsertLinks 2376, UpdateOrCreate 2395
- **Move and delete:**
  - Move 2425, Delete 2464, Restore 2568, PhysicalDelete 2587
- **Binding:**
  - SetDefault 2719, SetCopyDefault 2731
  - SetByForm 2833, SetByFormData 2913, SetByCsvRow 2996
  - CopyAndInit 3176, SetByApi 3236
- **Process, lookups, status control:**
  - SetByProcess 3470, SetByLookups 3604, SetByStatusControls 3656
  - GetProcessMatchConditions 3149, GetStatusControl 3162
- **Summaries and formulas:**
  - SynchronizeSummary 3683, UpdateFormulaColumns 3770
  - SetByFormula 3869, ExecFormulaExtended 3958
- **Title and conditions:**
  - SetTitle 4016
  - Matched 4032 (filter match used by notifications and process conditions)
- **Notifications:** GetNotifications 4249, Notice 4297, NoticeBody 4438
- **Loading:** Set(DataRow) 4690 (WhenloadingRecord SS)

## Items / Sites / others

- **ItemModel.cs** has one method per `/items/{id}/{action}`. Each calls `SetSite()` (~3603) and then switches on `ReferenceType`.
  - Index 307 (ReferenceId 0 → `SiteUtilities.SiteTop`), Calendar 511, Kamban 922, New 1026
  - Editor 1193, Import 1278, Export 1386, SearchDropDown 1472, GetByApi 1659
  - Create 1808, Update 2032, BulkUpdate 2109, UpdateByGrid 2156
  - Copy 2523, Move 2559, Delete 2602
  - ImportSitePackage 3163, ExportSitePackage 3210, Pdf 3396
  - LockTable 3432, SynchronizeSummaries 3520
- **SiteUtilities.cs** (20k lines):
  - CRUD: Create 1063, Update 1148, Copy 1331, Delete 1420
  - API: UpdateSiteSettingsByApi 2394
  - Templates: Templates 2839, CreateByTemplate 3523
  - Site menu: SiteMenuJson 3742, MoveSiteMenu 3788, CreateLink 4008, SortSiteMenu 4131, SiteTop 4720, SiteMenu 4807
  - **Site settings editor:** Editor 5852, EditorTabs 4337. The per-tab `*SettingsEditor`/`*Dialog` methods are:
    - Grid, filters, aggregations: GridSettingsEditor 6574, FiltersSettingsEditor 6953, AggregationsSettingsEditor 7269
    - Editor layout: EditorSettingsEditor 7402, EditorColumnDialog 8121
    - Links: LinksSettingsEditor 9588
    - Summaries and formulas: SummariesSettingsEditor 9813, FormulasSettingsEditor 10264
    - Processes: ProcessesSettingsEditor 10622, ProcessDialog 10841
    - Status controls: StatusControlsSettingsEditor 12189
    - Views: ViewsSettingsEditor 12630, ViewDialog 12719
    - Notifications: NotificationsSettingsEditor 14030
    - AI providers: AiProvidersSettingsEditor 14545
    - Reminders: RemindersSettingsEditor 14860
    - Exports: ExportsSettingsEditor 15340
    - Styles, scripts, HTML, server scripts: StylesSettingsEditor 16323, ScriptsSettingsEditor 16771, HtmlsSettingsEditor 17202, ServerScriptsSettingsEditor 17685
    - Dashboard parts: DashboardPartSettingsEditor 18447
    - Table locks: LockTable 19931
- **SiteModel.cs:**
  - The public `SetSiteSettings` (~2492) calls the private `SetSiteSettings(context, res)` (~3188). The private method is a big `switch (context.Forms.ControlId())` that handles every settings-dialog button.
  - Case ranges in that switch:
    - Grid and filters: ~4235
    - Editor: ~4506
    - Summaries: ~4994
    - Formulas: ~5200
    - Processes: ~5362
    - Status controls: ~6222
    - Views: ~6453
    - Notifications: ~6724
    - AI: ~6994
    - Reminders: ~7307
    - Exports: ~7595
    - Styles, Scripts, HTMLs, Server scripts: ~8039 / ~8230 / ~8412 / ~8596
    - Dashboard parts: ~9229
  - `Upsert*ByApi` methods are at 2530–3134. `WithinApiLimits` is at ~10081.
- **Wikis:** `WikiUtilities` Editor ~528, Update 2020, Delete 2381.
- **Dashboards:** `DashboardUtilities` Index 33, DashboardPartLayout 1461, CalendarLayout 2152, KambanLayout 2433, IndexLayout 2619.
- **Users:**
  - `UserUtilities`:
    - Index 30, Editor 1575, Create 2735, Update 2821
    - Import 3423, Export 4104
    - ChangePassword 4200, ResetPassword 4299
    - HtmlLogin 4591 (login page), SyncByLdap 4899, CreateApiKey 4999, SwitchUser 5217
  - `UserModel`:
    - Authenticate 4669/4754, AllowedIpAddress 4839, UpdateLockout 4918
    - 2FA 5017–5297, Allow 5406, Deny 5550
    - SetFormsAuthentication 5781, ChangePassword 5800, CreateApiKey 6053
- **Depts/Groups/Tenants:**
  - Depts and Groups: `*Utilities` Index/Editor/Create/Update/Delete/Import/Export/*ByApi.
  - Groups members UI: CurrentMembersJson 3307, SelectableMembersJson 3614.
  - Tenants: `TenantUtilities` SetBGServerScript 1980, ServerScriptScheduleDialog 2274.
- **Binaries:**
  - `BinaryUtilities`:
    - Site/tenant images: UpdateSiteImage 257, UpdateTenantImage 303
    - Inline image: UploadImage 423 (markdown/RTE)
    - Download 629, ApiDownload 647
    - Storage: BinaryStorageProvider 773 (DataBase/Local/AutoDataBaseOrLocal)
    - **UploadFile 823** (chunked attachment upload to temp)
    - ValidateFileHash 1123
  - Persistence: `Libraries/DataTypes/Attachments.cs` (Write 165, WriteToLocal 200).
- **Permissions:**
  - `PermissionUtilities`:
    - Site permissions: Permission 33, SetPermissions 768, OpenPermissionsDialog 1044
    - Record permission: PermissionForRecord 1268
    - Column access control: ColumnAccessControl 1943
  - Evaluation logic: `Libraries/Security/Permissions.cs`.
- **Sessions:** `Models/Sessions/SessionUtilities.cs`. View state is in `Libraries/Requests/Views.cs` (GetBySession) and `ViewModes.cs`.

## Libraries used by the models

- `Libraries/ViewModes/`: data/JSON builders for each view mode.
  - Analy*, BurnDown*, Calendar*/FullCalendarElement, Gantt*, KambanElement, TimeSeries*
  - CrosstabUtilities (Where, Csv, Elements, ChoicesX/Y)
- `Libraries/Search/`: `Indexes.cs` (generated) and `WordBreaker.cs`.
  - Global search page: Search 236 / SearchJson 261
  - Index build: Create 24, RebuildSearchIndexes 1021
- `Libraries/Html/`: `HtmlBuilder.cs` (fluent writer), `HtmlAttributes.cs`, `HtmlTypes.cs`.
- `Libraries/HtmlParts/`: HtmlBuilder extension methods.

  | File | Renders |
  |---|---|
  | HtmlTemplates | Page shell `Template` 21, `MainContainer`, `Error` |
  | HtmlGrids (gen) | GridTable, GridHeader, GridHeaderMenus, GridRows, Tr |
  | HtmlFields | One editor field per column type (Text/MarkDown/RTEditor/CodeEditor/DropDown/CheckBox/Radio/Spinner/Slider/Attachments…) |
  | HtmlControls | Raw controls (TextBox, MarkDown, RTEditor, DropDown, Button, Attachments…) |
  | HtmlNavigationMenu | Header menu |
  | HtmlViewFilters | Filter bar |
  | HtmlView / HtmlViewExtensions | View selector |
  | HtmlCommands | Bottom command buttons |
  | HtmlProcess | Process buttons |
  | HtmlComments | Comments |
  | HtmlLinks (gen) / HtmlLinkCreations | Linked tables / link-create buttons |
  | HtmlCalendar, HtmlCrosstab, HtmlGantt, HtmlBurnDown, HtmlTimeSeries, HtmlKamban, HtmlAnaly, HtmlImageLib | View-mode bodies |
  | HtmlScripts, HtmlStyles, HtmlHtmls, HtmlHeadLinks | Script/CSS/custom HTML injection (`ExtendedScripts`) |
  | HtmlDropDownSearches | Dropdown search dialog |
  | HtmlRecordInfo/Header/Switchers, HtmlHistoryCommands, HtmlVerUps, HtmlSeparates, HtmlMoves, HtmlCopies, HtmlImports, HtmlExport, HtmlBulkUpdate, HtmlTrashBoxCommands | Editor parts and dialogs |
  | HtmlSitePackage, HtmlBackgroundJobs, HtmlBreadcrumb, HtmlAggregations, HtmlSiteMenu, HtmlTags | Misc |

- `Libraries/Models/`:
  - `GridData.cs` (gen): grid query, Csv, Json
  - `Summaries.cs` (gen), `FormulaUtilities.cs`
  - `DropDowns.cs` (SearchDropDown / RelatingDropDown / SelectSearchDropDown)
  - `Imports.cs` (CSV validation), `Aggregations.cs`, `ImageLibData.cs`
  - `Versions.cs` (VerUp), `JsonExport*`
- `Libraries/DataTypes/`: column value types (Attachments, Comments, CompletionTime, Num, Status, Time, Title, User, WorkValue…). Each has ToControl/ToResponse/ToDisplay/ToExport/ToNotice/Td.
- `Libraries/Extensions/`:
  - Per-type conversions: ToDisplay, ToControl, ToResponse, ToExport, ToApiValue…
  - `TdExtensions` (grid cells), `MatchExtensions` (condition matching), `FullTextExtensions` (search text)
- `Libraries/General/`: `Error.cs` (gen, `Error.Types`), `Validators.cs`, `Log(s).cs`.

## Feature → where

| Feature | Where |
|---|---|
| Record list grid | IssueUtilities.Index / Grid / GridRows; HtmlGrids.cs; GridData.cs |
| Grid cell value | IssueUtilities.TdValue; Extensions/TdExtensions.cs |
| Filter / sort SQL | Settings/View.cs Where (~1844) / OrderBy (~3330) |
| Filter bar UI | HtmlParts/HtmlViewFilters.cs |
| Grid aggregations | Libraries/Models/Aggregations.cs; HtmlAggregations.cs |
| Editor page | IssueUtilities.Editor → EditorResponse; Field; HtmlFields.cs |
| Bottom buttons | HtmlCommands.cs; IssueUtilities.EditorMainCommands |
| Prev/next record | IssueUtilities.GetSwitchTargets; HtmlRecordSwitchers.cs |
| Create / Update / Delete (UI) | IssueUtilities.Create/Update/Delete → IssueModel.Create/Update/Delete |
| Form → model | IssueModel.SetByForm / SetByFormData |
| Copy / Move / Separate | IssueUtilities.Copy / Move / BulkMove / Separate |
| Trash, restore, history | IssueUtilities.TrashBox / Restore / RestoreFromHistory / Histories |
| CSV import | IssueUtilities.Import/ImportCore; IssueModel.SetByCsvRow; Libraries/Models/Imports.cs |
| Export | IssueUtilities.Export; IssueExportModel; Settings/ExportUtilities.cs |
| Background import/export | ExportAndMailNotify; Models/BackgroundJobs/BackgroundJobQueue.cs |
| Bulk update / delete | IssueUtilities.BulkUpdate / BulkDelete; HtmlBulkUpdate.cs |
| Inline grid edit | IssueUtilities.UpdateByGrid / NewOnGrid |
| Calendar / Kamban drag | UpdateByCalendar / UpdateByKamban |
| Crosstab, Gantt, BurnDown, TimeSeries, Analy, ImageLib | IssueUtilities.<Mode>; ViewModes/<Mode>*.cs; HtmlParts/Html<Mode>.cs |
| Formula | IssueModel.SetByFormula / UpdateFormulaColumns; Settings/Formula*.cs |
| Summaries (cross-site) | IssueModel.SynchronizeSummary; Libraries/Models/Summaries.cs |
| Lookups | IssueModel.SetByLookups; Settings/Lookup.cs |
| Auto numbering | IssueModel.ExecuteAutomaticNumbering; Models/AutoNumberings |
| Notifications trigger | IssueModel.GetNotifications / Notice / NoticeBody |
| Process buttons | IssueModel.SetByProcess; IssueUtilities.Update / BulkProcess; HtmlProcess.cs |
| Status control | IssueModel.SetByStatusControls / GetStatusControl |
| Server script hooks | Models/Shared/_BaseModel.cs (BaseItemModel.SetBy*) |
| Record lock | IssueUtilities.ExistsLockedRecord / UnlockRecord |
| Table lock | SiteUtilities.LockTable; Sessions/SessionExclusive.cs |
| Links | IssueModel.InsertLinks; HtmlLinks.cs; Models/Links |
| Title sync | ItemUtilities.UpdateTitles / UpdateSourceTitles |
| Dropdown search | Libraries/Models/DropDowns.cs; HtmlDropDownSearches.cs |
| Attachments | BinaryUtilities.UploadFile / Download; DataTypes/Attachments.cs |
| API record CRUD | IssueUtilities.*ByApi; IssueApiModel.cs; BulkUpsertApiModel |
| Validation | Models/<T>/<T>Validators.cs; Libraries/General/Validators.cs |
| Full-text search | Search/Indexes.cs; IssueModel.FullText |
| Site create / templates | SiteUtilities.Create / Templates / CreateByTemplate |
| Site settings editor | SiteUtilities.Editor + *SettingsEditor; SiteModel.SetSiteSettings switch |
| Site menu | SiteUtilities.SiteMenu / MoveSiteMenu / SortSiteMenu / CreateLink |
| Login / auth | UserUtilities.HtmlLogin; UserModel.Authenticate / Allow / Deny |
| Password / 2FA / API key | UserUtilities.ChangePassword / ResetPassword / CreateApiKey; UserModel 2FA block |
| Permissions UI | PermissionUtilities |
| Group members | GroupUtilities.*MembersJson; GroupMemberUtilities |
| Tenant BG server scripts | TenantUtilities.SetBGServerScript |
| Mail dialog | OutgoingMailUtilities.Editor / Send |
| PDF | IssueUtilities.Pdf; Libraries/Pdf |
| Comments | HtmlComments.cs; DataTypes/Comments.cs; ItemModel.DeleteComment |
