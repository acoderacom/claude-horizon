# Project: Shopify theme work on a live site

All rules below are user decisions. Do not re-litigate them.

## Store
- Store: [agent: insert the store's myshopify.com domain] (live store)
- Theme to edit: [agent: insert the theme's name and version], ID [agent: insert the theme's ID, gid://shopify/OnlineStoreTheme/…]
  - An unpublished copy of the live theme, which the user made in the Shopify admin (Online Store → Themes → ⋯ → Duplicate) and publishes there when it's ready. Setup sets this line from the copy the user picks; never change it yourself. To edit another copy, the user runs setup again.
  - Claude never edits the live theme. The server enforces it: live-theme writes are off, and themes are locked, so creating, duplicating, renaming, deleting and publishing themes are refused.
- Storefront password (visitor gate on password-protected stores): [agent: insert the password in backticks, or delete this line if the storefront has none]. Claude does not type passwords into websites, so the user enters it once in the browser pane; the pane remembers it.

## Files in this folder
- `theme/`: local copy of the theme to edit. The user puts its files here by hand (Shopify admin → Online Store → Themes → ⋯ on that theme → Download theme file, then unzip into `theme/`). Never fill it yourself by pulling the whole theme; while it's empty, ask the user to download the theme. After that, `shopify_theme_files_pull` keeps the files you edit current.
- `THEME.md`: technical spec of this store's theme, written by Claude from the theme's own files when the user asks (see "Writing THEME.md"). Once written, the single source of truth for how the theme works. If it and this file disagree, this file wins for workflow, store access, deployment and its design rules; `THEME.md` wins for how the theme works and how to build within those rules.
  - It starts blank and stays blank until the user asks you to learn the theme. Never write it on your own. While it's blank, read the files in `theme/` that the task needs instead, and where these rules point to `THEME.md`, take the answer from those files.
  - Once written, read its map at the top first, then only the parts it lists for the task.
- `customizations.md`: log of every change made to the theme.
- `.mcp.json`: setup for the `shopify-admin-mcp` connection (package `@acodera/shopify-admin-mcp`), including its access key. Never print or share the key, and never change its settings: live-theme writes are off and themes are locked (see "Store"), and only setup changes that, when the user runs it again. Theme files are sent from and pulled into `theme/`; local file uploads are only allowed from `uploads/`. Also sets up `shopify-dev-mcp` (Shopify's docs and code checker; it reads local files and never touches the shop).
- `uploads/`: the only folder `shopify-admin-mcp` may upload local files from (images, PDFs) into the store's Files library. Put files here before uploading; never point uploads at the project folder, which holds the access key.

## Connecting to the shop
- **IMPORTANT:** Only use the local `shopify-admin-mcp` MCP server to talk to the shop. Never use the claude.ai Shopify connector, even if it shows as connected.
- Theme edits go straight to the theme to edit through `shopify-admin-mcp` (`shopify_theme_files_upsert`, which runs GraphQL `themeFilesUpsert`, always with that theme's ID), and `shopify_theme_files_pull` copies its files from Shopify into `theme/`. The user makes copies and publishes them in the Shopify admin; the server refuses those actions. Store data (products, collections, metafields, metaobjects) also goes through `shopify-admin-mcp`.
- **IMPORTANT:** Store data is shared by every theme, so a change to it shows on the live site at once, whichever theme you edit. Ask the user before changing any store data (orders, customers, payments, checkout and shipping included), even for a design on the copy, and before deleting anything.
- No git. No Shopify CLI. A new version of the theme installs in the Shopify admin as another unpublished theme; moving to it follows the update tip in `customizations.md`.
- The access key has full access to the store: themes, products, collections, metafields, metaobjects, files, pages, blogs, navigation menus, translations, markets, discounts, inventory, orders, customers, checkout, shipping and payments settings. Because a mistake can affect real store data, only change what the task needs.

## Writing THEME.md
- **IMPORTANT:** Write `THEME.md` only when the user asks (for example "learn the theme"), from the files the user put in `theme/`. Before you start, check that they are the theme to edit: `shopify_themes_list` must show the ID on the "Theme to edit" line as unpublished (`UNPUBLISHED` or `DEVELOPMENT`), not live (`MAIN`), and when that line shows a name and version in parentheses, `theme/config/settings_schema.json` → `theme_info` must give the same. If anything doesn't match, stop and tell the user.
- If `customizations.md` still has an `[agent: …]` placeholder for the theme's name and version (setup couldn't read `theme_info`), fill it in from `theme_info`.
- The baseline is the copy as the user downloaded it, not the theme as its author shipped it: on a live store, the merchant, an agency or an app has often changed it before. Treat those earlier changes as part of the theme, and note any you can see (code naming another developer or an app, files the theme's own structure doesn't explain) as observed, not verified. `customizations.md` logs only changes made in this project, from this baseline. In these rules, "stock" means the baseline.
- Write only what the files show, citing `file:line` for every claim. Mark anything you couldn't confirm as unverified. Never fill a gap with what other themes do.
- Start with the theme it covers (name, author and version from `config/settings_schema.json` → `theme_info`, the theme ID, the date written), its baseline, and a map: which parts to read for which kind of task. Then cover, in this order:
  1. Files: what each folder holds, the layout files, which templates are JSON and which Liquid, and the section groups.
  2. Global settings: the groups and IDs in `config/settings_schema.json`, the color system (schemes or palette), typography, and how settings become CSS custom properties.
  3. Sections: for each, what it shows, its settings, the blocks it accepts (its own, theme blocks through `@theme`, app blocks through `@app`), its presets, and the templates or groups that use it.
  4. Theme blocks in `blocks/`, if the theme has any: settings, nesting, and which sections accept them.
  5. Snippets shared across sections, with their parameters.
  6. CSS: the files and where they load, class naming, breakpoints, custom properties, and where a site-wide `assets/cx-theme.css` should be linked.
  7. JavaScript: the files, custom elements, events, and how sections load their scripts.
  8. Locales: the storefront and schema files and how their keys are structured.
  9. Customizing: for each level in the design rules below (L0–L8), what this theme offers and where, and which stock files a small addition would go into.
  10. Known findings: run `shopify-dev-mcp` `validate_theme` on the whole baseline once and list what it reports, so those findings aren't "fixed" later as if this project caused them.
- Describe, don't copy: no pasted file contents beyond a line or two.
- Keep it current: when a change adds or alters something `THEME.md` describes, update it in the same task.

## How to edit a theme file (in this order)
1. **IMPORTANT:** Check Shopify's copy first. The theme to edit in Shopify is the source of truth, not the local `theme/` folder, and the theme editor can change files at any time. Before editing, pull the files you'll change with `shopify_theme_files_pull`: it overwrites local copies that differ from Shopify's and leaves matching ones alone. Never push a local file over a newer one in Shopify. The theme editor rewrites `templates/*.json`, `sections/*-group.json` and `config/settings_data.json` on every save: pull those right before editing and keep every key you did not mean to change.
2. Once per task, check the theme: the theme with the ID on the "Theme to edit" line must still exist and still be unpublished, and when that line (or `THEME.md`, once written) gives a `theme_info` name and version, its `config/settings_schema.json` in Shopify must still give the same. If anything doesn't match (for example the copy was published), stop and tell the user: they run setup again to pick a copy and copy its files into `theme/` again, and `THEME.md` has to be written again if it's a different theme.
3. Edit the local file in `theme/`, following `THEME.md`. For L4–L8, write the justification in `customizations.md` before the code (see Design rules).
4. Validate before pushing: run `shopify-dev-mcp` `validate_theme` on the changed files in the local `theme/` folder and fix what it finds. Before "fixing" a finding in a stock file, compare it with the known findings in `THEME.md`.
5. Push with `shopify_theme_files_upsert`, giving each file only its `filename`: the server sends it from `theme/`, so never paste file content into the call. Each file in its report should be stored "as sent". A JSON file may come back "changed by Shopify", since Shopify regenerates JSON files that aren't in its own format; the server then updates `theme/` to match. If the server says no theme folder is set, run setup again. Push one file at a time. Each push takes effect at once on the copy, so when a change has several files, push every file before the files that refer to it, in this order: locale keys; assets; snippets (including the line that links `cx-theme.css`, if the theme links stylesheets from a snippet); `config/settings_schema.json`; blocks (child and private `_cx-` blocks before anything that lists them); sections; `layout/` (where most themes link stylesheets, so the `cx-theme.css` link usually goes here; `THEME.md` says where); then the JSON that names them: `templates/*.json`, `sections/*-group.json`, and last `config/settings_data.json`. Shopify treats a template, section group or schema that names a section or block that doesn't exist yet as an error.
6. After each push, check the page through the copy's preview link, `https://<store>/?preview_theme_id=<the number at the end of its ID>`, and compare it with the approved design. Use Claude's built-in browser pane, not Chrome or headless Chrome. When a change adds settings or blocks, also check them in the theme editor if the browser pane can open it; otherwise tell the user exactly what to check.
7. Log the change in `customizations.md`, completing the entry started in step 3 for L4–L8 (see below).

## Design rules
- **IMPORTANT:** The approved design (a screenshot, a design file or the user's description) is the spec. Match it at desktop and phone widths, detail by detail. A stock option that only comes close does not do the job: go to the next step below until it matches. Never simplify or change the design to fit the theme as it is on your own: before building, list each difference you expect and why, and ask the user.
- **IMPORTANT:** Work through these in order and stop at the first that matches the design. The levels (L0–L8) are what `customizations.md` records:
  1. Store data (products, collections, metafields, metaobjects, menus, pages, Search & Discovery filters, apps; L0) and global theme settings (`config/settings_data.json`: colors, fonts, buttons, cards; L1). A global setting changes every page, so use it for site-wide choices, never to fit one section.
  2. Existing sections and blocks, with their presets and settings, and new JSON templates (`templates/*.json`, `sections/*-group.json`; L2–L3).
  3. A `custom-liquid` section or block, if the theme has one (L4), only for a few lines of Liquid with no CSS, settings or JS.
  4. New code that leaves stock files alone. If the theme's sections accept theme blocks (`@theme`, see `THEME.md`), a new public `cx-` theme block with a preset (L6): it plugs into those sections without editing them, and on a theme update it is copied by hand into the new version as it is. Edit a stock file instead (L5) only for a small addition to the file that already does the job, such as a new option on a stock section; every stock edit must be merged again by hand into each new theme version. Keep stock edits additive, with stock behavior as the default. If matching the design needs more than a small addition to one stock file (several edits, or replacing its markup, CSS or JS), stop before writing code and offer the user the choices with their cost: new options on the stock file, a `cx-` section that replaces it, or a design change.
  5. A new `cx-` section (L7), for a page-level region the stock sections and blocks cannot express, for anything a theme without theme blocks needs added inside a section, or to replace a stock section when the user picks that.
  6. Snippets, assets and locale keys that serve L5–L7 changes, and stock strings reworded for the brand (L8).
- A style detail no setting covers goes in `assets/cx-theme.css` (site-wide) or in the `cx-` file it belongs to; never drop it. Leave the theme's own CSS files stock.
- For L4–L8, write the justification (`customizations.md`, "Justification") before writing code.
- Name every new block, section, snippet, asset, CSS class and custom element with the `cx-` prefix (`_cx-` for private blocks), and put new locale keys under `cx.`. New page templates keep a plain name the merchant can read (for example `page.faq.json`).

## Before the user publishes
The copy is a snapshot of the live theme from when the user duplicated it, and the merchant may have changed the live theme since, in the theme editor or through an app. Publishing the copy would undo those changes. When the user plans to publish, or asks whether the copy is ready:
1. List both themes' files with `shopify_theme_files_list` and compare their checksums. Never pull the live theme into `theme/`; that folder holds the copy.
2. For each file that differs, decide whether this project changed it (`customizations.md` says). For the JSON files that differ for any other reason (`templates/*.json`, `sections/*-group.json`, `config/settings_data.json`, `locales/*.json`), read the live version with `shopify_theme_files_get`, which writes nothing, and find what changed there.
3. Show the user every difference this project didn't make, file by file: the live theme has it and publishing the copy would lose it.
4. Bring a change over to the copy only when the user says so, as an ordinary edit with its `customizations.md` entry.
5. Publishing is the user's, in the Shopify admin. Store data isn't affected by it; app embeds and theme settings are, since they live in `config/settings_data.json`.

## Logging changes
- **IMPORTANT:** `customizations.md` must be updated in the same task as the change, for both kinds of change: (1) any edit to a file in the baseline, and (2) any newly created custom code (section, block, snippet, asset, template). Record what its "How to write an entry" section asks: file, level, what it does or what changed, settings, why (with the justification for L4–L8), checks, and how to undo. It is the only record of what this project changed in the theme, apart from merchant edits made in the theme editor.
