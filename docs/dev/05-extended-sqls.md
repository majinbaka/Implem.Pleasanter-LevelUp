# 5. ExtendedSqls

An ExtendedSql is raw SQL that Pleasanter runs at well-defined points: record CRUD, view filtering and sorting, the REST API, server scripts, page rendering and 2FA. You define it in `App_Data/Parameters/ExtendedSqls/*.json` (+ `*.json.sql`) or in an Extensions row of type `Sql`. Loading and conditions are covered in [02](02-extended-infrastructure.md).

## 5.1 Definition

The class is `Implem.ParameterAccessor/Parts/ExtendedSql.cs`, which inherits `ExtendedBase`:

| Property | Meaning |
|---|---|
| `CommandText` | The SQL. Prefer a companion file `<name>.json.sql` |
| `OnCreating` / `OnCreated`, `OnUpdating` / `OnUpdated`, `OnDeleting` / `OnDeleted` | Record CRUD hooks (**same transaction** as the save) |
| `OnUpdatingByGrid` / `OnUpdatedByGrid`, `OnBulkUpdating` / `OnBulkUpdated`, `OnBulkDeleting` / `OnBulkDeleted` | Grid-edit and bulk variants (`{{Id}}` = 0) |
| `OnImporting` / `OnImported` | Once before and once after a whole CSV import |
| `OnSelectingWhere` (+ `OnSelectingWhereParams`) | A raw fragment ANDed into the view's WHERE |
| `OnSelectingOrderBy` (+ `OnSelectingOrderByParams`) | A raw fragment added to ORDER BY |
| `OnSelectingColumn` (+ `OnSelectingColumnParams`, `ColumnList`) | Replaces a grid column with `(CommandText) as <Column>` |
| `OnSelectingWherePermissionsDepts/Groups/Users` | Narrows the candidate lists in the permission picker |
| `Api` | Callable from `/api/extended/sql` and from the server-script `extendedSql` object |
| `DbUser` | `"Owner"` runs the API / server-script path with the Owner connection. Any other value uses the User connection |
| `Html` | Runs on every full page render and writes the result as JSON into `<input type=hidden id="<Name>">` |
| `OnUseSecondaryAuthentication` | Gets `@TenantId`, `@UserId`. Returning false skips 2FA |

**There is no per-DBMS variant.** Write the SQL for the DBMS in `Rds.json` `Dbms`. If you ship to several DBMSs, keep one file set per DBMS in separate folders and deploy only the matching one.

## 5.2 Placeholders and parameters (read before writing any SQL)

| Kind | Syntax | Source | Safe for user input? |
|---|---|---|---|
| Text substitution | `{{SiteId}}`, `{{Id}}`, `{{Timestamp}}` | `ExtendedSql.ReplacedCommandText` (string `Replace`) | IDs and the timestamp come from the server. **Quote `{{Timestamp}}`** (`'{{Timestamp}}'`) |
| Column placeholder | `{{key}}` | `view.AddColumnPlaceholder(key, column)` in `WhenViewProcessing` → `"Table"."Column"` | Only for column names you control |
| Built-in bound params | `@_T`, `@_D`, `@_U` (SQL Server), `@ipT`, `@ipD`, `@ipU` (PostgreSQL / MySQL) | tenant / dept / user of the current request (`Parameter.json` `SqlParameterPrefix`) | yes |
| API / server-script params | `@Key` | `Params` in the API body, or the JSON passed to `extendedSql.*` | **yes, bound parameters** |
| ExtendedField values | `@FieldName` | ExtendedField with `SqlParam: true` | yes |
| Batch index | `#CommandCount#` | statement index in the batch | n/a |

Rules:
- **Never put user-controlled text into the SQL by string building.** The only way to pass user input is through bound `@params`. Don't build SQL text in a server script and pass it as a param value.
- **A null API param is not bound.** If the SQL references `@X` and the caller sends `null` or omits it, the SQL errors. Send an explicit value, or default it in SQL (SQL Server: declare it).
- **Use the built-in `@_U` / `@ipU` for "current user" logic** instead of passing the user ID from the client.

## 5.3 CRUD hooks

- **They run in the same transaction** as the record's INSERT/UPDATE/DELETE. If your SQL fails, **the whole save is rolled back** and the user sees an error. That is useful for integrity, but it means a bug in a hook blocks every save on the site.
- **`OnCreating` runs before the INSERT** with `{{Id}}` = 0. Use `OnCreated` when you need the new ID.
- **When any `OnCreated` / `OnUpdated` hook applies, the record is re-read after the save,** so changes your SQL makes to the same record show up in the response.
- **Grid / bulk / import variants get `{{SiteId}}` only** (`{{Id}}` = 0). Write them set-based (e.g. "recalculate all rows of site X"), not per record.
- **CSV import does not fire the record hooks** (per-row saves use `extendedSqls: false`). Put import-time logic in `OnImporting` / `OnImported`.
- **Always restrict the hook:** `SiteIdList` (or a `SpecifyByName`), plus `Controllers` / `Actions` if needed. A hook with empty lists fires on every save in the instance.
- **Keep hooks fast.** They hold the record transaction (and its locks) open.

Example: keep a denormalized total on a parent record after a child is saved.

`ExtendedSqls/Orders/UpdateOrderTotal.json`:
```json
{
    "Name": "UpdateOrderTotal",
    "Description": "Recalculate order total when an order line changes. Owner: sales",
    "SiteIdList": [202],
    "OnCreated": true,
    "OnUpdated": true,
    "OnDeleted": true
}
```

`ExtendedSqls/Orders/UpdateOrderTotal.json.sql` (SQL Server):
```sql
update "Results"
set "NumA" = (
    select coalesce(sum(l."NumB"), 0)
    from "Results" l
    where l."SiteId" = {{SiteId}}
      and l."ClassA" = cast(p."ResultId" as nvarchar(32)))
from "Results" p
where p."SiteId" = 201
  and p."ResultId" = (select cast("ClassA" as bigint) from "Results" where "ResultId" = {{Id}});
```

The hard-coded `201` is the weak spot of this example: IDs differ per environment, so keep such constants in one place and document them.

## 5.4 View filters and sorts (`OnSelectingWhere` / `OnSelectingOrderBy`)

The fragment is added to the grid query's WHERE (or ORDER BY). It applies when one of these is true:
- a view has `OnSelectingWhere` set to the SQL's `Name` (set it in a server script `WhenViewProcessing`: `view.OnSelectingWhere = 'Name'`), or
- the filter key `OnSelectingWhere` holds the name, or
- **the entry is not `SpecifyByName` and passes its conditions. Then it applies to *every* view of the matched sites.** Use `SpecifyByName: true` unless you want that.

`OnSelectingWhereParams` makes the fragment apply only when all the listed filter keys are present. Pair it with an **ExtendedField** (`FieldType: "Filter"`, `SqlParam: true`) to get a custom filter box whose value arrives as `@FieldName`:

```json
// ExtendedFields/TaskKeyword.json
{ "Name": "TaskKeyword", "FieldType": "Filter", "TypeName": "nvarchar",
  "LabelText": "Open task contains", "SqlParam": true, "SiteIdList": [101] }
```
```json
// ExtendedSqls/TaskKeyword.json  (+ TaskKeyword.json.sql)
{ "Name": "TaskKeyword", "SiteIdList": [101], "OnSelectingWhere": true,
  "OnSelectingWhereParams": ["TaskKeyword"] }
```
```sql
-- TaskKeyword.json.sql (SQL Server): fragment only, no "where" keyword
exists (select 1 from "Issues" t
        where t."ClassA" = cast("Issues"."IssueId" as nvarchar(32))
          and t."Status" < 900
          and t."Title" like '%' + @TaskKeyword + '%')
```

- **Write a fragment, not a statement.** It is inserted inside a larger query. Refer to the main table by name (`"Issues"`, `"Results"`), and alias your sub-query tables so they don't collide.
- **Watch performance.** The fragment runs for every grid page, every count, export, calendar and so on. Make sure correlated sub-queries hit indexes.
- **Row-level permission filters are still applied** by Pleasanter. Your fragment only narrows the result further.

## 5.5 API SQL (`Api: true`): `/api/extended/sql` and `extendedSql`

Call from the client:
```js
// $p.apiExec adds the request token (Token) and the JSON content type for you
$p.apiExec($p.apiUrl(null, 'sql', 'extended'), {
    data: { Name: 'SalesByMonth', Params: { Year: 2026 } },
    done: (res) => console.log(res.Response.Data.Table)
});
// → { StatusCode: 200, Response: { Data: { Table: [ {...}, ... ] } } }
```

Call from a server script:
```js
const rows = extendedSql.ExecuteTable('SalesByMonth', JSON.stringify({ Year: 2026 }));
```

The server-script methods are `ExecuteDataSet`, `ExecuteTable`, `ExecuteRow`, `ExecuteScalar` and `ExecuteNonQuery`.

**Security: this is the riskiest extension point.**
- **The API endpoint checks only that the caller is authenticated** (session or API key) and that the item passes `ExtensionWhere`. **There is no site or record permission check**, and no `Api.Enabled` / user `AllowApi` check. Any logged-in user can call any `Api: true` SQL whose conditions they meet.
  - **Restrict callers** with `UserIdList`, `GroupIdList` or `DeptIdList`. `SiteIdList` does **not** help on the API route, because SiteId is 0 there and a positive list never matches.
  - **Enforce data access inside the SQL**, e.g. join on permissions or filter by `@_U` / `@ipU`.
  - **Prefer read-only SELECTs.** Expose writes only through server scripts that validate first. Use `DbUser: "Owner"` only for SQL that really needs DDL or owner rights.
- **The server-script path uses the real request context,** so `SiteIdList` / `Controllers` conditions work there.
- **It runs outside the record transaction** (`transactional: false`).
- **The result is keyed by DataTable:** `Table`, `Table1`, …
- An unknown or non-matching `Name` currently ends in a server error, not a clean 4xx, so handle failures on the client.

## 5.6 `Html: true`

The SQL runs on **every full page render** that matches its conditions, and the result is exposed to the browser in a hidden input. Use it only for small, non-sensitive lookup data, and restrict it with `SiteIdList` / `Actions`. Everything in it is readable by the user.

## 5.7 Multi-DBMS and style

- **Quote identifiers with double quotes** (`"Issues"."ClassA"`). That works on SQL Server and PostgreSQL. On MySQL it depends on `ANSI_QUOTES`, so test there.
- **Engine differences to watch:** `top` vs `limit`, string concatenation, `cast` types (`nvarchar` vs `varchar`/`text`), `getdate()` vs `now()`, and the parameter prefix (`@_U` vs `@ipU`).
- **Columns are typed by definition:** `ClassX` is a string, `NumX` decimal, `DateX` datetime, `CheckX` bit/bool. Link columns (`ClassX` holding a record ID) need a cast to join on a bigint ID.
- **Write to `Results` / `Issues` only through hooks or controlled SQL.** Raw UPDATEs bypass history (`_history` tables), notifications, formulas and full-text index updates. When the change must be audited or notified, use a server script with `items.Update` instead.

## 5.8 Errors and debugging

- **A failing SQL throws.** For CRUD hooks, the save rolls back. The exception goes to **SysLogs** (with details if `SysLog.OutputErrorDetails` is on).
- **The SQL text log** (`SqlDebugs.WriteSqlLog`) exists **only in DEBUG builds.** When running locally with `dotnet run` (Debug), check it to see the final SQL after placeholder substitution.
- **Develop the SQL in a DB client first,** with literal values in place of the placeholders. Then move it to the `.json.sql` file.
- **Only `SqlIo.Try` retries, and only on deadlocks** (`Rds.json` `DeadlockRetryCount`).

## 5.9 Checklist

- [ ] SQL in a `.json.sql` file in git, written for the configured DBMS
- [ ] Restricted by `SiteIdList` / `SpecifyByName` / user lists, with the reason in `Description`
- [ ] No user input concatenated into SQL. Only bound `@params`, with nulls handled
- [ ] `Api: true` SQL: callers restricted, data access enforced inside the SQL, read-only where possible
- [ ] CRUD hooks are fast and set-based where `{{Id}}` = 0, and you know which entry points (import, bulk) skip them
- [ ] View fragments are index-friendly and tested on a large site
- [ ] Hard-coded IDs documented, with a plan for each environment
