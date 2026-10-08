# App_Data configuration checklist for a new project

This is the order in which to review `Implem.Pleasanter/App_Data/Parameters/*.json` when you set up a new Pleasanter environment (local dev, staging or production). Defaults in the repo are tuned for a local demo. Several of them are unsafe or wrong for a real deployment.

## 0. How parameters are loaded (read this first)

- At startup, `Implem.DefinitionAccessor/Initializer.cs` (`SetParameters`) reads each `<Name>.json` into `Implem.ParameterAccessor/Parts/<Name>.cs`. Code reads them as `Parameters.<Name>`.
- Most files are **required**. A missing or broken JSON file stops startup (`ParametersNotFoundException` / `ParametersIllegalSyntaxException`). Optional ones: `BackgroundJobs`, `Quartz`, `MultiTenant`, `Scim`, `AutoTestSettings`, `Env`.
- **Env.json** (not in the repo, optional): `{ "ParametersPath": "/some/dir" }` makes the app read every parameter file from another directory. Use it to keep per-environment config outside the source tree.
- **Environment variables override secrets.** The prefix is `Service.Name` (default `Implem.Pleasanter`) or `Service.EnvironmentName` if set. Most-used variables:

  | Variable | Overrides |
  |---|---|
  | `<Name>_Rds_<Dbms>_SaConnectionString` / `_OwnerConnectionString` / `_UserConnectionString` | `Rds.*ConnectionString` |
  | `<Name>_Rds_ConnectionString` | fallback for all three |
  | `<Name>_Rds_MySqlConnectingHost` | `Rds.MySqlConnectingHost` |
  | `<Name>_Mail_SmtpUserName`, `_Mail_SmtpPassword`, `_Mail_OAuthClientId`, `_Mail_OAuthClientSecret`, `_Mail_SendGrid_ApiKey`, `_Mail_AwsSes_*` | `Mail.*` |
  | `<Name>_Security_AspNetCoreDataProtection_XmlAesKey` / `_BlobContainerUri` / `_KeyIdentifier` / `_KeyValueStoreConnectionString` | `Security.AspNetCoreDataProtection.*` |
  | `<Name>_BinaryStorage_AzureBlobStorageAccountUri` / `_AzureBlobContainerName` | `BinaryStorage.*` |
  | `<Name>_Kvs_ConnectionStringForSession` | `Kvs.ConnectionStringForSession` |
  | `<Name>_HtmlUrlPrefix` | `General.HtmlUrlPrefix` |
  | `TRUSTED_PROXY_AUTH_ENABLED`, `TRUSTED_PROXY_AUTH_HEADER` | `Authentication.TrustedProxyParameters` |

  Keep secrets in environment variables, not in committed JSON.
- **DB patches.** After the files load, `PatchParameters()` looks up a row in the `Parameters` table (`Title` = file name) and applies its `Body` as a JSON patch. So the value that is actually used can differ from the file. Check that table when a setting "doesn't take effect". `Rds`, `Migration` and `ParameterSetting` cannot be patched this way.
- `ParameterSetting.json`: `EnableScreenManagement` turns on editing parameters from the admin UI, and `EnableRestart` allows restarting the app from it. Leave both `false` unless you need them.

## 1. Database: `Rds.json` (must be done first)

| Key | Default | Check |
|---|---|---|
| `Dbms` | `PostgreSQL` | `SQLServer`, `PostgreSQL` or `MySQL`. It also selects which env variable names are read and which `Definitions/Sqls/<Dbms>` SQL set is used. |
| `SaConnectionString` | `...PWD=SetSaPWD` | Admin connection. CodeDefiner `_rds` uses it to create the DB and roles. Put it in an env variable. |
| `OwnerConnectionString` / `UserConnectionString` | `#ServiceName#_Owner` / `_User`, `PWD=SetAdminsPWD` / `SetUsersPWD` | `#ServiceName#` becomes `Service.Name`. **Change the placeholder passwords.** `CodeDefiner _rds /s /r` asks for the SA password and generates random Owner/User passwords. |
| `MySqlConnectingHost` | `%` | MySQL only: host part of the created users. Narrow it in production. |
| `SqlCommandTimeOut` | `0` | `0` = no timeout. Consider a limit in production. |
| `DeadlockRetryCount` / `DeadlockRetryInterval` | `4` / `1000` ms | Usually fine. |
| `DisableIndexChangeDetection` | `true` | When `true`, `_rds` doesn't rebuild indexes that differ from the definitions. |

After this file is set, run:

```bash
dotnet run --project Implem.CodeDefiner -- _rds /l "ja" /z "Asia/Tokyo" /y
```

`/l` and `/z` are written back into `Service.json` (`DefaultLanguage`, `TimeZoneDefault`).

`Migration.json` is only for `CodeDefiner migrate`, which copies data from another DB (`SourceConnectionString`). Ignore it for a fresh project.

## 2. Service identity: `Service.json`

| Key | Default | Check |
|---|---|---|
| `Name` | `Implem.Pleasanter` | Used as the DB name/role prefix, the env variable prefix and the data protection application name. **Decide it before the first `_rds`.** Changing it later points the app at a different DB. |
| `EnvironmentName` | `null` | Optional extra env variable prefix (e.g. `prod`). |
| `TimeZoneDefault` | `UTC` | Default time zone for new users. Set it with `_rds /z` or here. |
| `DefaultLanguage` | `en` | `en`, `ja`, `zh`, `de`, `ko`, `es`, `vn`. |
| `DefaultPassword` | `pleasanter` | Initial password of `Administrator` and of users created without a password. **Change it.** |
| `WithoutChangeDefaultPassword` | `false` | Keep `false` so users must change the default password at first login. |
| `AbsoluteUri` | `null` | Public base URL. Set it when running behind a proxy or when reminders and background jobs build links (they have no HTTP request to take the host from). |
| `RequireHttps` | `false` | `true` in production (redirects to HTTPS). |
| `MaxRequestBodySize` | `30000000` | Upload size limit in bytes. Keep it consistent with `BinaryStorage` limits and the reverse proxy limit. |
| `ShowStartGuide`, `ShowProfiles`, `ShowChangePassword`, `AnnouncementSiteId` | | UI options. |
| `Demo` | `false` | Must stay `false` unless you run a demo site. |

`MultiTenant.json`: `DefaultTenantId` (default `1`). `Locations.json`: `TopUrl`, `LoginAfterUrl` if users should land somewhere other than the top page.

## 3. Security: `Security.json`

Defaults are permissive. Review every item for production.

- **Password policy:** `PasswordPolicies` (only "6+ characters" is enabled by default; enable the lowercase/uppercase/digit/symbol rules as required), `EnforcePasswordHistories`, `PasswordExpirationPeriod`, `LockoutCount` (`0` = no lockout).
- **Privileged users:** `PrivilegedUsers` (list of login IDs with system admin rights). When this is patched from the DB, the arrays are merged, not replaced.
- **IP restriction:** `AllowIpAddresses`, `IpRestrictionExcludeMembers`.
- **Cookies/headers:** `SecureCookies` (`true` with HTTPS), `HttpStrictTransportSecurity.Enabled`, `SecureCacheControl`, `DisableMvcResponseHeader`, `AccessControlAllowOrigin` (CORS).
- **Content Security Policy:** `ContentSecurityPolicy.Enabled` / `ReportOnlyEnabled`. Start in report-only mode and check `/CspReport/Report` before you enforce it. Custom scripts and styles often break under CSP.
- **Two-factor auth:** `SecondaryAuthentication.Mode` (`None` by default). It needs working mail (section 6) when `NotificationType` is `Mail`.
- **Data protection keys** (`AspNetCoreDataProtection`): auth cookies and encrypted values depend on these keys.
  - Default (nothing set): keys are stored in the DB and encrypted with a built-in default AES key. **Set your own `XmlAesKey`** (via env variable).
  - Redis: set `KeyValueStoreConnectionString` (+ `KeyValueStoreKeyName`).
  - Azure: set `BlobContainerUri` + `KeyIdentifier` (Key Vault).
  - Every instance of the same service must share the same keys, or users get logged out when they hit another instance.
- **Reverse proxy:** `ForwardedHeaders.KnownNetworks` / `KnownProxies` so client IPs and scheme are correct (this matters for IP restriction, logs and rate limiting).
- **Health check:** `HealthCheck.Enabled` (`/healthz`). Set `EnableDatabaseCheck` if the load balancer should check the DB, and `RequireHosts` to limit who can call it.
- `CaptchaConfig.Type`, `TokenCheck`, `MimeTypeCheckOnApi`, `JoeAccountCheck` (rejects passwords equal to the login ID).

`RateLimit.json`: `Mode` is `Off` by default. Turn it on for internet-facing deployments. The policies (`General`, `List`, `Heavy`, `Api`, `AnonymousIp`, `PublicForm`, …) are already defined.

`Form.json`: public forms are disabled (`Enabled: false`). If you enable them, review `AttachmentExcludedExtensions`.

## 4. Authentication: `Authentication.json`

| Key | Check |
|---|---|
| `Provider` | `null` = form login with Pleasanter accounts. Other values switch to LDAP, SAML, etc. |
| `RejectUnregisteredUser` | Reject external-auth users that don't exist in Pleasanter. |
| `LdapParameters[]` | Search root, account/attribute mapping, `LdapTenantId`, sync user/password, `AutoDisable`/`AutoEnable`. Scheduled sync is enabled with `BackgroundService.SyncByLdap`. |
| `SamlParameters` | `SPOptions.EntityId` / `ReturnUrl` default to `http://localhost:59802`. **Change them to the real URL.** Also set `IdentityProviders[]`, `ServiceCertificates` and the attribute mapping. |
| `PasskeyParameters` | `ServerDomain` / `Origins` default to localhost. |
| `TrustedProxyParameters` | Header-based SSO behind an authenticating proxy. Only enable it when the app cannot be reached except through that proxy. |

Related: `Registration.json` (self sign-up, off by default), `User.json` (`DisableTopSiteCreation`, `DisableGroupCreation`, `DisableApi`, default `Theme`), `Scim.json` (SCIM provisioning, off by default).

## 5. Session and scale-out: `Session.json`, `Kvs.json`

- `Session.RetentionPeriod` (minutes, default 1440).
- For more than one app instance, set `Session.UseKeyValueStore: true` and `Kvs.ConnectionStringForSession` (Redis). Also share data protection keys (section 3) and enable `Quartz.Clustering` (section 7).

## 6. File storage: `BinaryStorage.json`

| Key | Default | Check |
|---|---|---|
| `Provider` | `Rds` | `Rds` stores attachments in the DB. `Local` + `Path` stores them on disk. Azure Blob is set with `AzureBlobStorageAccountUri` / `AzureBlobContainerName`. Large files in the DB make the DB grow fast. |
| `Path` | `null` | Required for `Local`. Must be a persistent volume in containers. |
| `UseStorageSelect` / `DefaultBinaryStorageProvider` | `false` / `DataBase` | Lets each column choose DB or local folder storage. |
| `LimitSize`, `LimitTotalSize`, `LimitQuantity`, `LocalFolder*` | MB / count | Default limits for attachment columns. |
| `BrowserAllowMimeTypes` | pdf, txt, images | Types that open in the browser instead of downloading. |

## 7. Mail, notifications and background jobs

**`Mail.json`**: nothing is sent until this is set. This affects notifications, reminders, password reset and 2FA codes.
- `Provider` (`Smtp`, `SendGrid`, `AwsSes`), `SmtpHost`, `SmtpPort`, `SmtpEnableSsl` / `SecureSocketOptions`, credentials (env variables), or `UseOAuth` + `OAuth*`.
- `FixedFrom` / `AllowedFrom` (sender address control), `SupportFrom` (default is pleasanter.org; **change it**), `InternalDomains`.

**`Notification.json` / `Reminder.json`**: enable or disable channels (Mail, Slack, Teams, ChatWork, LINE, …). `Notification.HttpClient` is off by default.

**`BackgroundService.json`**: every scheduled job is **off by default**:
- `Reminder` (otherwise scheduled reminders never fire)
- `SyncByLdap`, `DeleteSysLogs`, `DeleteTemporaryFiles`, `DeleteTrashBox` (+ retention), `DeleteUnusedRecord`, `DeleteMcpLogs`, `DeleteTenant`, `DeleteBackgroundJobs`
- Each job has a `*Time` list (`"HH:mm"`). `EnvironmentVariables` passes env variables to these jobs.

**`BackgroundJobs.json`** (`BackgroundQueue`, `WorkerCount`, `OutputFilePath`/`InputFilePath` for large import/export) and **`Quartz.json`** (`Clustering.Enabled` when more than one instance runs). `BackgroundTask.json` controls the in-process task loop (search index creation).

## 8. Logging and data retention

- `SysLog.json`: `EnableLoggingToDatabase` (default on), `EnableLoggingToFile`, `RetentionPeriod` (days; deletion only runs when `BackgroundService.DeleteSysLogs` is on), `LoginSuccess` / `LoginFailure` / `SignOut` audit logging (off by default; turn on for audit requirements), `OutputErrorDetails` (**set `false` in production** so stack traces are not shown to users), `NotLoggingIp`.
- File log format and paths are in `Implem.Pleasanter/appsettings.json` (NLog, writes under `logs/`).
- `Deleted.json` / `History.json`: whether deleted records and history can be restored or physically deleted.
- `General.DeleteHistoriesOldThan`, `General.DeleteTempOldThan` (minutes).

## 9. Optional features (off unless you enable them)

| File | What it enables | Check before you enable it |
|---|---|---|
| `Api.json` | REST API (`Enabled: true` by default) | `PageSize`, `LimitPerSite`. Also `User.DisableApi`. |
| `Script.json` | Server scripts (`ServerScript: true`) | `ServerScriptTimeOut`, `DisableServerScriptHttpClient` (outbound HTTP from scripts), `BackgroundServerScript`, `DisableServerScriptFile`. |
| `McpServer.json` | MCP server (`/mcp`) for AI clients | `ReadOnlyMode`, `RateLimit.*`, logging. |
| `AiConnect.json` | Sync records to OpenAI / Dify knowledge bases | `ProviderEndpoints`, `Rag.Enabled`. |
| `Scim.json` | SCIM user/group provisioning | `ExtendedAttributes`. |
| `Search.json` | Full-text search | `SearchDocuments` (index attachments), `CreateIndexes`. |
| `SitePackage.json` | Site package import/export | `IncludeDataOnImport`/`Export`, permission/notification options. |
| `CustomApps.json` | User app templates | `CustomAppsMax`. |
| `Mobile.json` | Responsive UI, mobile camera | |

## 10. UX defaults that are hard to change later

`General.json` has many grid/calendar/kanban limits, which can be tuned at any time. These ones affect data or habits, so decide them at the start:

- `FirstDayOfWeek` (default `1` = Monday), `FirstMonth` (default `4` = fiscal year starting in April, the Japanese convention).
- `ImportEncoding` (default `Shift-JIS`; change it to `UTF-8` for non-Japanese users), `UpdatableImport`, `ImportMax`, `BulkUpsertMax`.
- `CharToAddWhenCopying` (default `" - コピー"`, Japanese).
- `HtmlLogoText`, `HtmlHeadDescription`, `HtmlPortalUrl`, `HtmlContactUrl` (branding).
- `MinTime` / `MaxTime`, `GridPageSize`.

Other: `Permissions.json` (default permission bit patterns for General/Manager), `Site.json` (`SimpleMode`), `Validation.json` (`MaxLength`), `Version.json` (what the About page shows), `NavigationMenus.json` (top menu).

## 11. Customization folders (`Parameters/Extended*`)

These let you customize a deployment without changing the source. Files ending in `.txt` are samples and are ignored. Copy one and remove `.txt` to activate it. All files are read recursively at startup. The `Extensions` DB table can supply the same items at runtime (`ExtensionInitializer`).

| Folder | Content |
|---|---|
| `ExtendedScripts` / `ExtendedStyles` / `ExtendedHtmls` | Global client JS, CSS and HTML injected into pages. |
| `ExtendedServerScripts` | Server scripts applied to sites without editing each site's settings. |
| `ExtendedSqls` | SQL run on events or called from server scripts. Write it for the `Dbms` you selected. |
| `ExtendedFields` | Extra fields on user/dept/group forms. |
| `ExtendedNavigationMenus` | Extra items in the top menu. |
| `ExtendedPlugins` | DLL plugins (e.g. PDF). |
| `ExtendedStartGuides` | Start guide content. |
| `ExtendedColumnDefinitions` | Templates for the extended column types (Class, Num, Date, Description, Check, Attachments). They affect the DB schema, so re-run `_rds` after a change. |
| `ExtendedAutoTest` | Scenarios for the `Implem.TestAutomation` E2E runner (`AutoTestSettings.json`). |

`App_Data/Licenses/Implem.License.dll` is the license DLL for Enterprise features. Replace it if the project has a license.

## Quick checklist

1. [ ] Decide `Service.Name` (and `EnvironmentName`). Create `Env.json` if config lives outside the repo.
2. [ ] `Rds.json`: `Dbms` + connection strings in env variables. Replace the placeholder passwords.
3. [ ] Run `CodeDefiner _rds /l <lang> /z <tz> /y` (add `/s /r` to enter the SA password and generate Owner/User passwords).
4. [ ] `Service.json`: `DefaultPassword`, `AbsoluteUri`, `RequireHttps`, language/time zone.
5. [ ] `Security.json`: password policy, lockout, `PrivilegedUsers`, `SecureCookies`, HSTS, own `XmlAesKey`, `ForwardedHeaders`, health check.
6. [ ] `SysLog.OutputErrorDetails: false`, audit log flags, `RateLimit.Mode`.
7. [ ] `Authentication.json` if you use LDAP/SAML/passkeys (replace the localhost URLs).
8. [ ] `BinaryStorage.json`: storage provider, path or blob, size limits that match `MaxRequestBodySize`.
9. [ ] `Mail.json` + `SupportFrom`. Test by sending a notification.
10. [ ] `BackgroundService.json`: enable `Reminder` and the cleanup jobs you need.
11. [ ] Multiple instances: Redis session (`Session`/`Kvs`), shared data protection keys, `Quartz.Clustering`.
12. [ ] Optional features: API, server scripts, MCP, AI Connect, SCIM, full-text search.
13. [ ] `General.json` locale-related defaults (`ImportEncoding`, `FirstMonth`, `CharToAddWhenCopying`).
14. [ ] After startup, check the `Parameters` DB table for patches that override the files.
