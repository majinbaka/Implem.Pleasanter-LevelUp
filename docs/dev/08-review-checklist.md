# 8. Review and release checklist

Use this list in PR reviews and before a release. Each item links to the page that explains it.

## Design

- [ ] The feature uses the lightest mechanism that works: site settings, then Extended*, then core ([01](01-choosing-a-mechanism.md))
- [ ] Every business rule is enforced on the **server** (server script `Before*`, ExtendedSql hook, StatusControl, permissions), not only in client JS
- [ ] You know which entry points skip the logic and accept it:
  - CSV import skips record ExtendedSql hooks
  - `BeforeBulkDelete` cannot cancel
  - site-package import disables server scripts

## Security

- [ ] No user input concatenated into SQL. Only bound `@params` ([05 §5.2](05-extended-sqls.md#52-placeholders-and-parameters-read-before-writing-any-sql))
- [ ] Each `Api: true` ExtendedSql is restricted by user, group or dept lists, and enforces data access inside the SQL ([05 §5.5](05-extended-sqls.md#55-api-sql-api-true-apiextendedsql-and-extendedsql))
- [ ] `DbUser: "Owner"` is used only where it is really needed
- [ ] Client code builds DOM from record values with `.text()` / `textContent`, never `.html()` ([03 §3.7](03-client-scripts-styles-htmls.md#37-security))
- [ ] No `<script>` inside Htmls / ExtendedHtmls / CustomHtml, so it stays CSP-safe
- [ ] No secrets in client JS. Server-script secrets are kept to a minimum, and access to site settings is limited accordingly
- [ ] Server scripts don't rely on bypassing column permissions (assignments to columns the user cannot edit are dropped)
- [ ] `Html: true` ExtendedSqls expose only non-sensitive data
- [ ] Extensions rows are restricted by conditions, because the table has no tenant filter

## Correctness

- [ ] Client scripts are wrapped in an IIFE, `$p.events` handlers are chained and idempotent, and event binding is delegated ([03](03-client-scripts-styles-htmls.md))
- [ ] Values are changed with `$p.set`, not `.val()`
- [ ] Server scripts have `Functionalize` on, and `TryCatch` set deliberately ([04 §4.1](04-server-scripts.md#41-script-settings))
- [ ] `items.*` return values are checked, and the 200-row cap of `items.Get` is handled
- [ ] Recursion is guarded (`UpdateOnExit`, cross-site updates)
- [ ] Extended* items are restricted (`SiteIdList` / `SpecifyByName`), and `Controllers` / `Actions` are lowercase
- [ ] Positive and negative IDs are not mixed in condition lists ([02 §2.3](02-extended-infrastructure.md#23-conditions-who--where-an-extension-applies))
- [ ] SQL is written for the configured DBMS, and `{{Timestamp}}` is quoted

## Performance

- [ ] Nothing heavy runs in `WhenloadingSiteSettings`, `WhenloadingRecord`, `BeforeOpeningRow` or `BeforeOpening*` ([04 §4.6](04-server-scripts.md#46-performance))
- [ ] No `items.Get` / `httpClient` / `extendedSql` calls inside per-row loops
- [ ] `httpClient.TimeOut` is set
- [ ] View-filter SQL fragments and CRUD hooks are index-friendly and tested on a realistic data volume
- [ ] `logs.*` calls on hot paths use `sysLogs = false`, or are removed

## Testing

- [ ] Tested on the full editor, the dialog editor (opened from the grid), grid edit, bulk operations, REST API and CSV import, as applicable
- [ ] Tested as a normal user, not only as an admin (permissions change behavior)
- [ ] Tested with every language and theme you support, and in mobile/responsive mode when it is relevant
- [ ] Client changes previewed with `DraftOutputMode` / `?Draft=1` before going live
- [ ] Critical flows covered by an ExtendedAutoTest scenario where practical

## Release and deploy

- [ ] Extended* files and site-package JSON are committed to git and reviewed
- [ ] The JSON in `App_Data/Parameters/Extended*` validates (`jq .`), and no Extended* directory was deleted ([02 §2.1](02-extended-infrastructure.md#21-how-files-are-loaded))
- [ ] Environment-specific IDs (site, record, user, group) are listed, and each environment's values are prepared ([02 §2.5](02-extended-infrastructure.md#25-promoting-from-dev-to-prod))
- [ ] Extensions rows are created or updated, then `/Admins/ReloadParameters` is called (or the app restarted) **on every node**
- [ ] Schema changes (`ExtendedColumns`, definitions) are run with `CodeDefiner _rds` after a DB backup
- [ ] Debug leftovers are removed: `//debug//`, `ServerScriptsGetErrorDetails`, `console.log`, test `Draft` items
- [ ] A rollback plan exists: disable the item (`Disabled: true`, site `*AllDisabled` switches) rather than deleting it
