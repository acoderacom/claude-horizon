# Project: Shopify theme design work

All rules below are user decisions. Do not re-litigate them.

## Store
- Store: [agent: insert the store's myshopify.com domain] ([agent: insert "development store" or "live store"])
- Live theme: Horizon [agent: insert the live theme's Horizon version], ID [agent: insert the live theme's ID, gid://shopify/OnlineStoreTheme/…]
- Storefront password (visitor gate on development and password-protected stores): [agent: insert the password in backticks, or delete this line if the storefront has none]. Claude does not type passwords into websites, so the user enters it once in the browser pane; the pane remembers it.

## Files in this folder
- `theme/`: local copy of the live theme.
- `THEME.md`: technical spec of stock Horizon [agent: insert the Horizon version THEME.md covers] for agents: how the theme is built and how to customize it. The single source of truth for theme work. If it and this file disagree, this file wins for workflow, store access, deployment and its design rules; `THEME.md` wins for how the theme works and how to build within those rules (`THEME.md` §0.3).
  - Never read `THEME.md` whole. Read it from the top through §2, then only the sections its map ("How to read this file", at the top) lists for the task, which for any change include §18 and §17.
- `customizations.md`: log of every change made to the theme.
- `.mcp.json`: setup for the `shopify-admin-mcp` connection (package `@acodera/shopify-admin-mcp`), including its access key. Never print or share the key. Live-theme writes are switched on; theme files are sent from and pulled into `theme/`; local file uploads are only allowed from `uploads/`. Also sets up `shopify-dev-mcp` (Shopify's docs and code checker; it reads local files and never touches the shop).
- `uploads/`: the only folder `shopify-admin-mcp` may upload local files from (images, PDFs) into the store's Files library. Put files here before uploading; never point uploads at the project folder, which holds the access key.

## Connecting to the shop
- **IMPORTANT:** Only use the local `shopify-admin-mcp` MCP server to talk to the shop. Never use the claude.ai Shopify connector, even if it shows as connected.
- Theme edits go straight to the live theme through `shopify-admin-mcp` (`shopify_theme_files_upsert`, which runs GraphQL `themeFilesUpsert`), and `shopify_theme_files_pull` copies live files into `theme/`. Never use its `shopify_theme_duplicate` or `shopify_theme_delete` tools. Store data (products, collections, metafields, metaobjects) also goes through `shopify-admin-mcp`.
- No git. No Shopify CLI. No draft or duplicate theme. That rules out updating Horizon to a new version here, since an update installs the new version as an unpublished copy: if the user asks for one, explain this and stop.
- The access key has full access to the store: themes, products, collections, metafields, metaobjects, files, pages, blogs, navigation menus, translations, markets, discounts, inventory, orders, customers, checkout, shipping and payments settings. Because a mistake can affect real store data, only change what the task needs. Ask the user before touching orders, customers, payments, checkout or shipping, and before deleting anything.

## How to edit a theme file (in this order)
1. **IMPORTANT:** Check the live copy first. The live theme in Shopify is the source of truth, not the local `theme/` folder, and the theme editor can change files at any time. Before editing, pull the files you'll change with `shopify_theme_files_pull`: it overwrites local copies that differ from live and leaves matching ones alone. Never push a local file over a newer live one. The theme editor rewrites `templates/*.json`, `sections/*-group.json` and `config/settings_data.json` on every save: pull those right before editing and keep every key you did not mean to change (`THEME.md` §4.1).
2. Once per task, check the version: the live `config/settings_schema.json` must report `theme_version` [agent: insert the Horizon version THEME.md covers], the version `THEME.md` covers. If it does not, stop and follow `THEME.md` §0.1 (run setup again).
3. Edit the local file in `theme/`, following `THEME.md`. For L4–L8, write the `THEME.md` §2.3 justification in `customizations.md` before the code (see Design rules).
4. Validate before pushing: run `shopify-dev-mcp` `validate_theme` on the changed files in the local `theme/` folder and fix what it finds. Before "fixing" a finding in a stock file, compare it with the known stock findings in `THEME.md` §17.2.
5. Push with `shopify_theme_files_upsert`, giving each file only its `filename`: the server sends it from `theme/`, so never paste file content into the call. Each file in its report should be stored "as sent". A JSON file may come back "changed by Shopify", since Shopify regenerates JSON files that aren't in its own format; the server then updates `theme/` to match. If the server says no theme folder is set, run setup again. Push one file at a time. Each push is live at once, so when a change has several files, push every file before the files that refer to it, in this order: locale keys; assets; snippets (including the `snippets/stylesheets.liquid` line that links `cx-theme.css`); `config/settings_schema.json`; blocks (child and private `_cx-` blocks before anything that lists them); sections; `layout/`; then the JSON that names them: `templates/*.json`, `sections/*-group.json`, and last `config/settings_data.json`. Shopify treats a template, section group or schema that names a section or block that doesn't exist yet as an error.
6. After each push, check the page on the storefront and compare it with the approved design (`THEME.md` §17.1). Use Claude's built-in browser pane, not Chrome or headless Chrome. When a change adds settings or blocks, also check them in the theme editor if the browser pane can open it; otherwise tell the user exactly what to check (`THEME.md` §17.1).
7. Log the change in `customizations.md`, completing the entry started in step 3 for L4–L8 (see below).

## Design rules
- **IMPORTANT:** The approved design (the mockup, a screenshot or the user's description) is the spec. Match it at desktop and phone widths, detail by detail. A stock option that only comes close does not do the job: go to the next step below until it matches. Never simplify or change the design to fit stock Horizon on your own: before building, list each difference you expect and why, and ask the user.
- **IMPORTANT:** Work through these in order and stop at the first that matches the design (`THEME.md` §2.1):
  1. Store data (products, collections, metafields, metaobjects, menus, pages, Search & Discovery filters, apps; L0) and global theme settings (`config/settings_data.json`: palette, fonts, buttons, cards; L1). A global setting changes every page, so use it for site-wide choices, never to fit one band.
  2. Existing sections and theme blocks, with their presets and settings (`templates/*.json`, `sections/*-group.json`; L2–L3): for example the generic `section` filled with `group`, `text`, `image` and `button` blocks.
  3. A `custom-liquid` block (L4), only for a few lines of Liquid with no CSS, settings or JS.
  4. A new public `cx-` theme block with a preset (L6, `THEME.md` §2.2). It plugs into every stock section and block that accepts any theme block (`@theme`, `THEME.md` §3.6) without editing them, and on a Horizon update it is copied by hand into the new version as it is. Edit a stock file instead (L5) only for a small addition to the file that already does the job, such as a new option on a stock section; every stock edit must be merged again by hand into each new Horizon version. Keep stock edits additive, with stock behavior as the default. If matching the design needs more than a small addition to one stock file (several edits, or replacing its markup, CSS or JS), stop before writing code and offer the user the choices in `THEME.md` §2.1 with their cost: new options on the stock file, a `cx-` section that replaces it, or a design change.
  5. A new `cx-` section (L7), only for a page-level region that sections plus blocks cannot express, or to replace a stock section when the user picks that.
- A style detail no setting covers goes where `THEME.md` §9.10 says; never drop it. Leave `assets/base.css` stock; site-wide CSS goes in `assets/cx-theme.css` (`THEME.md` §9.9).
- For L4–L8, write the `THEME.md` §2.3 justification in `customizations.md` before writing code.
- Name every new block, section, snippet, asset, CSS class and custom element with the `cx-` prefix (`_cx-` for private blocks), and put new locale keys under `cx.` (`THEME.md` §2.5). New page templates keep a plain name the merchant can read (for example `page.faq.json`).

## Designing a mockup
Design in two passes, and keep them apart.

1. **Free design.** Be creative: design what suits the brand and its products, as a designer would, not as a theme developer. In this pass don't read `THEME.md` or look at `theme/`, and don't limit the design to what stock Horizon can do. The user decides on custom code later, band by band. Only three limits apply: name the colors as a small palette, use fonts from Shopify's font library, and leave out the parts Shopify owns (below). Design both desktop and phone widths, with real-looking copy.
2. **Horizon map.** Only after the user approves the look. Don't change the design while mapping unless the user asks.
   - Give every band a map, shown by an overlay the user can switch on:
     - `data-horizon`: the stock section, its preset, its blocks and the settings that produce the band (`THEME.md` §5.3, §6.3, §11).
     - `data-code` for anything stock can't produce: the level (`THEME.md` §2.1), the `cx-` file or `cx_style` value, and in one line why stock can't do it.
   - Check every `data-horizon` claim against the stock source in `theme/`. Fix the notes, not the design.
   - Match the colors to the theme's color palette (`THEME.md` §10) and flag any fonts that aren't in Shopify's font library.
   - Then show the user a list, band by band, in three groups: built from stock; close in stock but not exact (what differs); needs custom code (level, `cx-` file, and why). Don't build until the user has answered every item in the last two groups.

- **Parts Shopify owns:** the sheet that opens from the header's account icon, the customer account pages and checkout. Leave them out of the free design. If the user asks for them, design them only within what the theme can set (`THEME.md` §5.4) and say so in the map.

## Logging changes
- **IMPORTANT:** `customizations.md` must be updated in the same task as the change, for both kinds of change: (1) any edit to a stock Horizon file, and (2) any newly created custom code (section, block, snippet, asset, template). Record what its "How to write an entry" section asks: file, level, what it does or what changed, settings, why (with the `THEME.md` §2.3 justification for L4–L8), checks, and how to undo. It is the only record of how this theme differs from stock Horizon, apart from merchant edits made in the theme editor (`THEME.md` §2.4).
