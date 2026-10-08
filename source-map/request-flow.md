# Request flow: startup, routing, controllers, auth, API

All paths are relative to `Implem.Pleasanter/`. Line numbers are approximate. Search for the method name inside the file.

## Startup

- **`Program.cs`:** NLog → `UseStartup<Startup>()`.
- **`Startup.cs`:**
  - **Constructor (~60):** `Implem.DefinitionAccessor/Initializer.Initialize` loads Parameters and Definitions; `TrialLicenseUtilities.Initialize`.
  - **`ConfigureServices` (~108–348):**
    - CORS, then `AddSession` (`Parameters.Session`).
    - Global filters (~139):
      - `HandleErrorExAttribute`.
      - `AuthorizeFilter`: every action needs auth unless it has `[AllowAnonymous]`.
      - `CheckContextAttributes`.
      - `RequireHttps` (optional).
    - Auth (~150):
      - With SAML: Cookie + `AddSaml2` (`Libraries/DataSources/Saml.cs`).
      - Otherwise Cookie only, LoginPath `/users/login`.
      - `ITicketStore` is `Libraries/Security/AuthenticationTicketStore.cs` (DB session).
    - ExtendedLibraries DLL loading (~201).
    - MCP: `ConfigureMcpServer` (~368).
    - Rate limiters (~394).
    - Health checks (~230).
    - Hosted services: `ApplicationWarmupHostedService` and `CustomQuartzHostedService`.
    - DataProtection (~273: Azure, Redis or DB).
    - Antiforgery header `X-CSRF-TOKEN`.
    - Captcha (`Libraries/Security/Captcha/*`).
    - `AddFido2` for passkeys.
  - **`Configure` middleware order (~481–800):**
    1. TrustedProxySourceIpCapture
    2. ForwardedHeaders
    3. NoCacheHtml
    4. CurrentRequestContext (`AspNetCoreHttpContext.Current`, which `Context` relies on)
    5. ExceptionHandler `/errors/internalservererror`
    6. HSTS
    7. SecurityHeadersMiddleware (class at ~983)
    8. CSP nonce (`HttpContext.Items["Nonce"]`)
    9. Global exception → SysLog (~819)
    10. StatusCodePages (~528 → `/errors/*`; SCIM and ajax get JSON)
    11. StaticFiles
    12. SCIM swagger
    13. Routing
    14. CORS
    15. WarmupGate
    16. MCP middlewares
    17. OutputCache
    18. Session
    19. Authentication
    20. TrustedProxyAuthentication
    21. Authorization
    22. RateLimiter
    23. `SessionMiddleware` (~897: new session → ClientId cookie, Windows-auth LDAP upsert)
    24. Endpoints
  - **Conventional routes (~710–794), in this order:**

    | Name | Pattern |
    |---|---|
    | FormBinaries | `{reference}/{guid}/{controller}/{action}` |
    | Default | `{controller}/{action}` (Items/Index) |
    | Others | `{reference}/{id}/{controller}/{action}` (Binaries, PublishBinaries, OutgoingMails) |
    | **Item** | `{controller}/{id}/{action}` (numeric id) → **`/items/{id}/{action}`** |
    | Binaries | `{controller}/{guid}/{action}` |
    | BinariesUpload | `binaries/upload` |

  - **Other mappings:** API and SCIM use attribute routing. `MapMcp("/mcp")` is mapped when MCP is enabled.

## Filters / Middlewares

- `Filters/CheckContextAttributes.cs` (global, `OnAuthorization`):
  - Builds `new Context()`.
  - Checks:
    - parameter syntax error
    - IP allow-list (`IpAddresses.AllowedIpAddress`, skipped for `/api/` and `/scim/`)
    - tenant IP and contract deadline
    - form `Token` (CSRF)
    - signs out when the user does not exist
  - Then `SiteInfo.Refresh`.
- `Filters/CheckApiContextAttributes.cs`: on every `Controllers/Api/*` action. Reads the body, builds `Context(apiRequestBody)`, checks IP, rejects bad JSON with 400, and checks the Token when cookie-authenticated.
- `Filters/CheckScimContextAttributes.cs`: SCIM bearer token.
- `Filters/ConditionalValidateAntiForgeryTokenAttribute.cs`: public forms. `FormsAttributes.cs` / `PublishesAttributes.cs` gate those controllers.
- `Filters/HandleErrorExAttribute.cs`: exception → SysLog.
- `Middlewares/TrustedProxyAuthenticationMiddleware.cs` (header sign-in), `WarmupGateMiddleware.cs`, `NoCacheHtmlMiddleware.cs`.
- `Tools/BackgroundTasks.cs` + `*.py`: external cron scripts that call `/backgroundtasks/do`, `/reminderschedules/remind` and `/users/syncbyldap`.

## Context (`Libraries/Requests/Context.cs`)

The per-request object. It holds:
- request data (Forms, QueryStrings, PostedFiles, Controller, Action, Id)
- tenant, site, user and depts/groups
- language and timezone
- PermissionHash
- session data
- API data (`ApiRequestBody`, `RequestDataString`, `ApiKey`)
- flags: Publish, IsForm, ServerScriptDepth

`Set()` (~229) runs these steps in order:
1. `SetRequests` (reads `AspNetCoreHttpContext.Current`)
2. `SetSessionGuid` (~331)
3. `SetItemProperties` (~409, looks up the `Items` row to get SiteId)
4. **`SetUserProperties` (~481):** when the body has an `ApiKey`, the user comes from `Users.ApiKey`; otherwise from the cookie LoginId. Switch-user is handled here.
5. `SetTenantProperties` (~583)
6. `SetPublish` (~723)
7. `SetPermissions` (~909)

Other members: `Token()` (~1433, CSRF), `FormsAuthenticationSignIn/SignOut` (~1327).

Request JSON models (`Libraries/Requests/`):
- `Api.cs`: ApiKey, View, Offset, PageSize, …
- `ExportApi.cs`, `ImportApi.cs`, `RecordSelector.cs`, `SessionApi.cs`, `ExtendedApi.cs`
- Also in this folder: `Forms.cs`, `QueryStrings.cs`, `Views.cs` (current View in session), `ViewModes.cs`.

## Controller pattern

Every action follows the same pattern:
1. `var context = new Context(); var log = new SysLogModel(context);`
2. Delegate the work.
3. `log.Finish(...)`.

What an action returns:
- **HTML GET:** sets `ViewBag.HtmlBody` and returns `View()`. The Razor views in `Views/*` just output that string.
- **Ajax:** returns the JSON of a `ResponseCollection`.

### ItemsController (`Controllers/ItemsController.cs`), URL `/items/{id}/{action}`

Almost every action does `new ItemModel(context, referenceId: id).<Action>(context)`. `ItemModel` (`Models/Items/ItemModel.cs`) switches on `ReferenceType` and calls `SiteUtilities` / `IssueUtilities` / `ResultUtilities` / `WikiUtilities` / `DashboardUtilities`.

| Controller action (line) | Notes |
|---|---|
| Index (~30) | id=0 → `SiteUtilities.SiteTop`; Sites → SiteMenu; Dashboards; Issues/Results → Index |
| Calendar/Crosstab/Gantt/BurnDown/TimeSeries/Analy/Kamban/ImageLib (~121–293) | Gantt and BurnDown are Issues only |
| New (~316), Edit (~400), NewOnGrid, SmartDesign/UpdateSmartDesign (~369/386) | |
| Import (~444) / Export (~567) | Large jobs → `Models/BackgroundJobs/BackgroundJobQueue.cs` (EnqueueImport/Export) |
| Search (~665) | `Libraries/Search/Indexes.Search` (not ItemModel) |
| SearchDropDown/RelatingDropDown/SelectSearchDropDown (~689) | `Libraries/Models/DropDowns.cs` |
| GridRows/ReloadRow/CopyRow (~722–769) | |
| Create (~789), Update (~839, PUT), BulkUpdate, UpdateByGrid, BulkProcess (~855–887) | |
| Copy, MoveTargets, Move, BulkMove, Delete (DELETE), BulkDelete, PhysicalDelete, Restore, RestoreFromHistory, Histories, History, Separate | |
| MoveSiteMenu/CreateLink/SortSiteMenu (~932–962) | `SiteUtilities` directly |
| SetSiteSettings (~1076) | `SiteUtilities.SetSiteSettings` → `SiteModel.SetSiteSettings` switch |
| Import/ExportSitePackage (~1088–1133) | `Libraries/SitePackages/` |
| VisualizeSettings, RebuildSearchIndexes | |
| Permissions… / ColumnAccessControl (~1222–1351) | `Models/Permissions/PermissionUtilities.cs` directly |
| UpdateByCalendar/UpdateByKamban, BurnDownRecordDetails | |
| Synchronize Titles/Summaries/Formulas (~1390) | `SiteUtilities.Synchronize*` |
| LockTable/UnlockTable/ForceUnlockTable/UnlockRecord (~1422–1467) | |
| UserTemplate actions, Export/ImportMultilingualLabels | |

Permission and validation checks per table live in `Models/<T>/<T>Validators.cs`: OnEntry, OnGet, OnEditing, OnCreating, OnUpdating, OnMoving, OnDeleting, OnRestoring, OnImporting, OnExporting. The checks they call are in `Libraries/Security/Permissions.cs`.

### Other MVC controllers

| Controller | Delegates to |
|---|---|
| UsersController | `UserUtilities`/`UserModel`: Login (~305), Authenticate (~413), Logout (~434), Challenge/SamlLogin (~278/331), SyncByLdap (~534), SwitchUser (~596), password, API key, 2FA |
| DeptsController / GroupsController | `DeptUtilities` / `GroupUtilities` (CRUD, Import, Export, members) |
| TenantsController | `TenantUtilities`, `ScimTokenUtilities` |
| AdminsController | Admin index, InitializeItems, MigrateSiteSettings, ReloadParameters |
| BinariesController | `BinaryUtilities` (Upload chunked, Download, images) |
| FormsController / FormBinariesController | Public forms `/forms/{guid}/...` (captcha, AllowAnonymous) |
| PublishesController / PublishBinariesController | Anonymous published sites |
| OutgoingMailsController | `OutgoingMailUtilities` (`/{reference}/{id}/outgoingmails/{action}`) |
| PasskeysController | FIDO2 → `PasskeyUtilities`, `UserModel.Authenticate(isAuthenticationByPasskey)` |
| RegistrationsController, DemosController, ParametersController | `RegistrationUtilities`, `DemoUtilities`, `ParameterUtilities` |
| SysLogsController, McpLogsController, BackgroundJobsController | Log and job grids |
| BackgroundTasksController, ReminderSchedulesController | AllowAnonymous cron endpoints |
| ErrorsController, ResourcesController, CspReportController, VersionsController | |
| Api_ItemsController / Api_Users / Api_Depts / Api_Groups | **Legacy** API through the conventional route (`/api_items/{id}/get`) |

## REST API (`Controllers/Api/`)

- **Attributes:** `[CheckApiContextAttributes][AllowAnonymous][ApiController][Route("api/[controller]")][EnableRateLimiting("Api")]`.
- **Verbs:** every action is POST.
- **Action flow:** read the body → `new Context(apiRequestBody, api: true)` → if authenticated, call the delegate, otherwise `ApiResults.Unauthorized`.
- **Endpoints:**

  | URL `POST /api/...` | Delegate |
  |---|---|
  | `items/{id}/get|create|update|upsert|delete|bulkdelete|bulkupsert|import|export` | `ItemModel.*ByApi` → `IssueUtilities`/`ResultUtilities.*ByApi` |
  | `items/{id}/getsite|createsite|updatesite|deletesite|copysitepackage|synchronizesummaries|updatesitesettings|getclosestsiteid` | `ItemModel` / `SiteUtilities` |
  | `users|depts|groups/[{id}/]get|create|update|delete|import` | `UserUtilities`/`DeptUtilities`/`GroupUtilities.*ByApi` |
  | `binaries/{guid}/get|getstream`, `binaries/upload` | `BinaryUtilities.ApiDownload`, chunked upload |
  | `sessions/get|set|delete` | `SessionUtilities.*ByApi` |
  | `tenants/...` | `TenantUtilities` |
  | `extensions/...`, `extended/sql` | `ExtensionUtilities` (ExtendedSql) |
  | `{reference}/{id}/outgoingmails/send` | `OutgoingMailUtilities.SendByApi` |
  | `backgroundtasks/rebuildsearchindexes`, `demo/register`, `utility/getlicenseinfo` | |

- **Request bodies:**
  - `Libraries/Requests/Api.cs` (+Export/Import) is the common envelope.
  - Record bodies: `Models/<T>/<T>ApiModel.cs`, `*BulkUpsertApiModel.cs`, `Models/Shared/_BaseApiModel.cs` (ClassHash/NumHash/...).
  - Sites: `Models/Sites/ApiSiteSettings/*`.
- **Responses:** `Libraries/Responses/ApiResults.cs`, `ApiResponses.cs`, `ContentResultInheritance.cs` (`ToHttpResponse`).
- **Per-site API limit:** `SiteModel.WithinApiLimits`, `Libraries/Settings/ContractSettings.cs`.

## SCIM (`Controllers/Scim/`)

- **Users:** `UsersController` (`scim/v2/Users`) handles List/Get/Create/PUT/PATCH/Delete. `GroupsController` (`scim/v2/Groups`) handles the same for groups.
- **Metadata:** `ScimMetadataController` serves ServiceProviderConfig, ResourceTypes and Schemas.
- **Logic:** `Libraries/Scim/ScimDirectoryService*.cs` (partial class: Filters, Patch, Mapping, Members, Validation, …), `ScimTokenValidator.cs`.
- **Tokens:** `Models/ScimTokens/`.
- **Config:** `App_Data/Parameters/Scim.json`.

## Authentication and security

- **Providers:** Forms/local, LDAP, LDAP+Local, Windows, SAML / SAML-MultiTenant, Extension, TrustedProxy header and Passkeys. **There is no OIDC/OpenID.**
- `Libraries/Security/Authentications.cs`: `SignIn` (→ `UserModel.Authenticate`), `SignOut`, `SAML()`.
- **`Models/Users/UserModel.cs` `Authenticate` (~4669)** runs these steps in order:
  1. Disabled user.
  2. Credential check (`Authenticate(context)` ~4754 switches on `Parameters.Authentication.Provider`):
     - LDAP: `Libraries/DataSources/Ldap.cs` → `LdapDs.cs` / `LdapNovellDs.cs`.
     - Extension: `DataSources/Extension.cs`.
     - Local: `GetByCredentials`.
  3. Login expiry.
  4. Tenant IP.
  5. Lockout (`UpdateLockout` ~4918).
  6. 2FA (mail/TOTP, ~5017–5780).
  7. Password expiry.
  8. `Allow` → `SetFormsAuthentication`.
- **SAML:** `Libraries/DataSources/Saml.cs` (MapAttributes, UpdateOrInsert, SetSPOptions, RegisterSamlConfiguration, SamlLogin). Per-tenant IdP settings come from `ContractSettings`.
- **Passkeys:** `Controllers/PasskeysController.cs`, `Models/Passkeys/`, `Libraries/Settings/PasskeyData.cs`.
- **Password policy:** `Implem.ParameterAccessor/Parts/PasswordPolicy.cs`, enforced in `UserUtilities` (Create/Update/ChangePassword/ResetPassword) and `UserValidators`. History: `Libraries/Settings/PasswordHistory.cs`.
- **IP restriction:** `Libraries/Security/IpAddresses.cs` + `Libraries/Server/IpRange.cs`.
- **Permissions:** `Libraries/Security/Permissions.cs` (CanRead/CanCreate/CanUpdate/CanDelete/CanImport/CanExport/CanManageSite/CanManageTenant…, `SetPermissionsWhere`).
- **Rate limit:** `Libraries/RateLimit/BodyRateLimitHelper.cs` (policies General, List, Admin, Heavy, ApiHeavy, Api, AnonymousIp, PublicForm) and `[EnableRateLimiting]` on controllers. Config: `App_Data/Parameters/RateLimit.json`. MCP has its own limiter in `MCP/Infrastructure/McpRateLimit*.cs`.
- **Caches (`Libraries/Server/`):**
  - `SiteInfo.cs`: static caches for users, depts, groups and SiteSettings.
  - `TenantCache.cs` / `UpdateMonitor.cs`.
  - `DistributedLock.cs`.
  - `SiteMenu.cs`: site tree.
  - `Times.cs`: timezone helpers.

## Feature → where

| Feature | Where |
|---|---|
| Middleware order / DI | Startup.cs Configure / ConfigureServices |
| Routes | Startup.cs ~710 |
| Error pages | Startup.cs StatusCodePages; Controllers/ErrorsController.cs |
| CSP / security headers | Startup.cs; Implem.ParameterAccessor/Parts/ContentSecurityPolicy.cs |
| CSRF token | Context.Token(); Filters/CheckContextAttributes.cs |
| Login page / login POST | UsersController.Login/Authenticate; UserUtilities.HtmlLogin; UserModel.Authenticate |
| LDAP | DataSources/Ldap*.cs; BackgroundServices/SyncByLdapExecutionTimer.cs |
| SAML | Startup.cs ~150; DataSources/Saml.cs |
| Windows auth | Context.AuthenticationsWindows; Startup SessionMiddleware |
| Trusted proxy header auth | Middlewares/TrustedProxyAuthenticationMiddleware.cs |
| 2FA / TOTP | UserModel ~5017–5780; Parts/SecondaryAuthentication.cs |
| API key auth | Context.SetUserProperties; UserUtilities.CreateApiKey |
| Session storage | Models/Sessions/SessionUtilities.cs; Security/AuthenticationTicketStore.cs |
| Public forms | FormsController; Filters/FormsAttributes.cs |
| Published sites | PublishesController; Context.SetPublish |
| Warmup | Middlewares/WarmupGateMiddleware.cs; BackgroundServices/ApplicationWarmupHostedService.cs |
| Health check | Startup.cs; Parts/HealthCheck.cs |
| Switch user | UserUtilities.SwitchUser; UsersController ~596 |
