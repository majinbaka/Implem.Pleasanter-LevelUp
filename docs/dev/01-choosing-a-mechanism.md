# 1. Choosing where a feature lives

Pleasanter offers four layers for customization. Pick the **lowest layer that can do the job**: it is easier to deploy and to review, and it survives upgrades.

| Layer | Where it is stored | Scope | Who can change it | Deployed by | Survives upgrade |
|---|---|---|---|---|---|
| **A. Site settings**: Scripts, Styles, Htmls, ServerScripts, Processes, StatusControls, Formulas, Views… | `Sites.SiteSettings` JSON (DB) | one site | site admins, through the UI | site package export/import, or by hand | yes |
| **B. Extended* files** | `App_Data/Parameters/Extended*/` | whole instance, narrowed by conditions | server operators | file copy + restart/reload | yes, but keep them out of the upgrade overwrite |
| **C. Extensions table** | `Extensions` rows (DB) | whole instance (no tenant filter) | privileged users through `/api/extensions` | API call + reload | yes |
| **D. Core code** | C# / TS in this repo | everything | developers | build + deploy | **merge conflicts on every upstream update** |

## Decision guide

| You need to… | Use |
|---|---|
| Hide, disable or relabel fields, or add a button on one site | A: site **Script** / **Style**, or **ServerScript** `BeforeOpeningPage` with `columns.*`/`elements.*` |
| Make a field required, read-only or hidden by status | A: **StatusControls** first. Use a server script (`columns.X.ReadOnly`) only if the rule is more complex |
| Validate a value before saving and block the save | A: **ServerScript** `BeforeCreate`/`BeforeUpdate` + `context.Error(...)`. Client-side `before_validate` is only for UX: it can be bypassed |
| Compute a value from other fields | A: **Formula** (simple), then ServerScript `BeforeFormula`/`AfterFormula` |
| Create or update records on another site when a record is saved | A: ServerScript `AfterCreate`/`AfterUpdate` with `items.*` |
| Run a job on a schedule | A (tenant): **BackgroundServerScript** (needs `Script.BackgroundServerScript: true`) |
| Write to the DB in the **same transaction** as the record save | B: **ExtendedSql** `OnCreating/OnUpdated/...` |
| A filter that the normal view filter cannot express (sub-query, EXISTS, joins) | B: **ExtendedSql** `OnSelectingWhere` (+ ExtendedField for user input) |
| Run a read-only report query from JS or a server script | B: **ExtendedSql** `Api: true` (see the security notes in [05](05-extended-sqls.md)) |
| The same JS/CSS on many sites or on every page (company header, analytics) | B: **ExtendedScripts / ExtendedStyles / ExtendedHtmls** |
| The same server-script logic on many sites | B: **ExtendedServerScripts** (`SiteIdList`, `Shared`, `//Include:`) |
| Change the top navigation menu or the start guide | B: **ExtendedNavigationMenus**, **ExtendedStartGuides** |
| Custom PDF output | B: **ExtendedPlugins** (`IPdfPlugin` DLL) |
| More Class/Num/Date… columns than A–Z | B: `ExtendedColumns` (commercial or trial license) + `CodeDefiner _rds` |
| Manage B-type extensions without touching the server's files | C: **Extensions table** through `/api/extensions` |
| A new screen, a new controller, a change to core behavior | D: core code ([07](07-core-code-changes.md)) |

## Rules of thumb

- **Server-side rules must be enforced on the server.** Anything done only in client JS can be bypassed by the REST API, CSV import, grid editing or a crafted request. Use the client for UX and the server (server script, ExtendedSql, StatusControl, column permissions) for rules.
- **Know the entry points that skip your logic:**
  - CSV import runs per-row Create/Update with `extendedSqls: false`. Record-level ExtendedSql hooks do not fire; only `OnImporting`/`OnImported` run, once for the whole import.
  - `BeforeBulkDelete` cannot cancel the delete.
  - Server scripts are disabled during site-package import and reminder tests.
- **One site, one owner.** When several people add scripts to a site, use the `Title` of each item to say who owns it and why, and keep each item small.
- **Write extensions in the configured DBMS dialect.** ExtendedSql has no per-DBMS variant.
- **Avoid core changes** unless the feature cannot be built otherwise. When you must change core code, keep the change small and isolated, and follow [07](07-core-code-changes.md).
