# 3. Client-side JS / CSS / HTML (site settings and Extended files)

This page covers **Site settings → Scripts / Styles / HTML** and the instance-wide `ExtendedScripts`, `ExtendedStyles` and `ExtendedHtmls` files.

## 3.1 Settings model

`Libraries/Settings/Script.cs`, `Style.cs` and `Html.cs` have the same fields:

| Field | Meaning |
|---|---|
| `Title` | Free text. Put the purpose, the owner and a ticket number here |
| `Body` | Raw JS / CSS / HTML, output **without escaping** |
| `All` | Output on every screen of the site |
| `New`, `Edit` | Output on the full-page editor (new / existing record) |
| `Index`, `Calendar`, `Crosstab`, `Gantt`, `BurnDown`, `TimeSeries`, `Analy`, `Kamban`, `ImageLib` | Output on that view |
| `Disabled` | Turn the item off without deleting it |
| `DraftOutputMode`, `DraftKey` | `1` = output only when the URL has `?Draft=<key>` (key defaults to `1`). `2` = output only when it does not |

`Html` also has `PositionType`: `HeadTop`, `HeadBottom`, `BodyScriptTop` or `BodyScriptBottom`.

Kill switches:
- per site: `ScriptsAllDisabled`, `StylesAllDisabled`, `HtmlsAllDisabled`
- per tenant: `ContractSettings.Script/Style/Html`

**Use `DraftOutputMode` to test changes in production safely:**
1. Add the new version with mode `1`.
2. Open the page with `?Draft=1` and check it. Other users still get the old version, which you mark with mode `2`.
3. Switch the modes when you are done.

## 3.2 Where and in what order things are output

`Libraries/HtmlParts/HtmlTemplates.cs`, `HtmlScripts.cs` and `HtmlStyles.cs` build the page in this order:

```
<head>
  ExtendedHtml HtmlHeaderTop → site Html HeadTop → meta → app CSS → ExtendedStyles (<link>)
  → <title> → ExtendedHeader → site Html HeadBottom → ExtendedHtml HtmlHeaderBottom
<body>
  ExtendedHtml HtmlBodyTop → page content
  → site Styles (inline <style>: All first, then view/editor)
  → site Html BodyScriptTop
  → libraries (jQuery, jQuery UI, …) → app bundles ($p) → ES modules
  → ExtendedScripts (<script src="resources/scripts?...">)
  → site Scripts "All" (one inline <script>)
  → site Scripts for the view/editor (a second inline <script>)
  → site Html BodyScriptBottom → ExtendedHtml HtmlBodyBottom
  → on window load: $p.execEvents('on_editor_load' | 'on_grid_load' | 'on_<view>_load')
```

What this means for your code:

- **Site scripts run synchronously while the page is parsed.** `$p` and jQuery already exist, but the `on_*_load` events have not fired yet. Code that needs the rendered grid or editor goes into those events.
- **All enabled script items of one category are concatenated into one `<script>`.** A syntax error in one item stops **all** of them. Lint your code before you paste it.
- **The "All" tag and the view/editor tag share the global scope.** A top-level `const x` in both throws `Identifier 'x' has already been declared`. **Always wrap code in an IIFE** and expose shared helpers under `$p.ex`:

  ```js
  (() => {
      'use strict';
      $p.ex.sales = $p.ex.sales || {};
      $p.ex.sales.formatAmount = (v) => Number(v).toLocaleString();
  })();
  ```
- **Ajax responses never re-send site scripts.** When the editor opens in a dialog from the grid, you are still on the *index* page, so only **All + Index** scripts are loaded. If an editor feature must work both in the dialog and on the full page, put it in **All**, guarded by `$p.action()`.
- **Site scripts are not output on the site's own settings screen.** A broken script cannot lock you out of fixing it.
- **Scripts, styles and HTML are not inherited by child sites.** Use ExtendedScripts with a `SiteIdList` to share code across sites.

## 3.3 Client API you should use (`$p`)

The full reference is `Implem.PleasanterFrontend/wwwroot/src/types/pleasanter-script.d.ts`. Point your editor at it, e.g. `/// <reference path="…/pleasanter-script.d.ts" />`.

| Need | API | Notes |
|---|---|---|
| Page info | `$p.siteId()`, `$p.id()`, `$p.action()`, `$p.controller()`, `$p.tableName()`, `$p.referenceType()`, `$p.userId()`, `$p.loginId()`, `$p.deptId()`, `$p.groupIds()`, `$p.selectedIds()` | `$p.siteId('Title')` resolves a *joined* site by title |
| Column name from label | `$p.getColumnName('Customer')` | Accepts the label **or** the column name. Use labels for readability and column names for robustness |
| Control / field element | `$p.getControl('ClassA')` → `#Issues_ClassA`, `$p.getField('ClassA')` → `#Issues_ClassAField` | IDs are `<ReferenceType>_<Column>`. Grid inline edit adds `_<SiteId>_<Id>` |
| Read value | `$p.getValue('ClassA')` | Handles `data-raw` and checkboxes |
| **Set value** | `$p.set($p.getControl('ClassA'), value)` | A plain `.val()` changes the UI but is **not saved**. `$p.set` also updates `$p.data` |
| Grid | `$p.getGridRow(id)`, `$p.getGridCell(id, 'ClassA')` | |
| Delegated events | `$p.on('change', 'ClassA', fn)` | Bound on `document`, so it survives Ajax replacement |
| REST API | `$p.apiGet/apiCreate/apiUpdate/apiUpsert/apiDelete({ id, data, done, fail })` | Adds the token for you. Avoid `async: false`, which calls `$.ajaxSetup` and changes **every** later request |
| Messages | `$p.setMessage('#Message', JSON.stringify({ Css: 'alert-error', Text: '…' }))`, `$p.clearMessage()` | |

## 3.4 Events (`$p.events`)

`$p.execEvents(name, args)` calls these handlers in order:
1. `$p.events[name]`
2. `$p.events[name + '_' + controlId]`
3. `$p.events[name + '_' + data-action]`

**If a handler returns `false`, the action is cancelled.**

| Group | Events |
|---|---|
| Screen loaded (also fired again after Ajax re-render) | `on_editor_load`, `on_grid_load`, `on_calendar_load`, `on_crosstab_load`, `on_gantt_load`, `on_burndown_load`, `on_timeseries_load`, `on_analy_load`, `on_kamban_load` |
| Form / send | `before_setData`, `after_setData`, `before_validate`, `after_validate`, `before_send`, `after_send` |
| Ajax | `ajax_before_send`, `ajax_before_done`, `ajax_after_done`, `ajax_before_fail`, `ajax_after_fail`, `ajax_before_always`, `ajax_after_always`, `ajax_after_send` |
| Applying a server response | `before_set`, `after_set` |

**Each event name holds a single function.** If two scripts assign `$p.events.on_editor_load`, the second one wins. Always chain:

```js
(() => {
    const prev = $p.events.on_editor_load;
    $p.events.on_editor_load = function (args) {
        if (prev && prev.apply(this, arguments) === false) return false;
        // your code. Must be idempotent: it runs again after every Ajax re-render
        $p.getField('ClassB').toggle($p.getValue('ClassA') === '1');
    };
})();
```

Other event rules:
- **Make load handlers idempotent.** `on_editor_load` fires after switching records, after Ajax updates and when a dialog opens. Guard with a flag or a check, and don't append the same button twice.
- **Don't bind directly to elements under `#MainContainer`, `#Grid` or `#EditInDialogBody`.** They are replaced by `Html`/`ReplaceAll` responses. Use `$(document).on(...)` / `$p.on(...)`.
- **Field auto-postback refreshes** (`EditorFields` responses) do **not** fire `on_editor_load`. Use `after_set` if you need to react to them.
- **Use `before_send` / `before_validate` for UX checks only.** Repeat every rule on the server ([04](04-server-scripts.md)).

## 3.5 CSS

- **Scope selectors** to the page or the field, e.g. `#Issues_ClassAField`, `.grid-row[data-id]`, or a class set by a server script (`columns.X.ExtendedFieldCss`). A global `input { … }` affects dialogs and the menus too.
- Site Styles are output at the end of `<body>`, but they still apply to the whole page. To override app CSS, use specificity instead of `!important` where you can.
- **Themes and responsive mode change the markup.** Test with the themes you support and with `$p.responsive()` (mobile).
- **Don't change the layout from JS when CSS can do it.** CSS survives Ajax re-renders without any code.

## 3.6 ExtendedScripts / ExtendedStyles / ExtendedHtmls

These are files under `App_Data/Parameters/Extended{Scripts,Styles,Htmls}/`, or Extensions-table rows of type `Script` / `Style` / `Html`.

- **Scripts and styles are served as separate files** (`resources/scripts?v=<hash>&site-id=…`, `resources/styles?...`). They are cacheable and get the CSP nonce. The conditions are re-evaluated on the server for the current user.
- **File-based items have no conditions.** They are output on every page of every site, including login and the user pages. Write them defensively:

  ```js
  (() => {
      if ($p.siteId() !== 0 && $p.controller() === 'items') { /* … */ }
  })();
  ```

  For per-site, per-user or per-action targeting, use an **Extensions row** (with `SiteIdList`, `Actions`…) instead of a file.
- **Load order is by file name**, then sub-folders. Use prefixes (`00_polyfill.js`, `10_common.js`).

### 3.6.1 ExtendedHtmls positions

The file name (without `.html`) is the position id:

| Id | Where |
|---|---|
| `HtmlHeaderTop`, `HtmlHeaderBottom`, `HtmlBodyTop`, `HtmlBodyBottom` | every page |
| `ColumnTop`, `ColumnBottom` | around each editor field (use `ColumnList` in an Extensions row to target columns) |
| `LoginGuideTop`, `LoginGuideBottom` | login screen |
| `SecondaryAuthenticationGuideTop/Bottom`, `TotpRegisterGuideTop`, `TotpRegisterBottom` | 2FA screens |

- **Language suffixes do not make files exclusive.** `Foo_ja.html` gives position `Foo`, language `ja`, but each file is a separate entry (`Initializer.ExtendedHtmls`). So for a `ja` user, both `Foo.html` and `Foo_ja.html` are output, and an `en` user still gets `Foo_ja.html`, because it falls back to the first body. Keep **one file per position without a suffix**, and if you need per-language text, put it in a site/Extended script that picks the text at runtime. Test with each language you support.
- **Avoid other underscores in the name.** `My_Banner.html` is read as position `My`, language `Banner`.

Per-column HTML can also be injected without files, through the column settings or server scripts: `ExtendedHtmlBeforeField`, `BeforeLabel`, `BetweenLabelAndControl`, `AfterControl`, `AfterField`.

## 3.7 Security

- **Nothing in Scripts, Styles, Htmls, Extended* or dashboard `CustomHtml` is escaped.** Anyone who can edit site settings can run JS as every viewer of the site. Restrict site-admin permission accordingly.
- **Never build HTML from record values with string concatenation.** Values typed by users end up in your DOM. Use `$('<span>').text(value)` or `textContent`, not `.html(value)`.
- **Never put secrets (API keys, passwords) in client code.** Use a server script with `httpClient`, and keep the secret out of the script body if you can.
- **CSP** (`Security.json` → `ContentSecurityPolicy`, default `Enabled: false`, `ReportOnlyEnabled: true`) adds a nonce to every `<script>` that Pleasanter emits:
  - Site Scripts and ExtendedScripts get the nonce, so they keep working when CSP is enforced.
  - `<script>` tags written inside **Htmls / ExtendedHtmls / column HTML / CustomHtml have no nonce** and are blocked under enforced CSP. **Put JS in Scripts, not in Htmls.**
  - External CDNs need `script-src` / `connect-src` changes. With `'strict-dynamic'`, host allow-lists are ignored, so load external libraries from a nonce'd script, or host them yourself.
  - Watch `/CspReport/Report` (report-only mode) before you enable enforcement.

## 3.8 Dashboards

- **Inline `<script>` inside a CustomHtml part is not reliable.** It does not run on the initial GridStack load, but it does run after an Ajax refresh of the part. Put the logic in the **dashboard site's Scripts** (Index/All) and target `#DashboardPart_<id>`.
- Dashboards fire `on_grid_load` (action `index`). There is no per-part load event.

## 3.9 Checklist for a site script

- [ ] Wrapped in an IIFE, no global `const`/`let`, shared helpers under `$p.ex.<feature>`
- [ ] `$p.events.*` handlers chained, idempotent, and return `false` only to cancel on purpose
- [ ] Uses delegated events and `$p.set` (not `.val()`) to change values
- [ ] Output targets (`All`/`Edit`/`Index`…) are as narrow as possible, and the dialog editor case is handled
- [ ] No hard-coded site or record IDs, no secrets
- [ ] Every rule it enforces is also enforced on the server
- [ ] Tested with `?Draft=1` before replacing the live version
