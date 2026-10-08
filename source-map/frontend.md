# Frontend (Implem.PleasanterFrontend)

Unless a path says otherwise, it is relative to `Implem.PleasanterFrontend/wwwroot/src/`. C# paths start with `Implem.Pleasanter/`.

## Build and how the server loads bundles

- **`wwwroot/package.json`:**
  - `build` = `tsc` + `vite build` (main) + `vite build --config vite.config.app.ts`.
  - `dev` runs the watchers and browser-sync (configs in `wwwroot/dev/`).
- **Vite configs:**
  - `vite.config.ts`: entry `scripts/modules.ts` + every non-`_` `styles/*.scss`. Writes `manifest.json`. Copies `clone/**` → `Implem.Pleasanter/wwwroot/assets/...` and `plugins/**` → `assets/plugins`.
  - `vite.config.app.ts`: entry `scripts/app.ts` as an IIFE. Writes `app.manifest.json`.
  - `vite.config.shared.ts`: output dir `../../Implem.Pleasanter/wwwroot/assets`. Its `@` alias points to a dir that does not exist.
- **Vitest:**
  - `vitest.unit.config.ts`: happy-dom, setup `src/test/setup.ts` stubs `$p`.
  - `vitest.browser.config.ts`: Playwright, runs `*.browser.test.ts`.
- **MSBuild:** `Implem.PleasanterFrontend/Implem.PleasanterFrontend.csproj` (NoTargets) runs `npm install`/`npm run build`. It wipes `assets` when the git branch changes.
- **Entries:**
  - `scripts/app.ts` → `generals/index.ts`: the legacy `$p`. Import order is fixed in `index.ts`.
  - `scripts/modules.ts` → `modules/index.ts`: custom elements.
- **C# side of loading:**
  - `Libraries/Manifests/ManifestLoader.cs` reads the manifests.
  - `Libraries/HtmlParts/HtmlScripts.cs` emits, in order:
    1. the plugin `<script>`s;
    2. the manifests (`wwwroot/components/manifest.json` → `assets/app.manifest.json` → `assets/manifest.json`);
    3. the extended, site and user scripts;
    4. the `$p.execEvents('on_*_load')` calls.
  - `HtmlHeadLinks.cs` adds modulepreload links.
  - `HtmlStyles.cs` `LinkedStyles()` picks `legacy*.css` for ThemeVersion < 2.0 and `style*.css` otherwise.
  - `Libraries/Resources/JavaScripts.cs` `ViewMode()` maps each view mode to its init call (`$p.setGrid`, `setCalendar`, `drawGantt`…).

## $p core: client ↔ server

- **`_init.js`** creates `window.$p = {data, events, ex, modal, store}`. `$p.data[formId]` holds pending POST values.
- **`_form.js` `$p.send($control)`:**
  - Reads `data-action` / `data-method` from the control.
  - Builds the URL by replacing `_action_` in the form `action`, e.g. `/items/{id}/_action_` → `/items/{id}/update`.
  - Runs validation when the control has `.validate`, then calls `$p.ajax`.
- **`_ajax.js`:** `$p.ajax` (CAPTCHA, `data-confirm`, CSRF header, `ajax_*` events, `setByJson` on success) and `$p.multiUpload`.
- **Button → controller mapping:**
  - Tracing a button: a C# `HtmlControls.Button(onClick: "$p.send($(this));", action: "Update", method: "put")` → `PUT /items/{id}/update` → `Controllers/ItemsController.cs` `Update()` → `ItemModel.Update` → `<Table>Utilities.Update`.
  - Supporting pieces: the form action comes from `Libraries/Responses/Locations.cs`, and the data attributes from `Libraries/Html/HtmlAttributes.cs` (`DataAction` / `DataMethod`).
  - **To find the server handler for a button:** take its `data-action` value and look for the action of the same name in the controller.
- **`_dispatch.js` `$p.setByJson`** applies a C# `Libraries/Responses/ResponseCollection.cs` (`{Method, Target, Value, Options}`).
  - Per-table response collections: `ResponseSpecials.cs` (gen), `ResponseViews.cs`, `ResponseViewModes.cs`, `ResponseFilterDialogs.cs`, `ResponseLookups.cs`.
  - Supported methods: Html, ReplaceAll, Message, Href, PushState, Set, SetData, SetFormData, SetMemory, Append/Prepend/After/Before, InsertText, Remove, Attr/RemoveAttr/Css, Focus, SetValue (`Val()`), ClearFormData, CloseDialog, Paging, Toggle, Trigger, **Invoke** (`$p[target](value)`), Events, Click, scroll/disabled/log.
  - Gap: `SetValueAndFormData` (`ValAndFormData()`) has no client case.
- **`_event.js` `$p.execEvents(name)`:**
  - Calls `$p.events[name]`, then the `_<controlId>` variant, then the `_<data-action>` variant.
  - Hooks: before/after_setData, validate, send and set; `on_*_load`.
- **`_data.js`:** getData/set/setData/setMustData/clearData.
- **`_elements.js`:** context getters (`id`, `siteId`, `userId`…), `getControl`/`getField`/`getValue`, grid helpers.
- **`_controllevents.js`:** `.auto-postback` / `.control-auto-postback`.
- **`_view.js`:** `setValue`.
- **`_api.js`:** `$p.api*` REST wrappers posting to `api/{controller}/{id}/{action}`.
- **`display.ts`** is **generated** from `Implem.Pleasanter/App_Data/Definitions/Definition_Code/ClientDisplays*`, with strings from `App_Data/Displays/*.json` that have `ClientScript: true`.

## Feature → file (`scripts/generals/` unless a path is given)

| Feature | File(s) |
|---|---|
| Messages / loading / dialogs | message.js, messageevents.js, loading.js, dialog.js, dialogevents.js |
| Custom modal / image viewer | modal/ui-modal.ts, modal/imageViewerModal.ts |
| jQuery UI init (tabs, multiselect, datepicker, spinner, slider), Simple mode | jqueryui.js |
| Validation rules | validator.js |
| Unsaved-change warning | confirm.js, confirmevents.js |
| Record prev/next/reload/new/copy, global search | item.js, navigation.js, navigationevents.js, searchevents.js |
| Grid (infinite scroll, edit on grid, edit in dialog, selectedIds) | grid.js, observer.js, scroll.js |
| Grid checkboxes / sort / header menu | gridevents.js |
| `<grid-container>` | grid-container/grid-container.ts |
| Bulk update | bulkupdate.js |
| Aggregations | aggregationevents.js |
| Filters reset / negative label / range dialogs | filterevents.js, viewfilterslabelevents.js, setdaterangedialog.js, setnumericrangedialog.js |
| View mode switch | viewmode.js |
| Calendar (FullCalendar) | calendar.ts, calendarevents.ts |
| Kamban | kamban.js, kambanevents.js |
| Gantt / BurnDown / TimeSeries / Analy (d3) | gantt.js, ganttevents.js, burndown.js, burndownevents.js, timeseries.js, analy.js |
| Crosstab | crosstab.js, crosstabdrilldown.js |
| ImageLib | imagelib.js |
| Dashboard (GridStack) | dashboard.js, scrollevents.js |
| Attachments upload / D&D | attachments.js, attachmentsevents.js |
| Camera capture | video.js |
| Dropdown search | dropdownsearch.js, dropdownsearchevents.js |
| Relating (parent/child) dropdowns | relatingcolumns.js, multiselect.js |
| Selectable / basket / column picker | selectable.js, basket.js, basketevents.js, fieldselectable.js |
| Site settings dialogs | sitesettings.js (about 30 open*Dialog functions), sitesettingsevents.js |
| Editor columns UI (theme 2) | modules/editorColumns/*.ts |
| Process / status control / column / view access control | process.js, processevents.js, statuscontrol.js, columnaccesscontrol.js, view.js |
| Formula | formula.js |
| Import / export | import.js, export.js |
| Site package / multilingual labels / templates | sitepackage.js, multilinguallabel.js, template.js |
| Site menu / move / separate / delete site | sitemenu.js, move.js, separate.js, site.js |
| Permissions / groups | permission.js, group.js |
| Outgoing mail | outgoingmail.js, outgoingmailevents.js |
| Login / password | security.js, generatepassword.js, showpassword.js |
| 2FA (mail, TOTP, QR) / passkeys | authenticatebymail.js, splittotpform.js, qr.js, passkey.ts |
| Tenant / SCIM token / BG server script schedule | tenants.js |
| Switch user back | users.js |
| Announcement / start guide | announcement.js, startguide.js |
| Side menu / responsive | menuevents.js, responsive.js |
| Keyboard (Enter/Tab) | keyevents.js |
| Current time/user/dept buttons | controloptionevents.js |
| Anchor textbox / status color / record lock / expand | anchor.js, anchorevents.js, statusevents.js, lockevents.js, expand.js |
| Page info (tableName/controller/action) | siteinfo.js |
| Date helpers | _time.js |

## Modules (`scripts/modules/`, custom elements)

The C# emitters are in `Implem.Pleasanter/Libraries/HtmlParts/HtmlTags.cs` (~810–925).

| Element | File | Notes |
|---|---|---|
| `<date-field>` | datefield.ts | flatpickr + moment |
| `<code-editor>` | codeEditor/codeEditor.ts | CodeMirror 6, used for script/style settings |
| `<markdown-field>` | markdownField/markdownField.ts | marked + DOMPurify + highlight.js |
| `<rt-editor>` | richTextEditor/richTextEditor.ts | SunEditor 3. Has unit + browser tests |
| `<ui-carousel>` | uiCarousel/uiCarousel.ts | Glide.js |
| editor columns | editorColumns/initialize.ts (+ filter, dropdown, toastmenu, popup…) | Loaded only when `#ThemeVersionOver2` exists |
| misc | displaynone.ts, preventHrefJump.ts | |

## Styles, plugins, types

- **`styles/`:**
  - `legacy.scss` is for theme < 2.0.
  - `style.scss` is for theme ≥ 2.0. It `@use`s legacy, and its sections are marked `/* X ---- */` (Controls, Header, Editor, Dialog, gridTable…).
  - `*-responsive.scss` holds the mobile styles.
  - `_plugins.scss` pulls in the plugin CSS.
- **Component styles:** shadow-DOM styles sit next to each component's TS file and are imported with `?inline`.
- **Third-party libraries:** `plugins/` is copied verbatim (jQuery, jQuery UI, d3, fullcalendar, gridstack, …). `clone/assets/` holds images and 24 jQuery UI themes.
- **Types (`types/`):**
  - `pleasanter-script.d.ts` is the **public user-script API** (`$p.*`).
  - `pleasanter-server-script.d.ts` is the server-script API.
  - `pleasanter.d.ts` types the internal `$p`.
- **Where the public `$p` API is implemented:**
  - `siteinfo.js`: tableName, controller, action
  - `_elements.js`: id, siteId, getControl, getField, getValue, getGridRow…
  - `grid.js`: selectedIds
  - `_data.js`: set
  - `_form.js`: send
  - `process.js`: execProcess
  - `message.js`: setMessage
  - `_api.js`: api*
  - `_event.js`: events

## Svelte components (`Implem.PleasanterFrontend/components/`)

- **Build:** `npm run build` writes to `Implem.Pleasanter/wwwroot/components/`. That output is **committed** and is not built by MSBuild.
- **Components:** `src/main.ts` exports `lib/SmartDesign/Main.svelte` (`<smart-design>`) and `Trigger.svelte` (`<smart-design-link>`).
- **Server side:**
  - `<smart-design>` loads data with `GET items/{siteId}/smartdesign` and saves with `UpdateSmartDesign`. These map to `ItemsController.SmartDesign` / `UpdateSmartDesign`.
  - The C# emitter is `HtmlTags.SmartDesignLink`, called from `HtmlNavigationMenu.cs`.
- **Paths:** `tsconfig.app.json` maps `@pleasanter/*` → `../wwwroot/src/*`.
