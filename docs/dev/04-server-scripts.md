# 4. Server scripts (site, Extended, background)

Server scripts are JavaScript that runs on the server in ClearScript V8. The code is in `Libraries/ServerScripts/`. They come from three places:

| Source | Configured in | Scope |
|---|---|---|
| Site server scripts | Site settings → Server scripts (`SiteSettings.ServerScripts`) | one site |
| ExtendedServerScripts | `App_Data/Parameters/ExtendedServerScripts/*.json` (+ `*.json.js`), or Extensions rows of type `ServerScript` | sites matched by conditions ([02](02-extended-infrastructure.md#23-conditions-who--where-an-extension-applies)) |
| Background server scripts | Tenant settings (`TenantSettings.BackgroundServerScripts`) | tenant, on a schedule |

**For editor IntelliSense**, use `Implem.PleasanterFrontend/wwwroot/src/types/pleasanter-server-script.d.ts`. It declares every host object, with Japanese docs and examples:

```js
/// <reference path="../types/pleasanter-server-script.d.ts" />
```

## 4.1 Script settings

The settings class is `Libraries/Settings/ServerScript.cs`. `ExtendedServerScript` has the same flags, but no `TimeOut`.

| Field | Best practice |
|---|---|
| `Title`, `Name` | `Name` is the key for `//Include:<Name>`. Give every reusable script a stable `Name` |
| Conditions | Enable **only** the conditions you need. Each enabled condition is a V8 run |
| `Shared` | Library code that is prepended to every run on the site. Keep it small: it runs in every engine instance |
| `Functionalize` | **Always on.** The body is wrapped as `(()=>{ … })();`, so `return` works and variables don't leak |
| `TryCatch` | On for non-critical logic. The body is wrapped in `try{…}catch(e){ logs.LogException(...) }`, so an error is logged instead of failing the request. **Turn it off for validation scripts**: a swallowed error would let invalid data through |
| `TimeOut` | Only saved when `Script.ServerScriptTimeOutChangeable` is true. Default `Script.ServerScriptTimeOut` = 10 s |
| `Disabled` | Turn a script off without deleting it |

Other options:
- A body starting with `//debug//` attaches the V8 remote debugger and **pauses the request**. Use it only on a local machine and never commit it.
- Site switches: `ServerScriptsAllDisabled` (kill switch) and `ServerScriptsGetErrorDetails` (sends error details to the browser console). Enable the second only while developing.

## 4.2 How a run executes (and why `Functionalize` matters)

`ServerScriptUtilities.Execute` does the following:

1. It merges the extended scripts that match the conditions (first) with the site scripts, then expands `//Include:` (depth limit `ServerScriptIncludeDepthLimit`, default 10).
2. For a condition, it picks the scripts with that flag. **If none match, nothing runs, not even the `Shared` scripts.**
3. It creates **a new V8 engine for every run**, adds the host objects, and runs `shared bodies + matched bodies` as **one** script in one global scope.
4. It writes the changed values back to the model, columns, view and responses.

Consequences:
- Two scripts with a top-level `let x` give a SyntaxError for the whole run. **Turn on `Functionalize`.** The `Shared` bodies are *not* wrapped, so declare them as namespaced objects:

  ```js
  // Shared script, Name: "lib"
  var lib = lib || {};
  lib.isBlank = (v) => v === null || v === undefined || v === '';
  ```
- **The timeout is one deadline for the whole combined run.** It is checked only while JS runs, so a blocking `httpClient` call (default timeout 100 s) or a slow SQL is **not** interrupted. Set `httpClient.TimeOut` explicitly.
- **Recursion guard:** `items.*` calls, `UpdateOnExit` and so on run with a nested context at `ServerScriptDepth + 1`. At depth 10, scripts are **skipped silently**: no error is raised.

## 4.3 Conditions: when they fire and what they are for

The hooks are in `Models/Shared/_BaseModel.cs` `SetBy*ServerScript`, `View.SetByWhenViewProcessingServerScript` and `IssueModel.Create/Update/Delete`. Results/Wikis use the same flow.

| Condition | Fires | Use it for | Cost |
|---|---|---|---|
| `WhenloadingSiteSettings` | every Items request on the site, **and** every `items.*` call made against the site | `columns.*` labels and choices, `siteSettings.DefaultViewId`/`Sections` | high frequency, so keep it tiny |
| `WhenViewProcessing` | once per view build | `view.Filters`, `Sorters`, `AlwaysGetColumns`, `OnSelectingWhere`, `ColumnPlaceholders` | low |
| `WhenloadingRecord` | **every row loaded from the DB**: grid rows, export rows, `items.Get` results | rarely needed. Prefer `BeforeOpeningRow` / `BeforeOpeningPage` | **per row** |
| `BeforeOpeningPage` | once per page or editor | UI: `columns.X.ReadOnly/Hide/ExtendedFieldCss`, `elements.*`, `hidden.Add` | low |
| `BeforeOpeningRow` | **per grid row** (also export and link tables) | `model.ExtendedRowCss`, `ExtendedRowData`, `columns.X.ExtendedCellCss` | **per row** |
| `BeforeFormula` / `AfterFormula` | while form or API data is bound (`SetByForm`/`SetByApi`/CSV), before validation | derived values that must be saved | per save |
| `BeforeCreate` / `BeforeUpdate` / `BeforeDelete` | before the SQL. **Can cancel** with `context.Error()` | validation, defaulting | per save |
| `AfterCreate` / `AfterUpdate` / `AfterDelete` | after commit, notifications and `OnCreated`/`OnUpdated` ExtendedSqls | side effects on *other* records or systems | per save |
| `BeforeBulkDelete` / `AfterBulkDelete` | once, on an **empty** model. **Cannot cancel** | logging and cleanup only | per operation |

Create runs in this order: `BeforeCreate` → (cancel check) → `OnCreating` SQL → INSERT → summaries / auto-numbering → notifications → reload → `OnCreated` SQL → links → `AfterCreate`.

## 4.4 Changing values and cancelling

- **Only assigned properties are written back,** and only for columns that the current user **can edit** (`column.CanEdit`). An assignment to a column the user cannot edit is **silently dropped**. Don't use server scripts to bypass column permissions.
- **Changes in `Before*` / `*Formula` are saved. Changes in `After*` are not**, unless you set `model.UpdateOnExit = true`, which runs a second Update with Before/AfterUpdate again. **Guard it**, or it loops until depth 10:

  ```js
  // AfterUpdate, Functionalize on
  // The change check is the loop guard: in the nested update, saved.ClassZ already equals model.ClassZ.
  if (model.ClassZ !== saved.ClassZ) {
      model.NumA = (model.NumB || 0) * 1.1;
      model.UpdateOnExit = true;
  }
  ```
  If you have no such condition, use `context.ServerScript.ScriptDepth` (0 for a user request) as the guard. Better still, do the calculation in `BeforeUpdate`, which avoids the second save altogether.
- **To cancel, call `context.Error('message')` in `BeforeCreate`, `BeforeUpdate` or `BeforeDelete`** (also in grid edit). Then return. It has no effect in other conditions.

  ```js
  // BeforeUpdate, Functionalize on, TryCatch OFF
  if (model.Status >= 900 && !model.ClassA) {
      context.Error('Set "Customer" before closing the record.');
      return;
  }
  ```
- `context.Redirect(url)` works only on full-page requests. Use `context.AddMessage(text, css)` / `context.AddResponse(...)` for Ajax.

## 4.5 Host objects: key usage rules

| Object | Rule |
|---|---|
| `model`, `saved` | `saved` is read-only (the values before the edit). Compare `model.X !== saved.X` to detect changes |
| `items` | `Get`, `Create`, `Update`, `Upsert`, `Delete`, `BulkDelete`, `Sum/Count/Max/...`, `GetSiteByName/Title`, `GetClosestSite`. Runs **as the current user**, so permissions are enforced. **Returns `false`/`null`/`[]` on failure without throwing: check it.** `Get` on a site returns **at most `Api.PageSize` (200) rows**, so page or filter. Each call counts against the tenant API limit |
| `columns.<Name>` | UI properties: `ReadOnly`, `Hide`, `ValidateRequired`, `LabelText`, `Extended*Css`, `ExtendedHtml*`, `AddChoiceHash`/`ClearChoiceHash` |
| `view` | Only in `WhenViewProcessing` |
| `extendedSql` | Only ExtendedSqls with `Api: true`. Pass parameters as a JSON string: `extendedSql.ExecuteTable('Name', JSON.stringify({ CustomerId: 1 }))`. Not part of the record's transaction |
| `httpClient` | Synchronous. Set `TimeOut`, check `IsSuccess` / `StatusCode` / `IsTimeOut`. Disabled with `Script.DisableServerScriptHttpClient` |
| `logs` | `LogInfo/Warning/UserError/SystemError/Exception(message, method, console = true, sysLogs = true)`. **With `sysLogs` on, each call inserts a SysLogs row**, so pass `sysLogs = false` for debug output, or use `context.Log()` (browser console only) |
| `notifications` | `New()` / `Get(id)` then `Send()`. Prefer site Notification settings when they are enough |
| `$ps.file` | Only when `Script.DisableServerScriptFile` is false (off by default). Size and path are limited by `ServerScriptFileSizeMax` / `ServerScriptFilePath` |
| .NET arrays | Convert with `Array.from(x)` before `map`/`filter` |

## 4.6 Performance

1. **Never call `items.Get` in a loop over rows.** Each call loads the target site's settings (running *its* `WhenloadingSiteSettings`), queries the DB, counts the API limit and runs `WhenloadingRecord` per row. Fetch once with a view filter (`items.Get(siteId, JSON.stringify({ View: { ColumnFilterHash: { … } } }))`), or use `items.Sum/Count`.
2. **Avoid `WhenloadingRecord` and `BeforeOpeningRow` unless you need per-row output.** A 200-row grid page can create 400+ V8 engines.
3. **Keep `Shared` scripts small.** They are compiled into every run.
4. **Set short `httpClient.TimeOut` values,** and don't call external APIs from `BeforeOpening*` (every page view).
5. **Avoid writing logs on hot paths.**

## 4.7 ExtendedServerScripts

```
ExtendedServerScripts/
  Sales/
    CheckAmount.json      <- conditions and flags
    CheckAmount.json.js   <- the body (overrides "Body")
```

```json
{
    "Name": "CheckAmount",
    "Description": "Validate amount on Sales sites. Owner: sales team",
    "SiteIdList": [101, 102],
    "BeforeCreate": true,
    "BeforeUpdate": true,
    "Functionalize": true,
    "TryCatch": false
}
```

- **Always set `SiteIdList` (or other conditions).** Otherwise the script runs on every site.
- **Use `Shared: true` + `Name` for libraries,** and `//Include:<Name>` to pull a library into one specific script.
- **There is no `TimeOut` field.** Extended scripts use the site or the global timeout.
- Extended scripts run **before** the site's own scripts in the same combined body.
- Changes need a restart or `/Admins/ReloadParameters`.

## 4.8 Background server scripts

- They need both `Script.ServerScript` and `Script.BackgroundServerScript: true` (off by default). On a multi-node setup, `BackgroundService.EnvironmentVariables` decides which node runs them.
- Schedules: hourly, daily, weekly, monthly or once, as a Quartz cron in `ScheduleTimeZoneId` (falls back to `Service.TimeZoneDefault`, then UTC).
- **They run as the configured `UserId`.** That user must be enabled and needs permission on every site the script touches. Use a dedicated service user.
- **`//Include:` is not expanded here.** Tenant `Shared` background scripts are prepended instead.
- There is no HTTP context, so `context.Forms`, `responses` and `elements` mean nothing.
- **Make jobs idempotent and resumable.** A job can be re-run or overlap with a slow previous run. Process in pages (200 rows per `items.Get`) and record progress, e.g. a status column.

## 4.9 Errors and debugging

- **When a script error is not caught** (`TryCatch` off), the request fails:
  - Ajax gets HTTP 422 with `ServerScriptExecutionFailed`, plus details in the console when `ServerScriptsGetErrorDetails` is on.
  - Full pages redirect to the server-script error page.
  - The exception is written to **SysLogs**.
- **For debug output,** use `context.Log(JSON.stringify(x))`, which shows in the browser console of the Ajax response, or `logs.LogInfo(msg, 'where', true, false)`.
- **Test against the entry points that are easy to forget:** grid edit, bulk update, REST API, CSV import (formulas run, record ExtendedSqls do not), and copy.

## 4.10 Checklist

- [ ] `Functionalize` on, and `TryCatch` chosen on purpose (off for validation)
- [ ] Only the needed conditions are enabled, with nothing heavy in per-row or per-request conditions
- [ ] `items.*` results checked, no `items.Get` in loops, and the 200-row cap handled
- [ ] `UpdateOnExit` / cross-site updates guarded against recursion (`context.ServerScript.ScriptDepth`)
- [ ] Validation is in `Before*` with `context.Error`, and also covers API, grid and import
- [ ] `httpClient.TimeOut` set, and secrets not hard-coded where site admins can read them
- [ ] No `//debug//`, no noisy `logs.*` on hot paths
- [ ] No hard-coded IDs, so it survives a site-package move ([02](02-extended-infrastructure.md#25-promoting-from-dev-to-prod))
