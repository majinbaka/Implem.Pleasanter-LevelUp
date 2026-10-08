# Developer guide: building features on Pleasanter

These notes are for developers who build business features on Pleasanter, either by customizing a site (scripts, styles, HTML, server scripts) or by adding `App_Data/Parameters/Extended*` files and core code. Every rule here comes from reading the source in this repo. File paths are relative to `Implem.Pleasanter/` unless they start with another project name. Line numbers are approximate.

## Files

| File | Read it when you… |
|---|---|
| [01-choosing-a-mechanism.md](01-choosing-a-mechanism.md) | are deciding *where* a feature should live (site setting, Extended* file, Extensions table, or core code) |
| [02-extended-infrastructure.md](02-extended-infrastructure.md) | add any `Extended*` file or Extensions row: loading, matching rules (`SiteIdList`, negative IDs…), reload, deploy |
| [03-client-scripts-styles-htmls.md](03-client-scripts-styles-htmls.md) | write JS / CSS / HTML in **Site settings** (Scripts, Styles, Htmls) or in `ExtendedScripts/Styles/Htmls` |
| [04-server-scripts.md](04-server-scripts.md) | write server scripts (site, `ExtendedServerScripts`, background) |
| [05-extended-sqls.md](05-extended-sqls.md) | write `ExtendedSqls` (CRUD hooks, view filters, API, server-script SQL) |
| [06-other-extensions.md](06-other-extensions.md) | use ExtendedFields, NavigationMenus, StartGuides, Plugins (PDF) or ExtendedColumnDefinitions |
| [07-core-code-changes.md](07-core-code-changes.md) | change the C# / TypeScript of Pleasanter itself |
| [08-review-checklist.md](08-review-checklist.md) | review or release a customization: security, performance and deploy checklist |

## The ten rules that matter most

1. **Use the lightest mechanism that works.** Prefer site settings first, then Extended* files, and change core code last ([01](01-choosing-a-mechanism.md)).
2. **Wrap every client script in an IIFE** and register events with `$(document).on` / `$p.on`. Never bind directly to elements that Ajax replaces ([03](03-client-scripts-styles-htmls.md)).
3. **Chain `$p.events.*` handlers.** Each event holds one function, so assigning it again overwrites someone else's handler ([03](03-client-scripts-styles-htmls.md)).
4. **Turn on `Functionalize` (and usually `TryCatch`) for every server script.** All matched scripts run in one V8 global scope ([04](04-server-scripts.md)).
5. **Keep per-row conditions light.** `WhenloadingRecord` and `BeforeOpeningRow` run once per row, and each run creates a new V8 engine ([04](04-server-scripts.md)).
6. **Check the return values of `items.*`.** Failures return `false`/`null`/`[]` instead of throwing, and recursion past depth 10 is skipped silently ([04](04-server-scripts.md)).
7. **ExtendedSql `{{…}}` placeholders are string concatenation.** Pass user input only as bound `@params` (API / server script / ExtendedField `SqlParam`) ([05](05-extended-sqls.md)).
8. **An `Api: true` ExtendedSql has no site-permission check.** Restrict it with `UserIdList`/`GroupIdList`/`DeptIdList` and write the SQL so it cannot leak data ([05](05-extended-sqls.md)).
9. **Bad JSON in `App_Data/Parameters/Extended*` stops startup.** Changes take effect only after a restart or `/Admins/ReloadParameters` ([02](02-extended-infrastructure.md)).
10. **Site packages carry site settings but not Extended* files, and IDs inside script and SQL bodies are not rewritten.** Plan how you promote changes from dev to prod ([02](02-extended-infrastructure.md), [08](08-review-checklist.md)).

## Related docs

- [../startups/app-data-config-checklist.md](../startups/app-data-config-checklist.md): environment parameters (`Rds.json`, `Security.json`, …).
- [../../source-map/README.md](../../source-map/README.md): where each feature lives in the code.
- Type definitions for editor IntelliSense:
  - client `$p`: `Implem.PleasanterFrontend/wwwroot/src/types/pleasanter-script.d.ts`
  - server scripts: `Implem.PleasanterFrontend/wwwroot/src/types/pleasanter-server-script.d.ts`
- Official manual: https://pleasanter.org/manual
