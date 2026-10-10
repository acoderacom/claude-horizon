# Project: Shopify theme design work

All rules below are user decisions. Do not re-litigate them.

## Store
- Store: [agent: insert the store's myshopify.com domain] ([agent: insert "development store" or "live store"])
- Live theme: Horizon [agent: insert the live theme's Horizon version], ID [agent: insert the live theme's ID, gid://shopify/OnlineStoreTheme/…]
- Storefront password (visitor gate on development and password-protected stores): [agent: insert the password in backticks, or delete this line if the storefront has none]. Claude does not type passwords into websites, so the user enters it once in the browser pane; the pane remembers it.

## Files in this folder
- `theme/`: local copy of the live theme.
- `THEME.md`: technical spec of stock Horizon [agent: insert the Horizon version THEME.md covers] for agents: how the theme is built and how to customize it. The single source of truth for theme work. If it and this file disagree, this file wins for workflow, store access, deployment and its design rules; `THEME.md` wins for how the theme works and how to build within those rules (`THEME.md` §0.3).
- `customizations.md`: log of every change made to the theme.
- `.mcp.json`: setup for the `shopify-admin-mcp` connection (package `@acodera/shopify-admin-mcp`), including its access key. Never print or share the key. Live-theme writes are switched on; local file uploads are only allowed from `uploads/`. Also sets up `shopify-dev-mcp` (Shopify's docs and code checker; it reads local files and never touches the shop).
- `uploads/`: the only folder `shopify-admin-mcp` may upload local files from (images, PDFs) into the store's Files library. Put files here before uploading; never point uploads at the project folder, which holds the access key.

## Connecting to the shop
- **IMPORTANT:** Only use the local `shopify-admin-mcp` MCP server to talk to the shop. Never use the claude.ai Shopify connector, even if it shows as connected.
- Theme edits go straight to the live theme through `shopify-admin-mcp` (`shopify_theme_files_upsert`, which runs GraphQL `themeFilesUpsert`). Never use its `shopify_theme_duplicate` or `shopify_theme_delete` tools. Store data (products, collections, metafields, metaobjects) also goes through `shopify-admin-mcp`.
- No git. No Shopify CLI. No draft or duplicate theme. That rules out updating Horizon to a new version here, since an update installs the new version as an unpublished copy: if the user asks for one, explain this and stop.
- The access key has full access to the store: themes, products, collections, metafields, metaobjects, files, pages, blogs, navigation menus, translations, markets, discounts, inventory, orders, customers, checkout, shipping and payments settings. Because a mistake can affect real store data, only change what the task needs. Ask the user before touching orders, customers, payments, checkout or shipping, and before deleting anything.

## How to edit a theme file (in this order)
1. **IMPORTANT:** Check the live copy first. The live theme in Shopify is the source of truth, not the local `theme/` folder, and the theme editor can change files at any time. If the local file and its live copy differ, sync the local file from the shop (pull live, overwrite local). Never push a local file over a newer live one. The theme editor rewrites `templates/*.json`, `sections/*-group.json` and `config/settings_data.json` on every save: pull those right before editing and keep every key you did not mean to change (`THEME.md` §4.1).
2. Once per task, check the version: the live `config/settings_schema.json` must report `theme_version` [agent: insert the Horizon version THEME.md covers], the version `THEME.md` covers. If it does not, stop and follow `THEME.md` §0.1 (run setup again).
3. Edit the local file in `theme/`, following `THEME.md`.
4. Validate before pushing: run `shopify-dev-mcp` `validate_theme` on the changed files in the local `theme/` folder and fix what it finds. Before "fixing" a finding in a stock file, compare it with the known stock findings in `THEME.md` §17.2.
5. Push the file's content, so local and live stay in sync. Push one file at a time. Each push is live at once, so when a change has several files, push every file before the files that refer to it, in this order: locale keys; assets; snippets (including the `snippets/stylesheets.liquid` line that links `cx-theme.css`); `config/settings_schema.json`; blocks (child and private `_cx-` blocks before anything that lists them); sections; `layout/`; then the JSON that names them: `templates/*.json`, `sections/*-group.json`, and last `config/settings_data.json`. Shopify treats a template, section group or schema that names a section or block that doesn't exist yet as an error.
6. After each push, check the page on the storefront. Use Claude's built-in browser pane, not Chrome or headless Chrome. When a change adds settings or blocks, also check them in the theme editor if the browser pane can open it; otherwise tell the user exactly what to check (`THEME.md` §17.1).
7. Log the change in `customizations.md` (see below).

## Design rules
- Follow the ladder in `THEME.md` §2.1: store data, theme settings, and existing sections and blocks come before any code. When code is needed, the order in the rules below applies (a new theme block before a stock edit, unless the edit is a small addition).
- Prefer settings changes (`config/settings_data.json`) over Liquid edits.
- **IMPORTANT:** Build with Horizon's theme blocks. First use what exists: settings, and existing sections (such as the generic `section` and its presets) filled with existing theme blocks (`group`, `text`, `image`, `button`, … and their presets; `THEME.md` levels L1–L3). Only write code when that cannot do the job, and say why.
- **IMPORTANT:** When code is needed, make a new public `cx-` theme block with a preset (L6, `THEME.md` §2.2). It plugs into every stock section and block that accepts any theme block (`@theme`, `THEME.md` §3.6) without editing them, and on a Horizon update it is copied over as is. Edit a stock file (L5) only for a small addition to the file that already does the job, because every stock edit must be merged again by hand into each new Horizon version. Build a new `cx-` section (L7) only for a page-level region that sections plus blocks cannot express.
- For anything at L4–L8 (custom Liquid, stock code edits, new code), write the justification from `THEME.md` §2.3 in `customizations.md` before writing code. Name every new block, section, snippet, asset, CSS class and custom element with the `cx-` prefix (`_cx-` for private blocks), and put new locale keys under `cx.` (`THEME.md` §2.5). New page templates keep a plain name the merchant can read (for example `page.faq.json`).
- Leave `assets/base.css` stock. Global CSS rules go in a small custom stylesheet (see `THEME.md` §9.9: `assets/cx-theme.css`).

## Logging changes
- **IMPORTANT:** `customizations.md` must be updated in the same task as the change, for both kinds of change: (1) any edit to a stock Horizon file, and (2) any newly created custom code (section, block, snippet, asset, template). Record what its "How to write an entry" section asks: file, level, what it does or what changed, settings, why (with the `THEME.md` §2.3 justification for L4–L8), checks, and how to undo. It is the only record of how this theme differs from stock Horizon, apart from merchant edits made in the theme editor (`THEME.md` §2.4).
