# THEME.md — Horizon 4.2.0 technical specification

| Field | Value |
|---|---|
| Theme | Horizon by Shopify |
| **Version covered** | **4.2.0** only |
| Version source of truth | `config/settings_schema.json` → `[0].theme_version` (`config/settings_schema.json:5`) |
| Source snapshot | Shopify/horizon commit `f9aef27` (the commit `versions.json` names; release 2026-09-18) |
| Spec written | 2026-10-10 |
| Scope | Stock Horizon 4.2.0 files. Paths are relative to the theme root (the folder holding `config/settings_schema.json`); every `path:line` points at stock 4.2.0. |
| Audience | AI coding agents. Terse, normative. |

## How to read this file

MUST NOT read this file whole. Read by section:

1. Always: from the top to the `## 3.` heading (this map and §0–§2).
2. Every change: §18 before choosing how to build it; §17 before and after pushing.
3. Then only the sections the map lists for the task. Follow a `§` reference only when the task needs what it points to.

Get every heading's line number with one search (`grep -n '^##' THEME.md`), then read from a heading to the next one. For one name (block type, section, setting ID, snippet, token, class, event), search for the name instead of reading a whole catalog. This file maps the source; still read every theme file a change touches (§1).

| § | Covers | Read when |
|---|---|---|
| 0 | Version gate, labels, precedence, golden rules | Always |
| 1 | Render tree, folders | Always |
| 2 | Ladder L0–L8, justification record, change log, `cx-` naming, do-not list | Always |
| 3 | Theme blocks: kinds, targeting, `content_for`, static vs dynamic, `closest.*`, which containers accept which blocks (§3.6), nesting, saved-data contract | Adding, nesting or listing blocks (L3, L5, L6) |
| 4 | Template and section-group JSON: ownership, format, stock templates, groups | Editing `templates/*.json` or `sections/*-group.json`; alternate templates |
| 5 | Sections: generic `section` and its presets, section skeleton, catalog, header (§5.4) | Choosing a stock section; writing a `cx-` section (L7); header work |
| 6 | Blocks: authoring conventions, standard setting groups, public and private catalogs, parent coupling, schema rules, Liquid style | Choosing a stock block; writing or editing a block (L5, L6) |
| 7 | `layout/theme.liquid` render order, shell facts, editing layouts | Editing `layout/`; page width, scroll container, page-wide markup |
| 8 | Snippets: style helpers, component renderers, style owners | Rendering buttons, media, prices, icons and the like; writing a snippet |
| 9 | CSS: layers, rules, tokens, global classes, typography, spacing, CSS for a `cx-` block, global CSS (§9.9), where a style detail goes and the `cx_style` option (§9.10) | Writing any CSS; matching a design detail no setting covers |
| 10 | Color model, `contrast-override`, color-aware CSS | Color settings or color CSS |
| 11 | Every `config/settings_schema.json` group and setting ID | Theme settings (L1); reading `settings.*`; editing `settings_schema.json` |
| 12 | JavaScript: loading and import map, `Component`, events, section rendering, theme editor, utilities | Writing or changing any JS |
| 13 | Product page, featured product, product cards, collection and search, cart, quick add, metafields and dynamic sources | Work on any of these |
| 14 | Storefront and editor strings, `cx.` locale keys | Adding text shoppers or merchants see |
| 15 | Accessibility | Changing markup or interaction |
| 16 | Performance: images, loops, deferred content | Adding images, loops or deferred content |
| 17 | Verification checklist; `validate_theme` findings in stock files | Every change |
| 18 | Recipes: common need → level and stock pieces | Every change |
| 19 | Known stock defects and quirks | Copying stock code; before "fixing" stock code that looks wrong |
| 20 | Keys of the PLATFORM labels → shopify.dev URLs | Checking the docs behind a PLATFORM label |
| A | Code templates: A.1 leaf block, A.2 interactive block + JS module, A.3 section, A.4 locale keys, A.5 template JSON | Writing that kind of file |

## 0. Version gate, labels, precedence

### 0.1 Version gate (once per task)

1. Read the live `config/settings_schema.json` `[0].theme_version`.
2. `== "4.2.0"`: this spec applies.
3. Otherwise stop: edit no theme files, and do not update this file to match. Tell the user the live theme is Horizon <live version> while this project's docs cover 4.2.0, and to run setup again (`npx -y @acodera/shopify-admin-mcp@latest setup`, **Full theme design**). Setup moves the project to the live version when the template supports it, or says which version to switch the store to.
4. `release-notes.md` lists what 4.2.0 changed (RTL `dir` attribute, logical start/end alignment values, cart bfcache fixes).

### 0.2 Labels

| Label | Meaning |
|---|---|
| `path:line` | Verified in stock 4.2.0 source. |
| PLATFORM | Shopify platform behavior from shopify.dev (key in §20), not provable from theme source. |
| UNVERIFIED | Not proven by source or docs. Test before relying on it. |
| MUST / MUST NOT | Hard rule; breaking it damages saved merchant data, the editor or the storefront. |
| SHOULD | Default; deviate only with a reason in the change log. |

### 0.3 Precedence

1. Project instructions (`CLAUDE.md`): workflow, store access, deployment and design rules.
2. This file: how the theme works and how to change it, within the design rules in `CLAUDE.md`.
3. Shopify platform docs (§20) for anything this file does not cover.

### 0.4 Golden rules

1. **Horizon first.** Walk the ladder (§2.1) top-down and stop at the first level that works: one that matches the approved design at desktop and phone widths, not one that comes close (§2.1). L5 vs L6 is decided by the §2.1 rule, not by order. Custom code needs a written justification (§2.3).
2. **Theme blocks are the unit of work.** Compose with existing sections, blocks and presets; when code is unavoidable the default artifact is a new theme block, not a section (§2.2).
3. **Saved data is a contract.** MUST NOT rename, remove or repurpose block/section types (file names), setting IDs, setting `type`s, select/radio values, static block `id`s or preset keys referenced by templates (§3.8).
4. **JSON data files belong to the editor.** `templates/*.json`, `sections/*-group.json`, `config/settings_data.json` are rewritten on every save: read the current copy, merge, never overwrite (§4.1).
5. **Static block contract.** `{% content_for 'block', type: T, id: I %}` ⇄ JSON key `I` with `"type": T, "static": true`, never in `block_order` (§3.4).
6. **Private blocks (`_name`)** appear only where a parent lists them; `@theme` never includes them (§3.2).
7. **CSS:** one `{% stylesheet %}` per file, no Liquid in it; instance values via inline custom properties or `{% style %}`; theme tokens; logical properties; render style-owner snippets wherever their markup is reused (§9).
8. **Color:** no color schemes. Custom colors go through `snippets/contrast-override.liquid` + `color-custom-<id>` (§10).
9. **JS:** native HTML first; components extend `Component`, use `ref`/`on:` attributes, clean up on disconnect; no new runtime dependencies (§12).
10. **Strings:** storefront text → `locales/en.default.json` + `t`; editor labels → `locales/en.default.schema.json` + `t:`; shop content → settings/metafields (§14).
11. **Validate** every changed file with `validate_theme`; compare findings with the stock baseline (§17.2) before "fixing".
12. **Record** every stock edit and new file in `customizations.md` with level, checks and, for L4–L8, the justification (§2.4).
13. **Prefix `cx-`** every new block, section, snippet, asset, CSS class, custom element and locale namespace (§2.5).

## 1. Architecture at a glance

```
layout/theme.liquid                          (password page: layout/password.liquid)
├─ <head>  favicon, view-transition-opt-in, meta-tags, stylesheets (base.css), card-hover-effect-styles,
│          fonts, scripts (import map), theme-styles-variables (tokens), color-palette, theme-editor
│          (editor only), {{ content_for_header }}
├─ skip-to-content-link, chat-drawer
├─ #header-group   {% sections 'header-group' %} → sections/header-group.json (header-announcements, header)
├─ <main id="MainContent">  {{ content_for_layout }} → templates/<type>[.<suffix>].json → sections → blocks
├─ <footer>        {% sections 'footer-group' %} → sections/footer-group.json (footer, footer-utilities)
└─ overlays  cart-drawer (cart_type=drawer, not on cart template), theme-drawer, search-modal,
             quick-add-modal (quick_add or mobile_quick_add)
```

Render units, top-down: **layout → section group / template (JSON) → section (`sections/*.liquid`) → theme blocks (`blocks/*.liquid`, nestable 8 deep) → snippets (`snippets/*.liquid`, no schema)**.

| Folder | Files | Role | See |
|---|---|---|---|
| `layout/` | 2 | Document shells: global markup, shared CSS/JS, section groups, `content_for_header`/`content_for_layout` | §7 |
| `templates/` | 13 (12 JSON + `gift_card.liquid`) | Page composition. Editor-owned JSON. | §4 |
| `sections/` | 42 (40 `.liquid` + 2 group `.json`) | Page regions composing static + dynamic theme blocks | §5 |
| `blocks/` | 95 (44 public, 51 private `_*`) | Nestable components with own settings and targeting | §3, §6 |
| `snippets/` | 145 | `{% render %}` partials; 42 `*-styles` (36 own a `{% stylesheet %}`, 2 emit `{% style %}`, 4 emit inline custom properties) | §8 |
| `assets/` | 125 (81 JS, 3 CSS, 35 SVG, 6 dev files) | Flat; no subfolders; no build step | §9, §12 |
| `config/` | 2 | `settings_schema.json` (definitions), `settings_data.json` (saved values, editor-owned) | §11 |
| `locales/` | 57 (34 storefront, 23 schema) | UI strings + editor labels | §14 |

No `schemas/` folder: section and block schemas are the JSON inside `{% schema %}`; `config/settings_schema.json` is hand-edited. Search for an existing section, block, snippet, setting group, style helper or JS component before adding one, and read every file involved before editing it. This file is a map; the source is the territory.

## 2. How to customize: the ladder

User decision: **use Horizon as much as possible; custom code is allowed, but every custom change must say why Horizon could not do it.** Horizon can do it only when the result matches the approved design.

### 2.1 Levels

| Level | Mechanism | Code? | Keeps automatic updates? | Typical use |
|---|---|---|---|---|
| L0 | Store data: products, collections, metafields + definitions, metaobjects, menus, pages, Search & Discovery filters, apps | No | Yes | Content, navigation, filters, data for dynamic sources |
| L1 | Global theme settings (`config/settings_data.json` `current`) | No | Yes | Palette, fonts, type scale, buttons, inputs, cards, badges, cart type, page width, animations |
| L2 | Section/block settings on existing instances, incl. dynamic sources (`{{ closest.product.metafields.ns.key }}`) | No | Yes in `templates/*.json`; no in `sections/*-group.json` | Copy, media, layout, colors, spacing of one band or block |
| L3 | Composition in JSON: add/remove/reorder/nest existing sections and blocks; presets; alternate templates `templates/<type>.<suffix>.json` | JSON only | Yes in `templates/*.json`; no in `sections/*-group.json` (only `settings_data.json` and `templates/*.json` are outside the update check, PLATFORM [UPD]) | New page layouts; new bands from `section` presets + `group`/`text`/`image`/`button`… |
| L4 | `custom-liquid` block/section (Liquid stored as a setting value) | Liquid in data | Yes in `templates/*.json`; no in `sections/*-group.json` | A few lines of content-level Liquid (metafield print, embed). No `layout`/`section`/`javascript`/`stylesheet`/`schema` tags, no `content_for_*`, no `settings`, ≤50 KB (PLATFORM [IS]); `closest.*` inside it UNVERIFIED |
| L5 | Small addition to a stock file that already owns the behavior | **Stock edit** | No: manual re-merge per update | Behavior change inside a component used everywhere |
| L6 | New theme block `blocks/cx-*.liquid` | New file | No (any changed or added file stops automatic updates; added files UNVERIFIED, assume they stop them) | Component no stock block owns |
| L7 | New section `sections/cx-*.liquid` | New file | No | Page-level region with its own data loop, grid or JS that `section` + blocks cannot express |
| L8 | Supporting code: `snippets/cx-*`, `assets/cx-*` (incl. `assets/cx-theme.css`), locale keys | New/edited | No | Only in service of L5–L7 |

The order to work in (as in `CLAUDE.md`'s design rules and `customizations.md`), stopping at the first step that matches the design:
1. store data and global theme settings (L0–L1);
2. existing sections and theme blocks, with their presets and settings (L2–L3);
3. a `custom-liquid` block, only for a few lines (L4);
4. a new `cx-` theme block (L6), unless the change is a small addition to the stock file that already does the job: then edit that file (L5);
5. a new `cx-` section only for a page-level region that sections plus blocks cannot express, or to replace a stock section when the user picks that (L7).

Rules:
- A level works only when the result matches the approved design (mockup, screenshot or the user's description) at desktop and phone widths: layout, spacing, type, color, borders, shadows, hover and motion. If it only comes close, go to the next level. Changing the design to fit stock needs the user's approval (`CLAUDE.md`). A style detail no setting covers goes where §9.10 says; never drop it.
- L1 changes every page: use it for site-wide choices, never to fit one band; a band that differs needs L2 or a later step.
- L0–L4 keep automatic updates, except changes to `sections/*-group.json` (header, footer, announcement bar), which is not in the exception: "A theme is automatically updated only if all theme files, except `settings_data.json` and any JSON files in the `/templates` directory, are in their original state" (PLATFORM [UPD]).
- L5 vs L6: choose L5 only when the stock file already owns the responsibility and the change is small and additive; otherwise L6, even when L5 is possible. A manual update installs the new version as an unpublished copy (PLATFORM [UPD]); `cx-` files are copied into it as they are and stock edits are merged again, both by hand. Do not copy a stock block into a `cx-` file and edit the copy unless the stock block must stay unchanged elsewhere; say so in the justification.
- Keep every L5 edit additive: stock behavior stays the default (a new setting or option, a new allowed block type, an added line), and no existing setting changes what it does (§3.8).
- More than a small addition: when matching the design needs several edits to one stock file, or replaces its markup, CSS or JS instead of adding to it (for example rebuilding the header's menu or drawer), stop before writing code, offer the user these choices with their cost, and build the one they pick:
  1. new options on the stock file (L5): stock stays the default; every edit is merged again by hand on each Horizon update;
  2. a `cx-` section that replaces the stock one (L7), reusing stock snippets and blocks where it can; it may start as a copy of the stock file (say so in the justification). It is copied into a new version as it is, but stock fixes to the original no longer reach it, and the merchant's settings on the stock section don't carry over, so they are set again;
  3. a change to the design.
- L7 only for page-level needs: first prove that `sections/section.liquid` (13 presets; `@theme`, `@app`, `_divider`) plus blocks cannot do it, or have the user pick it to replace a stock section (rule above).
- L4 that grows beyond a few lines or needs CSS, settings or JS becomes a `cx-` block.

### 2.2 Theme blocks first

The default code artifact is a **public theme block** (`blocks/cx-name.liquid` with a preset). With a preset it appears at once in every container that accepts `@theme`: sections `section`, `_blocks`, `hero`, `main-page`, `main-404`, `password`, `product-list`, `collection-list`, `main-collection-list`, `main-blog`, `main-blog-post`, `main-cart`, `product-recommendations`, `product-information`, `featured-product-information`; blocks `group`, `popup-link`, `featured-collection`, `product-recommendations`, `_card`, `_slide`, `_layered-slide`, `_accordion-row`, `_content-without-appearance`, `_product-details`, `_product-list-content` (§3.6). (`predictive-search` declares `@theme` but never renders a block area; `_content` and `_collection-info` are orphans, §6.4.) Merchants add, remove, reorder and configure it without further code, and no stock file changes. Section-defined blocks "only work in the section they're defined within" (PLATFORM [BLK]). Make it private (`_cx-name`) only when it must be restricted to one parent, and list it in that parent's `blocks` array (a stock edit if the parent is stock).

### 2.3 Justification record (L4–L8)

Write this in the change-log entry before writing code:

```
Need:            <observable behavior the merchant/user needs>
Stock checked:   <sections/blocks/settings/presets evaluated, with file names>
Why not stock:   <the specific missing capability, per candidate>
Level chosen:    L5 | L6 | L7 | L8  (+ L4 if custom-liquid)
Files:           <new or edited files>
Settings/IDs:    <new setting IDs, block types, static ids, locale keys>
Upgrade impact:  <stock files edited (merge again by hand on update) / new files (copy as they are on update)>
```

### 2.4 Change log contract

Every stock-file edit and every new file MUST be recorded in `customizations.md` in the same task, using its entry format: path and kind (stock-modified vs new); level; what changed (quote one-line stock edits); settings touched (for `settings_data.json`: ID, stock value, new value); for L4–L8 the §2.3 block, written once in the main file's entry and referenced from the others; checks run and checks not possible (§17.1); how to undo. The log is the only record of how the theme differs from stock, apart from merchant edits made in the editor (those live in the JSON files and are not logged).

### 2.5 Naming for custom code

| Artifact | Pattern | Example |
|---|---|---|
| Public / private block | `blocks/cx-<name>.liquid` / `blocks/_cx-<name>.liquid` | `blocks/cx-countdown.liquid` |
| Section | `sections/cx-<name>.liquid` | `sections/cx-store-locator.liquid` |
| Snippet / style owner | `snippets/cx-<name>.liquid` / `snippets/cx-<name>-styles.liquid` | `snippets/cx-badge-styles.liquid` |
| JS module | `assets/cx-<name>.js`; import-map key `@theme/cx-<name>` only if imported | `assets/cx-countdown.js` |
| Custom element | `cx-<name>-component` | `<cx-countdown-component>` |
| CSS classes / vars | root = file name: `.cx-<name>`, `__el`, `--mod`; vars `--cx-<name>-*` | `.cx-countdown__value` |
| Schema locale keys | `t:cx.<name>.<key>` (top-level `cx` object in `en.default.schema.json`) | `t:cx.countdown.end_time` |
| Storefront locale keys | `cx.<name>.<key>` (top-level `cx` object in `en.default.json`) | `'cx.countdown.days' \| t` |
| DOM ids | `Cx<Name>-{{ block.id }}` / `-{{ section.id }}` | `CxCountdown-{{ block.id }}` |
| Alternate template | `templates/<type>.<suffix>.json`, plain readable suffix (it is the name merchants pick, PLATFORM [ALT]); no `cx-` | `templates/page.faq.json` |

`cx-` is collision-proof (stock already uses `custom-liquid`, `.button-custom`, `.custom-typography`) and greppable for upgrade merges. Reuse stock `t:settings.*`, `t:options.*`, `t:content.*`, `t:categories.*`, `t:info.*` keys for standard settings.

### 2.6 Hard "do not" list

- MUST NOT rename/remove/re-type block or section files, setting IDs, option values, static ids or preset block keys used in templates: "changing the ID of a setting is equivalent to removing the current setting and adding a new one with the default value" (PLATFORM [UPD]).
- MUST NOT edit `assets/base.css` for project styling (§9.9).
- MUST NOT put Liquid inside `{% stylesheet %}` / `{% javascript %}` (not rendered; Theme Check error).
- MUST NOT add runtime JS dependencies, bundlers or subfolders in `assets/`.
- MUST NOT add `custom_css` keys to JSON (merchant-owned).
- MUST NOT branch on `block.id` / `section.id` values (Theme Check `BlockIdUsage` for `block.id`; same rule for `section.id`); use them only for scoping.
- MUST NOT build on orphan private blocks (§6.4) without wiring them into a parent.
- MUST NOT modify or wrap `{{ content_for_header }}`.

## 3. Theme blocks: the system

### 3.1 Kinds

| Kind | Horizon 4.2.0 |
|---|---|
| Theme block (`blocks/<type>.liquid`, `type` = file name) | All 95 blocks. Reusable, nestable, can be static. |
| Section-defined (local) block | **None.** A section uses local blocks or theme blocks, never both; theme blocks cannot define local blocks (PLATFORM [BLK][TBS]). Do not introduce them. |
| App block (`@app`) | Accepted by most content containers (§3.6). |

### 3.2 Targeting (schema `blocks` array)

| Entry | Meaning |
|---|---|
| `{"type": "@theme"}` | Every **public** block that has a preset. Never `_*` blocks. |
| `{"type": "@app"}` | App blocks. `limit` not allowed on `@app`. |
| `{"type": "<name>"}` only | Only the listed types. |
| `@theme` + explicit types | Listed types shown first ("recommended"), the rest under "Show all"; private types listed here become available. |

- Private blocks are available only where listed (e.g. `_divider` in `sections/section.liquid:16-26`).
- Picker visibility needs a preset. Public blocks without one are placed only statically or by JSON: `accelerated-checkout`, `add-to-cart`, `filters`, `page-content` (no `presets` key); `contact-form-submit-button`, `featured-collection` (`"presets": []`). Stock `templates/page.json` places `page-content` dynamically via JSON.
- Static-only children need not be listed in the parent's `blocks` (`contact-form` has no `blocks` key yet renders static `contact-form-submit-button`, `blocks/contact-form.liquid:2`; `carousel` section has no `blocks` key).
- Theme Check `ValidBlockTarget`: every preset child must be covered by the parent's root `blocks`.

### 3.3 Rendering tags

```liquid
{% content_for 'blocks' %}                                     {% # dynamic area, saved order %}
{% content_for 'blocks', closest.product: product %}           {% # + resource context for children %}
{% content_for 'block', type: 'group', id: 'static-header' %}  {% # one static block %}
{% content_for 'block', type: '_product-card', id: 'static-product-card',
   closest.product: product, product_view_context: 'collection' %}
```

- Render the dynamic area once per file; `capture` it when several branches need it (`blocks/group.liquid:1-3`, `sections/section.liquid:1-3`).
- Stock calls `content_for` only in sections and blocks. To place block output inside a snippet, capture it and pass the HTML (`sections/product-information.liquid:5-23` → `snippets/product-information-content.liquid` params `media_gallery`, `product_details`, `additional_blocks`). `content_for` inside a snippet: UNVERIFIED.
- Keyword args on a static block become plain variables in the block; declare them with `@param` in its `{% doc %}`. Filters on `content_for` args are ignored → `assign` first (Theme Check `UnsupportedFilterArguments`). Reserved param names: `attributes, block, blocks, class, context, id, inherit, resource, resources, schema, section, sections, settings, snippet, snippets, src, style, styles, template, templates, type`.
- A block sees `block`, the rendering `section`, globals and static keyword args; never caller-local variables.

### 3.4 Static vs dynamic

| | Static (`content_for 'block'`) | Dynamic (`content_for 'blocks'`) |
|---|---|---|
| Editor | hide + configure; no reorder/remove/duplicate | add/remove/reorder/duplicate/configure |
| Liquid | conditional, looped or repeated renders allowed | rendered once in saved order |
| JSON | `"static": true`; key = Liquid `id`; **never in `block_order`** | in `block_order` |
| Presets | `"static": true` + key = `id`; absent statics are still created with defaults | must be in the preset |
| Limits | not counted | ≤50 per section |

Contract (MUST): `id` is a string literal unique within its immediate parent (Shopify never generates static ids); never rename `I` or change `T` once merchants have saved data.

Stock patterns:
- **Fixed structure:** `media-gallery` + `product-details` (`sections/product-information.liquid:6-18`), `header-logo` (`sections/header.liquid:82`).
- **Repeated static block** (one settings record, N renders) inside a loop with per-item context: `_product-card#static-product-card` (`sections/product-list.liquid:37-42`), `_product-card#product-card` (`sections/main-collection.liquid:70-75`), `_collection-card#static-collection-card` (`sections/collection-list.liquid:56`), `_blog-post-card#static-blog-post-card` (`sections/main-blog.liquid:151-159`), `_collection-link#link` (`sections/collection-links.liquid:56-62`).
- **Render variants:** same id with different keyword args: `_header-menu#header-menu` default / `variant: 'mobile'` / `variant: 'navigation_bar'` (`sections/header.liquid:86-94`).
- **Exclusive branches:** `sections/product-list.liquid:37-42` populated vs `:55` placeholder loop.
- **Template copy:** rendered inside `<template id="empty-cart-template">` with `force_empty: true` and live (`sections/main-cart.liquid:26,29` vs `:45,49`).
- **Context nulling:** `closest.product: null` stops cards inheriting the template product (`sections/product-list.liquid:55`, `sections/carousel.liquid:27`).

`validate_theme` flags every reuse of one static id in a file ("The id 'X' is already being used by another static block"): `header` (`header-menu`), `main-cart` (`cart-page-title`, `cart-page-items`), `product-list` (`static-product-card`), `collection-links` (`link`). These are intentional (§17.2); reuse the pattern only with the same intent.

### 3.5 Context: `closest.*`

- `closest.product | collection | article | blog | page | metaobject.<type>` resolves (PLATFORM [TBD]): nearest `content_for … closest.X:` setter → nearest ancestor block resource setting of that type (incl. own) → section resource setting → template resource.
- Horizon reads `closest.*` directly in Liquid (`blocks/buy-buttons.liquid:4`, `blocks/price.liquid:5`, `sections/product-information.liquid:2`) although docs call it settings-only; treat Horizon usage as the working contract.
- Settings and presets bind dynamic sources: `"text": "<h1>{{ closest.product.title }}</h1>"` (`templates/product.json:110`), `"product": "{{ closest.product }}"` (`:393`), `"{{ closest.collection.title }}"` (`templates/collection.json:19`).
- Explicit setters in stock: `sections/product-information.liquid:9,17` (`closest.product: closest.product`), `sections/featured-product.liquid:46,50` (`section.settings.product`), card loops (`closest.product: product`, `closest.collection: collection`; `closest.article: article` only in `sections/featured-blog-posts.liquid:46`, `main-blog` passes a plain `article:` keyword), `blocks/_product-card.liquid:10` and `blocks/_collection-card.liquid:12` forward to children.
- Implicit: `featured-product-information` passes no `closest.product`; blocks resolve it from the section's `product` setting (PLATFORM). `featured-collection` block's `collection` setting becomes `closest.collection` for its children (PLATFORM).
- Limits: 100 dynamic sources per JSON template, 50 per setting (PLATFORM [DS]).

### 3.6 Containers (who accepts what)

Blocks:

| Block | `@theme` | `@app` | Explicit (incl. private) |
|---|---|---|---|
| `group` | ✓ | ✓ | `_divider` |
| `popup-link`, `featured-collection`, `_accordion-row`, `product-recommendations` | ✓ | ✓ | — |
| `_card` | ✓ | ✓ | `text`, `_heading`, `image`, `video`, `product-card`, `collection-card` |
| `_slide`, `_layered-slide` | ✓ | ✓ | `_heading`, `button`, `text`, `group`, `image`, `video`, `icon`, `jumbo-text` |
| `_content-without-appearance` | ✓ | ✓ | `text`, `icon`, `image`, `button`, `video`, `group`, `spacer`, `_divider` |
| `_product-details` | ✓ | ✓ | `text`, `icon`, `image`, `button`, `video`, `group`, `spacer`, `accordion` + product blocks + `_divider` (§13.1) |
| `_product-list-content` | ✓ | ✓ | `text`, `button`, `spacer`, `_divider`, `_product-list-text`, `_product-list-button` |
| `accordion` | ✗ | ✗ | `_accordion-row` |
| `_carousel-content` | ✗ | ✗ | `_card` |
| `_marquee` | ✗ | ✗ | `text`, `icon`, `_divider` |
| `collection-card`, `_collection-card` | ✗ | ✗ | `text`, `spacer`, `button`, `group`, `collection-title` |
| `product-card`, `_product-card`, `_product-card-group` | ✗ | ✓ | card list (§13.3) + `custom-liquid` |

Orphans not listed (unreachable, §6.4): `_content` (`@theme`, `@app`, `_divider`), `_collection-info` (`@theme`, `@app`), `_footer-social-icons` (`_social-link`).

Sections:

| Section | Accepts |
|---|---|
| `section`, `_blocks`, `main-page`, `main-404`, `password`, `product-list` | `@theme`, `@app`, `_divider` |
| `collection-list`, `main-collection-list`, `main-blog`, `product-recommendations` | `@theme`, `@app`, recommended `text, icon, image, button, video, group, spacer, _divider` |
| `main-cart` | same without `_divider` |
| `main-blog-post` | `@theme`, `@app`, `_blog-post-title`, `_blog-post-info-text`, `_blog-post-featured-image`, `_blog-post-content` |
| `product-information`, `featured-product-information` | `@theme`, `@app` (area renders **after** the product grid, outside Quick Add) |
| `hero` | `@theme`, `text`, `button`, `logo`, `jumbo-text`, `spacer`, `group`, `_marquee`; **no `@app`** |
| `footer` | `_divider, @app, button, follow-on-shop, group, icon, image, menu, payment-icons, text, logo, jumbo-text, social-links, email-signup`; **no `@theme`** |
| `footer-utilities` | `footer-copyright, footer-policy-list, social-links` (`max_blocks: 3`) |
| `marquee` | `text, icon, logo, _divider` |
| `header-announcements` | `_announcement` |
| `slideshow` / `layered-slideshow` | `_slide` / `_layered-slide` (`max_blocks: 6`) |
| `product-hotspots` | `_hotspot-product` |
| `featured-blog-posts` | `_featured-blog-posts-title, _featured-blog-posts-card, _featured-blog-posts-image` |
| `predictive-search` | `@theme` in schema only; no `content_for 'blocks'` call, so added blocks never render |
| no `blocks` key (`quick-order-list`: `[]`) | `carousel`, `collection-links`, `featured-product`, `header`, `media-with-content`, `main-collection`, `search-header`, `search-results`, `logo`, `divider`, `custom-liquid`, `quick-order-list`, `cart-drawer-section`, `password-footer`, `predictive-search-empty`, `section-rendering-product-card` |

A new public `cx-` block with a preset is available in every `@theme` container above. Restricted parents (`footer`, `marquee`, cards, …) need an L5 edit to list it.

### 3.7 Nesting, limits, wrapper

- ≤8 nesting levels below the section; ≤50 merchant-addable blocks per section; ≤1,250 per JSON template or section group (static blocks do not count); ≤300 files in `blocks/`; ≤25 sections per template or group; `blocks/*.liquid` ≤256 KB (PLATFORM [LIM]).
- Block schema keys: `name` (≤25 chars resolved), `settings`, `blocks`, `presets`, `tag`, `class`. `limit`, `max_blocks`, `enabled_on`, `disabled_on` are section-only.
- Default wrapper `<div id="shopify-block-{id}" class="shopify-block {class}">` with editor attributes. With `"tag": null` (86 of 95 blocks) there is no wrapper: the file MUST output one root element carrying `{{ block.shopify_attributes }}` once; `class` then has no effect. 8 blocks have no `tag` key and use the wrapper: `_blog-post-card`, `_blog-post-content`, `_blog-post-info-text`, `_featured-blog-posts-title`, `icon`, `page`, `page-content`, `product-recommendations` (`icon.liquid:18` and `product-recommendations.liquid:52` also print the attributes inside, §19). `product-custom-property` uses `"tag": "product-custom-property-component"`.
- New blocks: `"tag": null` + attributes on the single root.

### 3.8 Saved-data contract

| Change | Effect |
|---|---|
| Rename/remove a block or section file | Instances break (Theme Check `JSONMissingBlock`/`JSONMissingSection`); editor drops them |
| Rename/remove a setting ID or change its `type` | Saved value lost → default |
| Remove/rename a select/radio value | Saved value invalid → default |
| Narrow a `range` | Out-of-range values reset |
| Repurpose an ID, option value or block type | Saved values apply with the wrong meaning, silently |
| Rename a static `id` | Saved settings orphaned; new id starts at defaults |
| Remove a type from a parent's `blocks` | Existing children invalid |
| Add `enabled_on`/`disabled_on` | Existing placements invalid |

Safe: add settings (with defaults), options, accepted block types, presets; change labels/`info`; widen ranges. Before renaming or removing a section setting, grep `blocks/` and `snippets/` for `section.settings.<id>` (§6.5).

## 4. Templates and section groups (JSON)

### 4.1 Ownership

- `templates/*.json`, `sections/*-group.json`, `config/settings_data.json` start with an "auto-generated … may be overwritten" comment (`templates/product.json:1-9`) and are rewritten on every editor save. Read the **live** copy, merge your change, keep every unrelated key and every id.
- Parsers must accept a leading `/* … */` (all JSON templates, both groups, `settings_data.json`, `en.default*.json`) and `//` comments (`en.default.json`, `en.default.schema.json`).
- Ids: stock uses semantic (`main`, `media-gallery`) and generated (`hero_jVaWmY`, `group_icgrde`) ids. Docs say alphanumeric; Horizon uses `-`/`_`. Safe charset `[a-z0-9_-]`; new dynamic ids `<type>_<6 random alphanumerics>`.

### 4.2 Format

```jsonc
{
  "sections": {
    "<section_id>": {
      "type": "<sections/ file name>", "name": "t:names.x",   // name optional
      "disabled": true,                                       // optional; unused in stock
      "settings": { "<id>": <value> },
      "blocks": {
        "<static_id>": { "type": "<block>", "static": true, "settings": {}, "blocks": {}, "block_order": [] },
        "<block_id>":  { "type": "<block>", "settings": {}, "blocks": {}, "block_order": [] }
      },
      "block_order": ["<block_id>"]                           // dynamic children only
    }
  },
  "order": ["<section_id>"],
  "layout": "password"                                        // optional; only templates/password.json
}
```

- Each nested parent has its own `blocks` + `block_order`. Values may be palette references `"{{ settings.color_palette.<slot> }}"`, dynamic sources, raw colors (`#00000026`, `rgba(...)`), `shopify://collections/all` links.
- Static blocks are added with their container's preset using defaults (PLATFORM [TBST]); a static block called in Liquid but absent from saved JSON rendering with schema defaults: UNVERIFIED.
- Limits: 25 sections per template, 50 blocks per section, 1,000 JSON templates, 512 KB per template (PLATFORM [LIM]). Alternate templates `templates/<type>.<suffix>.json` (stock: `page.contact.json`). `gift_card` stays `.liquid`.

### 4.3 Stock templates

| Template | Sections | Key static blocks |
|---|---|---|
| `index.json` | `hero` → `product-list` | `static-header` (`_product-list-content`), `static-product-card` (`_product-card` → gallery, title, price) |
| `product.json` | `product-information` (`main`) → `product-recommendations` | `media-gallery` (`_product-media-gallery`), `product-details` (`_product-details` → group(title text + price), `_divider`, `variant-picker`, `buy-buttons` (static `quantity`, `add-to-cart`, `accelerated-checkout`), text(description)); dynamic `disclosures` |
| `collection.json` | `section` (title + description texts) → `main-collection` | `filters`, `product-card` (`_product-card` → gallery, title, price, swatches) |
| `list-collections.json` | `main-collection-list` | `static-collection-card` (`_collection-card` → `collection-title`, static `collection-card-image`) |
| `search.json` | `search-header` → `search-results` | `heading` (`_heading`), `search` (`_search-input`); `filters`, `product-card` |
| `cart.json` | `main-cart` → `product-list` | `cart-page-title`, `cart-page-items`, `cart-page-summary` |
| `blog.json` | `main-blog` | `static-blog-post-card` (`_blog-post-card` → static `heading`, `blog-post-details`, `image`) |
| `article.json` | `main-blog-post` | none (dynamic `text`, `_blog-post-info-text`, `_blog-post-featured-image`, `_blog-post-content`) |
| `page.json` | `main-page` | none (dynamic `text` = `<h1>{{ closest.page.title }}</h1>`, `page-content`) |
| `page.contact.json` | `main-page` → `section` | `contact-form` → static `submit-button` |
| `404.json` | `main-404` → `product-list` | product-list statics |
| `password.json` | `password` (layout `password`) | none |
| `gift_card.liquid` | `{% layout none %}`, own import map | — |

Deepest stock nesting in templates: 3.

### 4.4 Section groups

- `sections/header-group.json` (`"type": "header"`) and `footer-group.json` (`"type": "footer"`): template shape + `type`, `name`; rendered by `{% sections %}` (`layout/theme.liquid:42,164`).
- Header group: `header-announcements` (one `_announcement`) → `header` (static `header-logo`, `header-menu`; the `header-menu` block carries `menu: main-menu`, `menu_style: featured_products`).
- Footer group: `footer` (group(texts + `email-signup`), `menu` footer, `menu` main-menu; background `color1`) → `footer-utilities` (`footer-copyright`, `footer-policy-list`, `social-links`).
- Placement (`enabled_on` / `disabled_on`, never both):

| Restriction | Sections |
|---|---|
| `enabled_on.groups: [header]` | `header-announcements` |
| `enabled_on.groups: [footer]` | `footer`, `footer-utilities`, `logo` |
| `enabled_on.templates` | `main-collection` (`collection`), `quick-order-list` (`product`) |
| `disabled_on.groups: [header]` | `section`, `hero`, `featured-product`, `media-with-content`, `main-page`, `main-blog`, `password` |
| `disabled_on.groups: [header, footer]` | `carousel`, `collection-links`, `collection-list`, `featured-blog-posts`, `featured-product-information`, `layered-slideshow`, `main-cart`, `main-collection-list`, `marquee`, `product-hotspots`, `product-information`, `product-list`, `product-recommendations`, `section-rendering-product-card`, `slideshow` |
| none | `_blocks`, `custom-liquid`, `divider`, `header`, `cart-drawer-section`, `main-404`, `main-blog-post`, `password-footer`, `predictive-search*`, `search-header`, `search-results` |

- Sections with no preset (key absent or `[]`) cannot be added in the editor; they exist only through JSON: all `main-*`, `header`, `search-*`, `product-information`, `cart-drawer-section`, `predictive-search*`, `password*`, `section-rendering-product-card`.
- New `cx-` sections SHOULD set `disabled_on: { "groups": ["header"] }` unless built for the header.

## 5. Sections

### 5.1 The generic section: `section` (and `_blocks`)

`sections/section.liquid` ("Section") is the composition workhorse; build new bands from it before writing a section.

```
sections/section.liquid: capture children = content_for 'blocks' (:1-3); render 'section' (:5-10)
  snippets/section.liquid
    contrast-override (skip_contrast when background media)                       (:10-20)
    <div class="section-background color-custom-{id}">                            (:22)
    <div class="section section--{section_width} color-custom-{id}">              (:23-24)
      --section-min-height small|medium|large|full-screen|custom (svh)            (:31-43)
      background-media (image|video) + border-override + overlay                  (:45-61)
      <div class="spacing-style layout-panel-flex layout-panel-flex--{dir} [mobile-column] section-content-wrapper"
           style="layout-panel-style + spacing-style">  {{ children }}            (:63-77)
```

Settings (`sections/section.liquid` lines): `content_direction` column|row (37), `vertical_on_mobile` (53), `horizontal_alignment` (60), `vertical_alignment` (85), `align_baseline` (106), `horizontal_alignment_flex_direction_column` (113), `vertical_alignment_flex_direction_column` (134), `gap` 0–100 =12 (159), `section_width` page-width|full-width (173), `section_height` ""|small|medium|large|full-screen|custom (189) + `section_height_custom` 0–100 (schema unit `%`, rendered as svh by `snippets/section.liquid:33`) (221), `background_media` none|image|video (236), `background_color` (256), `video`/`video_position`, `background_image`/`background_image_position`, `toggle_overlay`/`overlay_color`/`overlay_style`/`gradient_direction`, `border`/`border_width`/`border_opacity`/`border_color`/`border_radius`, `padding-block-start`/`-end`. No inline padding.

13 presets (composition recipes):

| Preset | Tree |
|---|---|
| Custom section | empty, height small |
| Rich text | `text`(heading) · `text` · `button` |
| FAQ | `text`(h3) · `accordion` → 5 × `_accordion-row` → `text` |
| Video | `video` · `group`(row) → `text`, `button`(unstyled) |
| Pull quote | `text`(h2) · `button`(unstyled) |
| Contact form | `text` · `contact-form` → static `contact-form-submit-button` |
| Email signup | `text` · `text` · `email-signup` |
| Icons with text | 3 × `group` → `icon` + `group` → `text`(h3) + `text` |
| Split showcase | 2 × `group`(background image, height fill) → `spacer` + `text` + `button` |
| Image with text | `image` · `group` → `text`(h3) + `text` + `button` |
| Multicolumn | 3 × `group` → `text`(h4) + `text` |
| Image compare | `group` → (`group` → 2 × `text`), (`group` → 2 × `button`) · `comparison-slider` |
| Large logo | `text` · `logo` |

`_blocks`: same markup and schema minus `disabled_on`, one preset (Custom section without the `section_height` setting). It is the platform wrapper for AI-generated blocks and top-level app blocks (`sections/_blocks.liquid:1-5`; PLATFORM [AIB][AB]). Platform-reserved: do not repurpose; keep in sync with `section`.

### 5.2 Hand-rolled section skeleton

```liquid
{% liquid
  if section.settings.background_color != blank
    render 'contrast-override', background_color: section.settings.background_color, section_id: section.id
  endif
%}
<div class="section-background{% if section.settings.background_color != blank %} color-custom-{{ section.id }}{% endif %}"></div>
<div
  class="section section--{{ section.settings.section_width }} spacing-style gap-style {% if section.settings.background_color != blank %}color-custom-{{ section.id }}{% endif %}"
  style="{% render 'spacing-style', settings: section.settings %}{% render 'gap-style', value: section.settings.gap %}"
>
  {% content_for 'blocks' %}
</div>
```

Needs settings `section_width`, `gap`, `background_color`, `padding-block-start`, `padding-block-end` (copy from `sections/product-list.liquid`); drop `gap-style` without a `gap` setting. Precedents: `sections/carousel.liquid:1-24`, `collection-list.liquid:16-40`, `product-list.liquid:1-25`, `divider.liquid:7-20`.

Rules:
- `.section` is a 3-column grid (margin | centre | margin, `assets/base.css:307-340`); `.section--page-width > *` sits in the centre, `.section--full-width > *` spans all (`:360-367`); `.section--mobile-full-width`, `.section--full-width-margin`, `.force-full-width` (`:369-387`). Do not rewrite as `max-width` + `margin:auto` (`:303-306`).
- `.section` is transparent inside `.shopify-section` (`:343-345`); the background is the sibling `.section-background` (`:351-357`). Put `color-custom-{{ section.id }}` on **both** divs.
- Recurring setting IDs (reuse): `background_color` (35 sections), `padding-block-start/-end` (34), `section_width` (24), `gap` (16), the layout set, `section_height`, background media, overlay, border, resource-list set (`layout_type`, `columns`, `mobile_columns`, `columns_gap`, `rows_gap`, `carousel_on_mobile`, `mobile_card_size`), `icons_style`/`icons_shape`.
- Transparent header: the first section in `<main>` gets `--section-top-offset` added to its padding (`assets/base.css:1820-1838`); opt out with class `disable-section-top-offset` (`media-with-content`, `slideshow`).
- Schema `class` lands on the `shopify-section` wrapper; `tag` ∈ `article|aside|div|footer|header|section` (PLATFORM [SS]).
- New sections MUST start with a contract comment (`{% comment %} Section contract. <region, responsibility> @example <how it is reached> {% endcomment %}`); stock sections have none.

### 5.3 Section catalog

Editor name in parentheses when it differs from the file name. Preset names drop the `<Section name>: ` prefix where stock uses one (`Slideshow: Full frame`, `Collection list: Bento`, …); `hero` and `media-with-content` are shown as stock names them.

| Section | Purpose | Blocks | Presets |
|---|---|---|---|
| `section` (Section) | Generic band | `@theme`, `@app`, `_divider` | 13 (§5.1) |
| `_blocks` (Section) | Platform wrapper | same | 1 |
| `hero` | Full-bleed banner: 2 desktop + 2 mobile media slots, overlay, blurred reflection, whole-hero link | `@theme`, `text`, `button`, `logo`, `jumbo-text`, `spacer`, `group`, `_marquee` | Hero, Hero: Marquee, Hero: Bottom aligned |
| `slideshow` | `_slide` carousel, full-frame or inset | `_slide` | Full frame, Inset |
| `layered-slideshow` | Tabbed stacked panels (≤6) | `_layered-slide` | 1 |
| `carousel` | Header + horizontal `_card` carousel | static `group#static-header`, `_carousel-content#static-carousel-content` (→ `_card`) | 1 |
| `media-with-content` (Media with text) | Split media / content | static `_media-without-appearance#media`, `_content-without-appearance#content` | Editorial, Editorial: Jumbo text |
| `marquee` | Scrolling strip | `text`, `icon`, `logo`, `_divider` | 1 |
| `collection-links` | Collection titles; spotlight slideshow or text list | static `_collection-link#link` (loop) | Spotlight, Text |
| `collection-list` | Collection cards: grid, carousel, bento, editorial | header `@theme…`; static `_collection-card#static-collection-card` (loop) | Bento, Grid, Carousel, Editorial |
| `product-list` (Featured collection) | Products of one collection | static `_product-list-content#static-header`, `_product-card#static-product-card` (loop); trailing `@theme`, `@app`, `_divider` | Grid, Carousel, Editorial |
| `featured-blog-posts` (Blog posts) | Articles of one blog | `_featured-blog-posts-title`, `-card`, `-image` declared; static `_featured-blog-posts-title#static-blog-title`, `_featured-blog-posts-card#static-blog-post-card` (loop) | Carousel, Grid, Editorial |
| `featured-product` (Product highlight) | One product: media + card | static `_media-without-appearance#media`, `_featured-product#featured-product` | 1 |
| `featured-product-information` (Featured product) | Product-page layout for a chosen product | static `_featured-product-information-carousel#media-gallery`, `_product-details#product-details`; `@theme`, `@app` | 1 |
| `product-hotspots` | Image with hotspots | static `text#heading`; `_hotspot-product` | 1 |
| `product-recommendations` (Recommended products) | Related / complementary (lazy, hydration) | header `@theme…`; static `_product-card#static-product-card` | 1 |
| `logo` (footer only) | Large logo band | none | 1 |
| `divider` | Rule band | none | 1 |
| `custom-liquid` (Custom Liquid) | Merchant Liquid (L4) | none | 1 |
| `header-announcements` (Announcement bar; header only) | Rotating announcements | `_announcement` | 1 |
| `header` | Header (§5.4) | static `_header-logo`, `_header-menu` ×3 | none |
| `footer` (footer only) | Footer grid | explicit list (§3.6) | 1 |
| `footer-utilities` (Policies and links; footer only) | Copyright, policies, social | 3 types, `max_blocks: 3` | 1 |
| `main-page` (Page), `main-404` (404), `password` (Section) | Page bodies | `@theme`, `@app`, `_divider` | none |
| `main-blog` (Blog posts) | Blog listing | header area; static `_blog-post-card` (loop) | none |
| `main-blog-post` (Blog post) | Article + comments | `@theme`, `@app`, `_blog-post-*` | none |
| `main-collection` (Collection) | Collection grid + filters | static `filters`, `_product-card#product-card` | none |
| `main-collection-list` (Collection list) | All collections | header area; static `_collection-card` | none |
| `main-cart` (Cart) | Cart page | static `_cart-title`, `_cart-products`, `_cart-summary`; `@theme…` | none |
| `search-header` (Search), `search-results` | Search page | statics `_heading`, `_search-input`; `filters`, `_product-card` | none |
| `product-information` | Product page | static `_product-media-gallery`, `_product-details`; `@theme`, `@app` | none |
| `quick-order-list` | Variant table (product templates) | none | 1 |
| `password-footer` | Password page footer | none | none |
| `cart-drawer-section` (Cart drawer), `predictive-search` (Search popover), `predictive-search-empty`, `section-rendering-product-card` (Product card rendering) | Section Rendering API targets | none rendered | none |

Resource lists (`collection-list`, `main-collection-list`, `product-list`, `featured-blog-posts`) render through `snippets/resource-list.liquid` (`grid|carousel|bento|editorial`, `carousel_on_mobile`); `product-recommendations` (section and block) emit the same markup inline with `resource-list-carousel` (grid|carousel). Items are separated by `<!--@list/split-->`.

### 5.4 Header

- Static only: `_header-logo#header-logo`; `_header-menu#header-menu` rendered three times (desktop, `variant: 'mobile'` drawer, `variant: 'navigation_bar'`; `sections/header.liquid:81-95`).
- Rows `top`/`bottom`: `.section.section--full-width-margin` grids with left/center/right columns built by `snippets/header-row.liquid` from `<item>_position`/`<item>_row` settings; per-row colors via `contrast-override` with `section_id: <row>-<section.id>`.
- Menu vs drawer: `#header-component[data-menu-style="menu"|"drawer"]`, set pre-paint (`layout/theme.liquid:141-149`; touch device or `overflow-list[minimum-reached]` → drawer). Submenus are mega-menu panels (`snippets/mega-menu-list.liquid`; `menu_style` text|collection_images|featured_products|featured_collections).
- Sticky: `enable_sticky_header` always|scroll-up|never → `sticky` attribute; `data-sticky-state` from `assets/header.js`.
- Transparent: per-template toggles (home/product/collection) → `transparent` attribute, inverse logo; `--header-height`, `--header-group-height`, `--transparent-header-offset-boolean` measured by `layout/theme.liquid:45-135`. New first-paint consumers of those vars MUST `render 'measure-header-heights'`.
- Hydration: `hydrate(section.id)` re-renders the header after load; only `data-hydration-key` targets (`header-menu`, `header-drawer-mobile`, `header-menu-mobile-navigation-bar`) are morphed. Menu featured products are queried only on that second render (`section.index == blank`, `snippets/mega-menu-list.liquid:29-41`).
- 1,817 lines, 42 settings. Change it through settings first; header code edits are L5.

## 6. Blocks: catalog and authoring

### 6.1 Authoring conventions (template: Appendix A.1)

File order: `{% doc %}` → Liquid/markup → one `{% stylesheet %}` → `{% schema %}`.

1. `{% doc %}` on every new block: what it renders, `@param` for every `content_for` keyword arg, `@example`. (Stock: 20 of 95 blocks have one. Theme Check validates keyword args against it: `UnrecognizedContentForArguments`, `MissingContentForArguments`, `ValidContentForArgumentTypes`.) `block.settings` reads are not `@param`s.
2. First logic line: `assign block_settings = block.settings`.
3. Colors: with `background_color`/`text_color`, `render 'contrast-override'` with `section_id: block.id` before the root and add `color-custom-{{ block.id }}` only when one is set (`blocks/page.liquid:3-10`, `blocks/accordion.liquid:3-10`).
4. One root element, `"tag": null`, `{{ block.shopify_attributes }}` once.
5. Pair utility classes with their helper renders (§8.1): `spacing-style` ⇄ `render 'spacing-style'`, `size-style` ⇄ `size-style`, `border-style` ⇄ `border-override`, `layout-panel-flex layout-panel-flex--{dir} [mobile-column]` ⇄ `layout-panel-style`, `custom-typography custom-font-size custom-font-weight` ⇄ `typography-style`.
6. Instance values: inline custom properties namespaced `--cx-name-*`; `{% style %}` scoped by `.cx-name--{{ block.id }}` only for selectors, pseudo-classes or media queries (`blocks/icon.liquid:2-10`).
7. Containers: `{% content_for 'blocks' %}` once; `blocks: [@theme, @app, _divider]` unless restricted.
8. Read `closest.*`; read `section.settings` only in a block private to one section.
9. Escape plain merchant strings (`| escape`); output `richtext`/`inline_richtext` as-is.
10. DOM ids stable and unique per instance: append `block.id` (+ `section.id` if needed).
11. At least one preset with `category` from `t:categories.*` (`basic`, `layout`, `decorative`, `forms`, `links`, `footer`, `collection`, `product`, `blog_post`, `custom`). The picker lists presets alphabetically by name, grouped by category, uncategorized first (PLATFORM [TBS]).
12. All schema strings are `t:` keys (§14).

### 6.2 Standard setting groups (copy verbatim so shared helpers work)

| Group | IDs (type, default) | Consumer |
|---|---|---|
| Padding (`t:content.padding`) | `padding-block-start`, `padding-block-end`, `padding-inline-start`, `padding-inline-end` (range 0–100 px =0; labels `t:settings.top/bottom/left/right`) | `spacing-style` (scales >20px by 0.7 below 990px) or `spacing-padding` (unscaled) |
| Size (`t:content.size`) | `width` fit-content\|fill\|custom =fill; `custom_width` 0–100 % =100 ⟨`width == 'custom'`⟩; `width_mobile`, `custom_width_mobile`; `height` fit\|fill\|custom =fit; `custom_height` | `size-style` |
| Borders (`t:content.borders`) | `border` none\|solid =none; `border_width` 0–100 =1, `border_opacity` 0–100 % =100, `border_color` (placeholder) ⟨`border != 'none'`⟩; `border_radius` 0–100 =0 | `border-override` (opacity applies only when `border_color` blank) |
| Layout (`t:content.layout`) | `content_direction` column\|row =column; `vertical_on_mobile` =true ⟨row⟩; `horizontal_alignment` flex-start\|center\|flex-end\|space-between ⟨row⟩; `vertical_alignment` flex-start\|center\|flex-end =center ⟨row⟩; `align_baseline` ⟨`vertical_alignment == 'flex-end'`⟩; `horizontal_alignment_flex_direction_column` ⟨`!= 'row'`⟩; `vertical_alignment_flex_direction_column` (+space-between) ⟨`== 'column'`⟩; `gap` 0–100 =12 | `layout-panel-style` |
| Colors (`t:content.appearance`) | optional overrides `background_color`, `text_color`, `border_color` … type `color`, **no default**, `"placeholder": "t:settings.default"` (blank = inherit) | `contrast-override` |
| Always-resolving colors | `"default": "{{ settings.color_palette.foreground }}"` / `.background`, or a literal such as `#00000026` (overlays) | direct |
| Background media | `background_media` none\|image\|video; `background_color` ⟨none⟩; `video`, `video_position` cover\|contain; `background_image`, `background_image_position` cover\|fit; `toggle_overlay`, `overlay_color` =#00000026, `overlay_style` solid\|gradient, `gradient_direction` to top\|to bottom | `background-media`, `overlay` |
| Typography (text family) | `type_preset` rte\|paragraph\|h1…h6\|custom =rte (`info: t:info.edit_presets_in_theme_settings`); when custom: `font` (values `var(--font-body--family)` …), `font_size` (rem strings), `line_height`/`letter_spacing` tight\|normal\|loose, `case` none\|uppercase, `wrap` pretty\|balance\|nowrap | class `{{ type_preset }}` + `typography-style` |
| Lightweight preset | select `""`\|paragraph\|h1…h6 emitted as a class (`heading_preset`, `link_preset`, `text_preset`, and `type_preset` in `accordion`/`popup-link`) | `.paragraph`, `.h1`–`.h6` |
| Link | `link` (url), `open_in_new_tab` =false | markup |
| Hidden per-instance flag | `"visible_if": "{{ false }}"`, set only by presets/templates: `group.placeholder`, `_heading.read_only`/`show_alignment`, `_blog-post-info-text.show_alignment`, `swatches.hide_padding` | markup |

Kebab-case ids only for `padding-*` and `x-position`/`y-position`; everything else snake_case. Stock has inconsistent twins (`alignment` vs `horizontal_alignment`; `image_ratio` vs `aspect_ratio` vs `ratio`; `font` as CSS var vs bare key in `jumbo-text`/`logo`): copy the group from the closest stock block of the same family.

`visible_if`: `"{{ block.settings.x == 'y' }}"` / `"{{ section.settings.x … }}"`. Stock also reads `section.settings` and global `settings` from a block's `visible_if` (`blocks/_product-media-gallery.liquid:258`, `blocks/_featured-product-information-carousel.liquid:77`, `blocks/logo.liquid:111`); undocumented but working. Supported on basic inputs, sidebar settings and `color`, `color_background`, `color_scheme`, `font_picker`, `html`, `image_picker`, `inline_richtext`, `link_list`, `liquid`, `richtext`, `text_alignment`, `url`, `video`, `video_url` (PLATFORM [SET]); not on resource pickers, `*_list` or `color_palette`. Conditions cannot read what a dynamic source resolves to.

### 6.3 Public block catalog (44)

Every public block with a preset is addable in every `@theme` container (§3.6). "—" = no preset.

| Block | Category | Purpose | Children | Notable settings / notes |
|---|---|---|---|---|
| `group` | layout | Flex container: layout, size, background media, overlay, border, full-card link, padding. **Default building unit.** | `@theme`, `@app`, `_divider` | 36 settings; renders `snippets/group.liquid` |
| `text` | basic (Text, Heading) | Rich text / headings; level from the markup, style from `type_preset` | — | `text` (richtext), width fit-content\|100%, `max_width`, alignment, typography, `text_color`, pill (`background`, `background_color`, `corner_radius`), unscaled padding. Supports dynamic sources. |
| `button` | basic | Link styled as button | — | `label`, `link`, `style_class` button\|button-secondary\|button-unstyled\|button-custom (+ custom colors), size |
| `image` | basic | Single image | — | `image`, `link`, `image_ratio` adapt\|portrait\|square\|landscape, size, border, padding |
| `icon` | basic | Built-in icon or upload | — | `icon`, `image_upload`, `width`, `link`, `icon_color` |
| `video` | basic | Upload or YouTube/Vimeo | — | `source`, `video`, `video_url`, autoplay, loop, `cover_image`, `alt`, `aspect_ratio`, border |
| `spacer` | layout | Flexible/fixed gap | — | `size` percent\|pixel, mobile override |
| `accordion` | layout | Collapsible rows | `_accordion-row` | icon caret\|plus, dividers, lightweight `type_preset` (=h6), colors, border |
| `custom-liquid` | custom | Merchant Liquid (L4) | — | `custom_liquid` (type `liquid`). Omits `block.shopify_attributes`. |
| `email-signup` | forms | Newsletter (`{% form 'customer' %}`) | — | display type, integrated button, input + button styles |
| `contact-form` | forms | `{% form 'contact' %}` | static `contact-form-submit-button` | size, input styles; DOM ids not unique per instance |
| `contact-form-submit-button` | — | Submit button (static only) | — | label, style (3 options, no unstyled) |
| `jumbo-text` | decorative | Text auto-fit to width | — | `text`, `font` (bare key), `text_effect`, `animation_repeat` |
| `logo` | basic | Shop logo (`settings.logo` / `logo_inverse`), shop-name fallback | — | unit percent\|pixel, sizes, `inverse` |
| `menu` | links | Link list with heading; mobile accordion | — | `menu`, `heading`, heading/link presets, accordion |
| `social-links` | footer | Social icon row | — | `icon_color`; 12 platform URL settings + `custom_url` |
| `payment-icons` | footer | Enabled payment icons | — | alignment, gap |
| `follow-on-shop` | footer | Shop "Follow" button (`shop.features.follow_on_shop?`) | — | padding |
| `footer-copyright` | footer | © year + shop link | — | `show_powered_by`, font size, case |
| `footer-policy-list` | footer | Policies popover | — | font size, case |
| `popup-link` | links | Text link opening a modal/drawer | `@theme`, `@app` | `behavior`, `heading`, lightweight `type_preset` |
| `comparison-slider` | decorative | Before/after slider | — | images, labels, slider colors, ratio, size, border |
| `collection-card` | collection | Card for a picked collection | `text`, `spacer`, `button`, `group`, `collection-title`; static `_collection-card-image` | `collection`, placement, alignment, gap, border |
| `collection-title` | collection | `closest.collection` title | — | text-family settings |
| `featured-collection` | — | Container providing `closest.collection` | `@theme`, `@app` | `collection`. `presets: []`, no stock reference. |
| `page` | basic | Picked page title + content | — | `page`, `max_width` (no CSS consumer), colors |
| `page-content` | — | Current `page.content` | — | `text_color`. Placed via JSON in `templates/page.json`. |
| `product-title` | product | `closest.product` title (link) | — | text-family settings |
| `product-description` | product | `text` preset bound to `{{ closest.product.description }}` | — | identical to `text` |
| `price` | product | Price, sale, installments, tax note | — | `show_sale_price_first`, `show_installments`, `show_tax_info`, typography |
| `variant-picker` | product | Variant options | — | `variant_style` dropdowns\|buttons, `show_swatches`, custom colors |
| `buy-buttons` | product | Product form: quantity, add to cart, accelerated checkout, pickup, gift-card recipient | static `quantity`, `add-to-cart`, `accelerated-checkout` | `stacking`, `show_pickup_availability`, `gift_card_form`. One per section (§13.1). |
| `add-to-cart` | — | Add button (static in `buy-buttons`) | — | `style_class` (3 options) + custom colors |
| `accelerated-checkout` | — | Dynamic checkout buttons (static in `buy-buttons`) | — | no settings; no markup standalone (still renders `buy-buttons-styles`) |
| `quantity` | product | Quantity selector | — | input styles. Not wired to a form standalone. |
| `product-inventory` | product | Stock level message | — | `inventory_threshold`, `show_inventory_quantity` |
| `sku` | product | Variant SKU | — | typography |
| `review` | product | Stars from `reviews.rating` metafields | — | `stars_style`, `show_number` |
| `swatches` | product | First swatch option of `closest.product` (cards) | — | alignment, padding |
| `product-custom-property` | product | Buyer input → `properties[key]` | — | key, text\|checkbox, `required`, max length. Needs `buy-buttons` in the section. |
| `product-card` | product | Card for a picked product | card list (§13.3) minus `buy-buttons` | `product`, gap, width, border |
| `product-recommendations` | product | Recommendations inside a container (lazy) | `@theme`, `@app`; static `_product-card` | `recommendation_type` related\|complementary, layout |
| `disclosures` | product | `shopify.disclosure` metafield accordion | — | heading, presets, dividers |
| `filters` | — | Filtering, sorting, density (static in collection/search) | — | `enable_filtering`, `filter_style` horizontal\|vertical, sorting, density |

### 6.4 Private block catalog (51)

| Block | Owner(s) | Role |
|---|---|---|
| `_divider` | `group`, `section`, `_blocks`, `main-page`, `main-404`, `main-blog`, `main-collection-list`, `password`, `product-list`, `collection-list`, `product-recommendations` (section), `footer`, `marquee`, `_marquee`, `_content-without-appearance`, `_product-details`, `_product-list-content` (+ orphan `_content`) | Rule |
| `_accordion-row` | `accordion` | `<details>` row; accepts `@theme`, `@app` |
| `_card` | `_carousel-content` | Carousel card |
| `_carousel-content` | `carousel` (static) | Track of `_card` |
| `_slide` / `_layered-slide` | `slideshow` / `layered-slideshow` | Slide / layered panel |
| `_marquee` | `hero` | Inline marquee |
| `_heading` | `_card`, `_slide`, `_layered-slide`; static in blog cards and `search-header` | Heading with `text` keyword arg |
| `_media-without-appearance` | `media-with-content`, `featured-product` (static) | Media half |
| `_content-without-appearance` | `media-with-content` (static) | Content group |
| `_announcement` | `header-announcements` | Announcement |
| `_header-logo`, `_header-menu` | `header` (static) | Logo; menu (3 variants) |
| `_search-input` | `search-header` (static) | Search field |
| `_collection-link` | `collection-links` (static, loop) | Collection link; renders static `_image`, `_inline-collection-title` |
| `_collection-card` | static in `collection-list`, `main-collection-list` | Collection card; renders static `_collection-card-image` (also static in public `collection-card`) |
| `_product-card` | static in `product-list`, `main-collection`, `search-results`, `product-recommendations` (section + block) | Product card (§13.3); children `_product-card-gallery`, `_product-card-group` (nests itself) |
| `_product-list-content` | `product-list` (static header) | Header; children `_product-list-text`, `_product-list-button` |
| `_product-details`, `_product-media-gallery` | `product-information` (static); `_product-details` also `featured-product-information` | Product page columns |
| `_featured-product` | `featured-product` (static) | Highlight card; renders static `_featured-product-gallery`, `_featured-product-price` |
| `_featured-product-information-carousel` | `featured-product-information` (static) | Media carousel |
| `_hotspot-product` | `product-hotspots` | Hotspot (own `product`, `x-position`, `y-position`) |
| `_featured-blog-posts-title`, `_featured-blog-posts-card` | `featured-blog-posts` (static) | Blog band; card renders static `_featured-blog-posts-image`, `_blog-post-info-text`, `_blog-post-description` |
| `_blog-post-card` | `main-blog` (static) | Listing card; renders static `_heading`, `_blog-post-info-text`, `_blog-post-image`, `_blog-post-description` |
| `_blog-post-title`, `_blog-post-info-text`, `_blog-post-featured-image`, `_blog-post-content` | `main-blog-post` | Article parts |
| `_cart-title`, `_cart-products`, `_cart-summary` | `main-cart` (static) | Cart page parts |
| **Orphans** `_collection-image`, `_collection-info`, `_content`, `_footer-social-icons`, `_inline-text`, `_social-link` (used only by the orphan `_footer-social-icons`) | none | Unreachable. Prefer `group`, `social-links`, `_inline-collection-title`. |

Reusing a stock private block statically in a `cx-` section is allowed and often right (e.g. `_product-card` in a product loop gives merchants the full card editor). Check its coupling first (§6.5) and provide the section settings it reads.

### 6.5 Parent coupling (reads of `section.settings`)

| Block / snippet | Reads | Owner |
|---|---|---|
| `_carousel-content` | `icons_style`, `section_width`, `columns`, `mobile_columns`, `columns_gap`, `icons_shape` | `carousel` |
| `_collection-link` | `image_height` (undefined in the section → nil), `layout` | `collection-links` |
| `_featured-blog-posts-card` / `-title` | `layout_type` / `blog` | `featured-blog-posts` |
| `_header-logo`, `_header-menu` | transparent flags, inverse logo, `menu_row`, `background_color_{row}`, `section_width` | `header` |
| `_hotspot-product` | `hotspot_color`, `bullseye_color`, `product_title_preset`, `product_price_preset` | `product-hotspots` |
| `_layered-slide` | `drop_shadow` | `layered-slideshow` |
| `_slide` | `background_color`, `content_position`, `slide_height` | `slideshow` |
| `_product-list-button` | `max_products` | `product-list` |
| `snippets/card-gallery` | `product_card_size`, `mobile_product_card_size`, `columns_gap_horizontal`, `columns`, `layout_type`, `product_grid_width` (image `sizes` only; falls back to `(min-width: 750px) 50vw, 100vw`) | any card host |
| `snippets/collection-card` | `layout_type` (bento/editorial classes) | any host |
| `snippets/background-media` | `section_width` (image `sizes`), via its `section` param | any |
| `_product-media-gallery`, `_featured-product-information-carousel` | `content_width` (also in `visible_if`), `equal_columns`, `gap` (via `util-product-media-sizes-attr`) | product sections |
| `snippets/header-drawer`, `header-actions` | localization and actions settings | `header` |

Other context reads: `section.id` for DOM/form ids and Quick Add / SRA targets (`product-custom-property`, `buy-buttons`, `product-inventory`, `contact-form`, `filters`, `product-recommendations`, `_product-details`, `_slide`, `_layered-slide`, `_blog-post-card`, `_cart-summary`, `_hotspot-product`, `_product-card-gallery`, `_featured-product-gallery`); `section.index` (`background-media`, `card-gallery` loading priority, `_slide`, SRA detection in `mega-menu-list`/`header-drawer`); `section.blocks | find_index` (`_slide`, `_layered-slide`, `_announcement`).

### 6.6 Schema design rules

- Settings are a public API: add one only for a choice a merchant should control; keep structural details in code so every combination of controls is valid. Shop-wide choices → theme settings; content/presentation of one context → section/block settings.
- Use resource pickers (`product`, `collection`, `page`, `blog`, `article`, `image_picker`, `video`, `link_list`, `url`, `metaobject`, `*_list`), never free-text handles. Prefer focused controls, useful defaults and `visible_if` over many always-visible options; keep each `visible_if` aligned with the IDs and values it references.
- Use only attributes the setting type supports (`range` needs numeric `min`, `max`, `default`; `richtext` default starts with `<p>`/`<ul>`; `url` default only `/collections` or `/collections/all`; resource pickers take no `default`; `link_list` default `main-menu` or `footer` per docs, stock also uses `customer-account-main-menu` at `sections/header.liquid:1325`) and the schema kind (block: `name`, `settings`, `blocks`, `presets`, `tag`, `class`; section adds `limit`, `max_blocks`, `enabled_on`/`disabled_on`, `default`, `locales`). Omit `settings` when there are none.
- Presets are useful starting compositions. Keep preset keys, `block_order`, `"static": true` markers and nested settings aligned with what the Liquid renders; re-check presets and templates when a setting, accepted type or composition changes (Theme Check `ValidSettingsKey`, `SchemaPresetsBlockOrder`, `SchemaPresetsStaticBlocks`, `ValidBlockTarget`).
- Static for required structure (fixed place and identity); dynamic where merchants control presence and order. Static blocks may still have settings and children.

### 6.7 Liquid style

Liquid renders the initial HTML; JS adds behavior only. `{% liquid %}` for multi-line logic (`#` comments inside); `{%- -%}` where whitespace matters. Dot notation; bracket access only for computed keys (`settings[key]`, `snippets/spacing-style.liquid`). Only objects, tags and filters that exist in source or platform docs (`validate_theme` rejects unknown ones). Render optional content only when it has a value (no empty wrappers or headings). Escape plain merchant strings; never build HTML from unescaped text. Comments only for non-obvious intent or coupling.

## 7. Layout and page shell

### 7.1 `layout/theme.liquid`

`<html class="[shopify-design-mode]" lang="{{ request.locale.iso_code }}" dir="{{ request.locale.direction | default: 'ltr' }}">` (`:2-8`).

| Line | Render | Gate | Emits |
|---|---|---|---|
| 10-16 | favicon link | `settings.favicon != blank` | |
| 18 | `view-transition-opt-in` | `page_transition_enabled` or `transition_to_main_product` | `@view-transition` + render blocker on `#MainContent` |
| 20 | `meta-tags` | | charset, viewport (`viewport-fit=cover`), OG/Twitter, title, canonical, description |
| 21 | `stylesheets` | | preload `overflow-list.css`; `base.css` (`stylesheet_tag: preload: true`) |
| 22 | `card-hover-effect-styles` | `{% style %}` only if `card_hover_effect != 'none'` | card hover CSS (global so search-modal cards get it) |
| 23 | `fonts` | non-system fonts | font preloads |
| 24 | `scripts` | | import map, module preloads, global modules, `Theme` global (§12.1) |
| 25 | `theme-styles-variables` | | `@font-face` + `:root` tokens + 4 UI color scopes |
| 26 | `color-palette` | | `:root` color tokens + `body` colors |
| 28-30 | `theme-editor` | `request.design_mode` | `assets/theme-editor.js` |
| 32 | `{{ content_for_header }}` | | last in `<head>`; never wrap or modify |

Body: `<body class="page-width-{{ settings.page_width }} card-hover-effect-{{ settings.card_hover_effect }}">` → `skip-to-content-link` → `chat-drawer` → `.page-wrapper` [`#header-group` → inline header-measure script (`:45-150`) → `<main id="MainContent" class="content-for-layout" data-template data-page-transition-enabled data-product-transition>` → `<footer>` group] → `cart-drawer` (always rendered; the snippet outputs nothing unless `cart_type == 'drawer'` and `template.name != 'cart'`, `snippets/cart-drawer.liquid:11`) → `theme-drawer` → `search-modal` → `quick-add-modal` (`quick_add or mobile_quick_add`).

Shell facts for custom code:
- At ≥990px the scroll container is `.page-wrapper`, not `window` (`assets/base.css:28-56`); use `@theme/scroll-container` (§12.6).
- `body.page-width-{narrow|normal|wide}` sets `--page-content-width` (90/120/150rem, `snippets/theme-styles-variables.liquid:138-140`), `--page-width` (content + 2 × `--page-margin`) and `--page-margin` (`max(16px, safe-area insets)`; 40px ≥750px) (`assets/base.css:265-296`).
- `<main>` has no `tabindex="-1"`.
- `layout/password.liquid`: `stylesheets` first, no card-hover, no `.page-wrapper`, no groups/drawers/search/quick-add; static `{% section 'password-footer' %}`; `{% form 'storefront_password' %}` dialog; `password-layout-styles` (layouts cannot hold `{% stylesheet %}`).
- `templates/gift_card.liquid`: `{% layout none %}`, own head and three-entry import map (`@theme/component`, `@theme/utilities`, `@theme/qr-code-generator`; `:13-19`); loads `base.css`, `template-giftcard.css`, tokens and palette; not `scripts`.

### 7.2 Editing layouts

L5. Legitimate: one line rendering a global `cx-` snippet (§9.9 Route B). Link a global stylesheet from `snippets/stylesheets.liquid` instead (§9.9 Route A), which also covers the password and gift-card shells. Keep `content_for_header` last in `<head>` and `content_for_layout` inside `<main id="MainContent">`. Mirror global additions in `layout/password.liquid` when the password page needs them.

## 8. Snippets

`{% render 'name', param: value %}`, isolated scope: pass every caller-local value. Globals (`settings`, `shop`, `cart`, `request`, `routes`, `closest.*`, …) are available. Some stock snippets read `block`/`section` from context even though no caller passes them (`snippets/button.liquid:11`, `snippets/card-gallery.liquid:13`); new snippets SHOULD take what they use as params. 134 of 145 stock snippets start with `{% doc %}`; new ones MUST (`@param {type} name - desc`, `[name]` optional, `@example`).

### 8.1 Style helpers (inline-style emitters paired with base.css classes)

| Snippet | Call | Reads | Emits | Pair class |
|---|---|---|---|---|
| `spacing-style` | `style="{% render 'spacing-style', settings: block.settings %}"` | `padding-block-start/-end`, `padding-inline-start/-end` | `--padding-*`; >20 → `max(20px, calc(var(--spacing-scale) * Npx))` (`--spacing-scale` 0.7 <990px, 1 ≥990px) | `.spacing-style` |
| `spacing-padding` | same | same | all four `--padding-*` in px, unscaled | `.spacing-style` |
| `size-style` | `settings:` | `width`, `custom_width`, `height`, `custom_height`, `width_mobile`, `custom_width_mobile` | `--size-style-width(-mobile)`, `--size-style-width-mobile-min`, `--size-style-height` | `.size-style` |
| `gap-style` | `value:`, `[name]`, `[scale_min]` (=24; doc says 20), `[disable_scaling]` | — | `--{name}`; >24 → `max(24px, calc(var(--gap-scale, 1) * Npx))` (`--gap-scale` 0.7 <990px, set by the pair classes) | `.gap-style`, `.layout-panel-flex` |
| `layout-panel-style` | `settings:` | layout set + `gap` | `--flex-direction`, `--flex-wrap(-mobile)`, `--gap`, `--horizontal-alignment`, `--vertical-alignment(-mobile)` | `.layout-panel-flex`, `--row`, `--column`, `.mobile-column` |
| `border-override` | `settings:` | `border`, `border_width`, `border_opacity`, `border_color`, `border_radius` | `--border-*` + `overflow: clip` when radius >0 | `.border-style` |
| `typography-style` | `settings:`, `[type]`, `[preset]` | custom typography set | only when preset is `custom`: `--font-size` (fluid `clamp()` at ≥3rem), `--font-weight` (falls back to the font's weight token), `--font-family`, `--line-height` (or `--line-height--{display\|heading\|body}` without a custom size), `--letter-spacing`, `--text-transform`, `--text-wrap` | `.custom-typography`, `.custom-font-size`, `.custom-font-weight` |
| `contrast-override` | `background_color`, `text_color`, `border_color`, `section_id` (required), `[skip_contrast]`, `[preset]` content\|ui | — | `{% style %}` scoped to `.color-custom-<section_id>` (§10.2) | `color-custom-<id>` |
| `background-media` | `background_media`, `background_video(_position)`, `background_image(_position)`, `[placeholder]`, `[block]`, `[section]` | `section.index` (priority), `section.settings.section_width` (sizes) | media markup + own CSS | — |
| `overlay` | `settings:`, `[layer]` | `overlay_color`, `overlay_style`, `gradient_direction`, `border_radius` | overlay div | — |
| `input-custom-styles` | `block_settings:` | `input_background_color`, `input_text_color`, `input_border_color` (+ global `palette_input_background`, `page_text_color`) | `--color-input-background`; `--color-input-text(-rgb)` always (explicit → `page_text_color` if contrast ≥4.5 → palette extreme); `--color-input-border` only when set | — |
| `button-custom-styles` | `block_settings`, `block_id` | `style_class`, `custom_button_*`, `link_text_color` | `{% style %}` `.button-custom--<id>` (+ hover); `.button-unstyled--<id>` link color when `style_class == 'button-unstyled'` | `.button-custom` |

### 8.2 Component renderers (reuse, do not re-implement)

| Snippet | Renders | Key params |
|---|---|---|
| `section` | generic section wrapper (§5.1) | `section`, `children`, `section_id`, `background_color` |
| `group` | group-like container | `children`, `settings`, `shopify_attributes`, `[class]`, `[style]`, `[background_color]`, `[block_id]` |
| `text` | every text-like block (renders `text-block-styles`; `<rte-formatter>` for rte/paragraph) | `[block]`, `[class]`, `[fallback_text]`, `[width]` |
| `button` | `<a>` button from `block.settings` | `link`, `[block]` |
| `image` | height-based `<img>` with 1x–3x srcset | `image`, `[height]`, `[class]`, … |
| `icon` / `icon-or-image` | SVG paths (70 names; caller supplies the `<svg>`, usually `viewBox="0 0 20 20"`; `caret`, `double-sided-caret`, `3d-model` differ) / svg-or-upload | `icon` / `icon`, `image_upload`, `width`, `class_name` |
| `divider` | horizontal/vertical rule | `id`, `settings`, `[full_width]`, `[attributes]` |
| `video` | `<video>` or deferred YouTube/Vimeo iframe | 15 params |
| `media` | media block contents (`_media-without-appearance`; doc also names a non-existent `_media`) | |
| `slideshow`, `slideshow-slide`, `slideshow-controls`, `slideshow-arrows` | slideshow component | many |
| `resource-list`, `resource-list-carousel`, `bento-grid`, `editorial-*-grid` | list layouts | `resource-list`: `settings`, `list_items(_array)`, `slide_count`, `content_type`; `resource-list-carousel`: `ref`, `slides`, `slide_count`, `settings`; bento/editorial: `items` |
| `resource-card`, `resource-image` | card for product/collection/article/page (predictive search, mega menu) | |
| `product-card`, `card-gallery`, `price`, `swatch`, `variant-main-picker`, `variant-swatches`, `quantity-selector`, `add-to-cart-button`, `quick-add`, `product-media*`, `product-grid` | commerce (§13) | |
| `accordion-custom-component` | `<accordion-custom>` wrapper | open/disable per breakpoint |
| `theme-drawer`, `theme-drawer-header` | drawer infrastructure | |
| `localization-form`, `search`, `search-modal`, `header-*`, `mega-menu-list` | header parts | |
| `format-price`, `unit-price`, `tax-info`, `volume-pricing-info` | price helpers | |
| `util-*` | computed attributes (`sizes`, card size, hover color) | |
| `measure-header-heights` | pre-paint header measurement | `only_when_first_section`, `only_on_desktop` |

### 8.3 Style owners (`*-styles.liquid`)

42 snippets; 36 hold one `{% stylesheet %}` that is collected only when the snippet renders. **Any file that emits a component's markup MUST render that component's style owner** (or a snippet that does), including fragment, hydration and Quick Add paths. Consumer list: §9.6.

## 9. CSS architecture

### 9.1 Layers

| Layer | Where | Delivery |
|---|---|---|
| Foundations, resets, section grid, utilities, type presets, buttons, fields, keyframes | `assets/base.css` (1,838 lines) | `stylesheet_tag` in `snippets/stylesheets.liquid:2`; never subsetted |
| Design tokens + `@font-face` | `snippets/theme-styles-variables.liquid` (`{% style %}`) | `<head>` after base.css |
| Color tokens | `snippets/color-palette.liquid` (`{% style %}`) | `<head>` after tokens |
| Component CSS | one `{% stylesheet %}` per file (154 files: 20 sections, 51 blocks, 83 snippets) | platform bundle via `content_for_header`; **subsetting** ships only files rendered on the page (PLATFORM [SUB]) |
| Shared component CSS | 36 `snippets/*-styles.liquid` owners | rendered by every consumer |
| Instance values | inline `style="--x: …"` + `{% style %}` (26 tags in 24 files: 22 scoped by `section.id`/`block.id`; 4 global head styles) | per render |
| Shadow DOM | `assets/overflow-list.css` | `<template shadowrootmode>` |
| Gift card | `assets/template-giftcard.css` | `templates/gift_card.liquid:10` |

No `@layer`, no `@scope`. No stock `{% stylesheet %}` contains Liquid or a second tag.

### 9.2 Rules

- Inspect the owning source first: settings-backed tokens, shared classes, helpers, composition pattern. Style through the tokens and classes the settings drive; never hard-code a value a setting controls.
- One static `{% stylesheet %}` per file; no Liquid in it. Setting-driven values → inline custom properties (preferred) or instance-scoped `{% style %}` (`blocks/icon.liquid:2-10`); `{% style %}` color values live-update in the editor.
- Subsetting: CSS exists only if its file is in the render tree. Define classes in the file that uses them, a rendered parent or a style owner rendered by every consumer (Theme Check `ValidScopedCSSClass`). For `cx_style` looks, the style owner is a `cx-` snippet the stock file renders (§9.10). Verify delivery on full pages and in Section Rendering responses.
- Selectors: BEM-ish, single-class, shallow `&` nesting; resolve conflicts by scope and source order; root class = file name; modifiers from setting values (`cx-name--{{ value }}`); state via attributes. Stock uses `:where()` (54×) to stay overridable; doubled classes (`.h1.h1`, `.text-block.text-block`) only to beat known stock rules. Match adjacent stock property ordering.
- Tokens only (§9.3); namespace component vars `--cx-name-*`.
- Logical properties only (`margin-inline-start`, `inset-inline`, `padding-block`, `text-align: start|end`); 4.2.0 maps `left`/`right` setting values to `start`/`end`. Physical exceptions need `/* stylelint-disable-line csstools/use-logical -- reason */`. Mirror directional icons under `[dir='rtl']`; verify LTR and RTL.
- `min-width: 0` on flex/grid children; constrain media; wrap merchant text.
- Motion: `transform`/`opacity`, motion tokens, decorative animation inside `@media (prefers-reduced-motion: no-preference)` (no global reset).
- Breakpoints: `750px` (`min-width: 750px` / `max-width: 749px`), `990px` (spacing/gap scale, scroll container, drawers modal↔inline). JS: `mediaQueryLarge` 750, `MODAL_BREAKPOINT` 990. Container queries where the component's own width matters (`snippets/resource-list-styles.liquid:20-79`).

### 9.3 Tokens (values in `snippets/theme-styles-variables.liquid` unless noted)

| Group | Tokens |
|---|---|
| Fonts (settings) | `--font-{body\|subheading\|heading\|accent}--family/--style/--weight` |
| Type presets (settings) | `--font-{paragraph\|h1…h6}--family/--style/--weight/--size/--line-height/--case`, `--font-h{N}--letter-spacing`, `--font-{preset}--spacing` |
| Fixed sizes | `--font-size--3xs` .625rem … `--font-size--6xl` 3.5rem (12 steps) |
| Line height / tracking | `--line-height--{display\|heading\|body}-{tight\|normal\|loose}`, `--letter-spacing--{display\|heading\|body}-{tight\|normal\|loose}` |
| Measure | `--max-width--{body\|heading}-{normal\|narrow}`, `--max-width--display-{normal\|narrow\|tight}` |
| Spacing | `--padding-{3xs…6xl}`, `--margin-{3xs,2xs,xs,sm,md,lg,xl,2xl,4xl,6xl}` (no `3xl`/`5xl`), `--gap-{3xs…3xl}` (rem), `--spacing-scale-md` .7, `--spacing-scale-default` 1 |
| Page | `--page-width`, `--page-margin`, `--page-content-width` (`assets/base.css:265-301`), `--narrow/normal/wide-page-width`, `--narrow/normal-content-width`, `--sidebar-width`, `--theme-drawer-width` |
| Section heights | `--section-height-{small\|medium\|large}` |
| Radii / borders (settings) | `--style-border-radius-{xs\|sm\|md\|lg\|50}`, `--style-border-radius-{pills\|inputs\|buttons-primary\|buttons-secondary\|popover}`, `--style-border-width(-primary\|-secondary\|-inputs)`, `--style-border-popover`, `--style-border-drawer` |
| Layers | `--layer-section-background` -2, `--layer-lowest` -1, `--layer-base` 0, `--layer-flat` 1, `--layer-raised` 2, `--layer-heightened` 4, `--layer-sticky` 8, `--layer-window-overlay` 10, `--layer-header-menu` 12, `--layer-overlay` 16, `--layer-menu-drawer` 18, `--layer-temporary` 20 |
| Opacity | `--opacity-{5,8,10,15,20,25,30,40,50,60,70,80,85,90}`, `--opacity-subdued-text` .8, `--opacity-muted-text` .6; brightness-aware `--opacity-5-15`, `-10-25`, `-35-55`, `-40-60`, `-30-60` |
| Motion | `--animation-speed-fast` .0625s, `--animation-speed` .125s, `--animation-speed-medium` .15s, `--animation-speed-slow` .2s, `--drawer-animation-speed`, `--animation-easing`, `--ease-out-cubic`, `--ease-out-quad`, `--animation-timing-*`, `--spring-d{300,280,220,180}-b0-easing/-duration`; base.css: `--hover-transition-duration` .25s, `--hover-transition-timing`, `--submenu-animation-speed` |
| Icons / focus / touch | `--icon-size-{2xs\|xs\|sm\|md\|lg}`, `--icon-stroke-width`, `--focus-outline-width`, `--focus-outline-offset`, `--minimum-touch-target` 44px |
| Inputs / buttons | `--input-padding*`, `--input-box-shadow*`, `--checkbox-*`, `--button-padding-block` 16px, `--button-padding-inline` 24px, `--button-font-family-primary/secondary`, `--button-text-case-primary/secondary` |
| Status colors | `--color-error`, `--color-success`, `--color-instock`, `--color-lowstock`, `--color-outofstock`, `--color-white(-rgb)` |

### 9.4 Global classes in `assets/base.css`

| Group | Classes |
|---|---|
| Section grid | `.section`, `.section-background`, `.section--page-width`, `.section--full-width`, `.section--mobile-full-width`, `.section--full-width-margin`, `.force-full-width`, `.section--height-*`, `.section-content-wrapper`, `.page-width-content`, `.disable-section-top-offset` |
| 14-col grid | `.grid` (flex <750px; 14 columns ≥750px; `grid-column: var(--centered)` / `var(--full-width)`) |
| Composition | `.spacing-style`, `.size-style`, `.border-style`, `.gap-style`, `.layout-panel-flex(--row\|--column)`, `.mobile-column`, `.group-block*`, `.custom-typography`, `.custom-font-size`, `.custom-font-weight` |
| Typography | `.paragraph`, `.h1`…`.h6` (bare `h1`…`h6` also styled), `.rte` |
| Buttons / links | `.button`, `.button-secondary`, `.button-custom`, `.button-unstyled(--with-icon\|--transparent)`, link underline animation |
| Forms | `.field`, `.field__input`, `.field__label`, radio/checkbox base |
| Utilities | `.hidden`, `.hidden--mobile`/`.mobile:hidden`, `.hidden--desktop`/`.desktop:hidden`, `.visually-hidden`, `.hide-when-empty`, `.list-unstyled`, `.text-left/center/right` (logical), `.justify-*`, `.svg-wrapper`, `.icon-*`, `.flex`, `.contents`, `.relative`, `.flip-x`, `.wrap-text`, `.sticky-content(--desktop)` |
| Behaviors | `*:focus-visible` outline; `.shopify-block:has(> [shopify-block-empty]) {display:none}`; view-transition rules (reduced-motion gated); `.page-wrapper` scroll container ≥990px; `@keyframes` (`:1472-1682`) |

### 9.5 Typography

Seven presets: `paragraph`, `h1`–`h6`. Sizes from `type_size_*`: `<size/16>rem` below 48px, `clamp(min, size×0.1vw, max)` at ≥48px. Apply by class (`class="{{ block_settings.type_preset }}"`; `rte` = rich-text defaults) or tokens (`font-size: var(--font-h3--size)`); custom typography via `typography-style` + classes. Fonts: `snippets/fonts.liquid` preloads; `@font-face` (regular/bold/italic/bold-italic, `font-display: swap`) in `theme-styles-variables`. Never load fonts elsewhere; use the four font roles.

### 9.6 Style owners and consumers

| Owner | Consumers |
|---|---|
| `product-card-styles` | `snippets/product-card`. Grid CSS belongs to `snippets/product-grid`, whose empty branch (`:105-109`) pre-renders `product-card-styles`, `text-block-styles`, `price-styles`, `product-card-gallery-zoom-details-styles` so filtered-in results arrive styled |
| `add-to-cart-button-styles` | `blocks/add-to-cart`, `snippets/quick-add`, `sections/product-information` (sticky ATC) |
| `buy-buttons-styles` | `blocks/buy-buttons`, `product-card`, `_product-card`, `accelerated-checkout`, `featured-product`, `product-hotspots` |
| `variant-picker-styles` | `variant-main-picker`, `variant-swatches`, `quick-add`, `list-filter`, card hosts |
| `quantity-selector-styles` | `quantity-selector` + card/recommendation hosts |
| `price-styles` | `snippets/price`, `unit-price`, `cart-products`, `quick-order-list`, `quick-add`, `product-grid`, `_featured-product-price` |
| `text-block-styles` | `snippets/text` + any file emitting `.text-block` |
| `swatch-styles`, `product-badges-styles`, `quick-add-styles`, `quick-add-modal-styles` | card hosts (`_product-card`, `featured-product`, `product-hotspots`, `product-recommendations` block + section); swatch also `variant-swatches`, `variant-main-picker`, `filters`, `blocks/product-card`; badges `_product-card-gallery`; quick-add `snippets/quick-add`; quick-add-modal `snippets/quick-add-modal` |
| `product-media-container-styles`, `product-media-gallery-content-styles` | both: `product-media-gallery-content`, `quick-add-modal`; container only: `card-gallery` |
| `deferred-media-video-styles` | `snippets/video`, `product-information`, `featured-product-information` |
| `deferred-media-product-model-styles` | `snippets/product-media`, `snippets/quick-add`, `product-information`, `featured-product-information` |
| `dialog-styles` | `popup-link`, `search-modal`, `quick-add-modal` (+ `quick-add-modal-styles`), `cart-disclosure-tooltip` |
| `accordion-styles`, `summary-icon-styles` | `accordion`, `disclosures`; `<summary>` icon emitters |
| others (`slideshow-styles`, `resource-list-styles`, `theme-drawer-styles`, `pills-styles`, `predictive-search-styles`, `checkbox-styles`, `fly-to-cart-styles`, `payment-terms-styles`, `video-background-styles`, `video-placeholder-styles`, `cart-typography-styles`, `collection-wrapper-styles`, `gift-card-recipient-form-styles`, `product-card-gallery-zoom-details-styles`, `product-recommendations-styles`, `text-component-shimmer-styles`, `card-hover-effect-styles`, `password-layout-styles`) | their components |

Verify the paths that bypass composition: empty → populated grid after filtering, first Quick Add open, media swaps, hydration responses.

### 9.7 Spacing

Padding settings → `spacing-style` (scaled >20px) or `spacing-padding` (raw: text-family blocks, `page`, `payment-icons`, `follow-on-shop`, `_blog-post-description`, sections `custom-liquid`, `header-announcements`, `slideshow`, `layered-slideshow`). Gap → `gap-style` (scaled >24px) via `layout-panel-style` or direct. Section padding defaults are usually 0; the first `<main>` section adds the transparent-header offset.

### 9.8 CSS for a new `cx-` block

Precedents: `blocks/spacer.liquid` (modifiers + namespaced vars + context selectors `.layout-panel-flex--row > …`), `blocks/image.liquid` (spacing/size/border helpers), `blocks/page.liquid` (contrast-override). Template: Appendix A.1.

### 9.9 Global CSS without editing `base.css`

Only for site-wide rules (tokens, links, buttons, focus, header, footer) that no setting, block stylesheet or instance style can express. Every page loads this file before it renders, so a look for one kind of section or block goes in its style snippet instead (§9.10).
- **Route A (recommended):** `assets/cx-theme.css` + one line in `snippets/stylesheets.liquid` after `base.css`: `{{ 'cx-theme.css' | asset_url | stylesheet_tag }}` (L5; covers theme, password and gift-card shells). It beats `base.css` at equal specificity, precedes the inline token `<style>` blocks (override `:root` tokens with `:root:root`), and loads before `{{ content_for_header }}` (`layout/theme.liquid:21` vs `:32`), which injects the `{% stylesheet %}` bundle, so the bundle wins equal-specificity ties (PLATFORM [LCR]): override component CSS by specificity or scoping, not source order.
- **Route B:** a `snippets/cx-<name>-styles.liquid` owner rendered from the layout (precedent `card-hover-effect-styles`, `layout/theme.liquid:22`); subject to subsetting; add to `layout/password.liquid` if needed.
- Settings-driven global values: a `{% style %}` rendered in `<head>` (pattern `theme-styles-variables`).
- Never put global overrides in a block's `{% stylesheet %}`.

### 9.10 Where a style detail goes

A visual detail of the approved design that no setting covers (a shadow, a fade direction, a column ratio, a hover effect) goes in the first row that fits. Never drop it.

| Detail | Where | Level |
|---|---|---|
| Part of a new component | That `cx-` block's own `{% stylesheet %}` (§9.8) | L6 |
| On some instances of a stock section or block | A `cx_style` option on the stock file that owns the styling (below), so the merchant turns it on per instance | L5 |
| On every instance, site-wide (all links in text, all card badges) | `assets/cx-theme.css` (§9.9 Route A) | L5 + L8 |

- MUST NOT style one instance by its generated DOM id (`#shopify-section-…`, `block.id`) or through the editor's Custom CSS (`custom_css`, §2.6): the merchant cannot see or keep that link, and the id is generated (UNVERIFIED whether it survives a theme copy; assume it does not).
- New options are saved data (§3.8): add values, never rename or remove them.
- Rules in `assets/cx-theme.css` and the `cx-` style snippets that reach into stock markup (stock classes such as `.group-block`, attributes such as `[data-menu-style]`, stock custom properties) stop working without an error if an update renames them. List each one in that file's entry in `customizations.md`, so the first check after an update knows what to retest.

**The `cx_style` option.** One per stock section or block that needs a look no setting covers, so every stock edit has the same small shape and merging it again after an update is mechanical:
1. Schema: a `header` and one `select` setting, id `cx_style`, label `t:cx.style.label`, with `""` first (`t:cx.style.default`, the stock look) and one value per look, e.g. `eyebrow` or `pill`. Its labels go under `cx.style` in `locales/en.default.schema.json` (§14), shared by every file.
2. Liquid: when the value isn't blank, add the class `cx-style-<value>` to the element the file already renders: through its snippet's `class` parameter where it has one (`{% render 'text', block: block, class: cx_style_class %}`), otherwise on its outer element.
3. CSS: each file's looks go in its own style snippet, named after the stock file (`snippets/cx-text-styles.liquid` for `blocks/text.liquid`, `snippets/cx-section-styles.liquid` for `sections/section.liquid`), holding one `{% stylesheet %}` with the rules for each `.cx-style-<value>`. The stock file renders it unconditionally next to the class line (`{% render 'cx-text-styles' %}`), so Shopify sends the looks only on pages where that section or block appears, and they're there when the merchant switches styles in the editor. The stock edit stays the schema, the class and that one line; never put the looks in the stock file's own `{% stylesheet %}`. Shopify doesn't guarantee the order of the stylesheets it bundles, so the rules must out-rank the stock ones, e.g. by starting with `:root ` (§9.9).
4. Use the same value name in every file where the look is the same; each file's snippet holds its rules, or `cx-theme.css` holds them once when the look is site-wide. A new look is a new value. A look that needs its own markup, settings or JS is a `cx-` block (L6), not a `cx_style` value.

## 10. Color system

### 10.1 Model

No color schemes (`color_scheme` unused). Color comes from:
1. One global `color_palette` (`config/settings_schema.json:63-70`; schema default slots `background`, `foreground`; saved data adds `color1`–`color3`) and role settings (`page_background_color`, `page_text_color`, `palette_primary_button_*`, `palette_secondary_button_*`, `palette_input_*`, `palette_variant_*`, `palette_selected_variant_*`, `popover_shadow_color`/`popover_drop_shadow`) → `:root` tokens in `snippets/color-palette.liquid`. Drawer and popover background/text/border colors and badge background/text colors feed the UI scopes and `--style-border-popover/-drawer`; `quick_add_background`/`quick_add_text` feed `.custom-color-quick-add` (`snippets/theme-styles-variables.liquid:653`).
2. Four global UI scopes (`preset: 'ui'`): `.color-custom-popover`, `.color-custom-drawer`, `.color-custom-badge-sale`, `.color-custom-badge-sold-out` (`:659-687`).
3. Per-section/block `background_color`/`text_color` → `contrast-override` → `.color-custom-<id>`.
4. Per-block inline overrides (`--color` on text, `button-custom-styles`, `input-custom-styles`, `variant-button-custom-styles`).

Palette references: a `color` value may be `"{{ settings.color_palette.<slot> }}"`; Shopify resolves it before Liquid reads it, and a deleted palette color is stored as a reference to its replacement (PLATFORM [IS]). Prefer references over hex so palette edits propagate.

### 10.2 `contrast-override`

- Params: `background_color`, `text_color`, `border_color`, `section_id` (required, unique), `skip_contrast`, `preset` (`content` | `ui`). No output when background is blank/alpha 0 and no text color.
- Text color: explicit `text_color` → `page_text_color` if contrast ≥4.5 → `--palette-darkest` (bg brightness >128) else `--palette-lightest`.
- With a background, on `.color-custom-<id>, .color-custom-<id> .text-block`: `--color-background(-rgb)`, brightness opacities, `--color(-rgb)`, `--color-foreground(-rgb/-muted/-subdued)`, `--color-border(-rgb)`; on `.color-custom-<id>`: `--input-disabled-*`, `--checkbox-border`, `color`, `background-color` (`snippets/contrast-override.liquid:104-144`).
- Text color only: `--color(-rgb)`, `--color-foreground(-rgb/-muted/-subdued)`, `--input-disabled-*`, `--checkbox-border`, `color`; `--color-border(-rgb)` only for `ui`; `content` also sets `color` on `.text-block`, `.custom-typography`, footer-utilities and social-icons descendants (`:145-197`).
- Never emits button or input chrome tokens: those keep global palette colors inside a custom scope.
- `skip_contrast: true` when a background image/video shows (background tokens only).
- Render once per scope with a unique id; add `color-custom-<id>` only when a color is set (sections: both `.section-background` and `.section`). Stock `snippets/section.liquid:22-24` adds it unconditionally; harmless.

### 10.3 Color-aware CSS

- Read the inherited `--color-*` tokens (`:root`, `.color-custom-<id>`, UI scopes); never re-declare them. Complete colors: `color: var(--color-foreground)`, `background: var(--color-background)`, `border-color: var(--color-border)`. Text: `color: var(--color, var(--color-foreground))`. Alpha: `rgb(var(--color-foreground-rgb) / var(--opacity-70))`.
- Global `-rgb` companions exist for background, foreground, border, palette extremes (`--palette-lightest/-darkest`), input text, variant text, selected variant text, popover shadow, white, `--color` (scoped), `--backdrop-color-rgb` (→ never-valued `--color-shadow-rgb`, §19); `--color-drawer-shadow-rgb` is declared but empty (its setting is undefined, §11). Component scopes add `--hover-color-rgb`, `--color-transparent-text-rgb`, `--star-fill-color-rgb`, `--pill-label(-background)-color-rgb`, `--menu-top-level-font-color-rgb`. When overriding a token, override its `-rgb` companion too.
- Tints that must stay visible on dark backgrounds: `--opacity-40-60`, `--opacity-35-55`, `--opacity-10-25`. Muted text: `--color-foreground-muted/-subdued`.
- A scope that changes `--color-foreground-rgb` MUST re-declare the derived tokens (`-muted/-subdued`, `--input-disabled-*`, `--checkbox-border`): reuse `contrast-override`. No hard-coded hex. Verify contrast (4.5:1 text, 3:1 large text/UI).

## 11. Global settings (`config/settings_schema.json`)

17 groups after `theme_info`.

| Group (line) | Setting IDs |
|---|---|
| Logo and favicon (11) | `logo`, `logo_inverse`, `logo_height`, `logo_height_mobile`, `favicon` |
| Colors (60) | `color_palette` (the only `color_palette` setting) |
| Typography (73) | `page_text_color`; `type_body_font`, `type_subheading_font`, `type_heading_font`, `type_accent_font`; `type_size_paragraph`, `type_line_height_paragraph`; per heading N=1…6: `type_font_hN` (h1/h2: heading\|accent; h3–h6: + subheading\|body), `type_size_hN` (10–184 select), `type_line_height_hN`, `type_letter_spacing_hN`, `type_case_hN` |
| Page (1101) | `page_background_color`, `page_width` narrow\|normal\|wide (=narrow) |
| Animations (1132) | `page_transition_enabled`, `transition_to_main_product`, `add_to_cart_animation`, `card_hover_effect` none\|lift\|scale\|subtle-zoom |
| Badges (1180) | `badge_position`, `badge_corner_radius`, `badge_sale_*`, `badge_sold_out_*`, `badge_font_family`, `badge_text_transform` |
| Buttons (1286) | `palette_primary_button_{background,text,border}`, `primary_button_border_width`, `button_border_radius_primary`, `type_font_button_primary`, `button_text_case_primary`, secondary equivalents, `pills_border_radius` |
| Cart (1463) | `cart_type` page\|drawer, `product_title_case`, `cart_price_font`, `auto_open_cart_drawer`, `show_cart_note`, `cart_note_open_by_default`, `show_add_discount_code`, `show_installments`, `show_accelerated_checkout_buttons`, `empty_cart_button_link`, `cart_thumbnail_*` |
| Drawers (1625) | `drawer_background_color`, `drawer_text_color`, `drawer_border_color` |
| Icons (1649) | `icon_stroke` thin\|default\|heavy |
| Input fields (1674) | `palette_input_{background,text,border}`, `input_border_width`, `inputs_border_radius`, `type_preset` |
| Popovers and modals (1757) | `popover_background_color`, `popover_text_color`, `popover_border_radius`, `popover_border_color`, `popover_border_width`, `popover_drop_shadow`, `popover_shadow_color` |
| Prices (1815) | `currency_code_enabled_{product_pages,product_cards,cart_items,cart_total}` |
| Product cards (1848) | `quick_add`, `mobile_quick_add`, `quick_add_background`, `quick_add_text`, `show_second_image_on_hover`, `product_card_carousel` |
| Search (1896) | `empty_state_collection`; `product_corner_radius`, `card_corner_radius`, `card_title_case` (consumed only by `snippets/search-modal.liquid:44`) |
| Swatches (1947) | `show_variant_image`, `variant_swatch_{width,height,radius,border_style,border_width,border_opacity}` |
| Variant pickers (2026) | `palette_variant_*`, `palette_selected_variant_*`, `variant_button_border_width`, `variant_button_radius`, `variant_button_width` |

- `config/settings_data.json`: `{ "current": {…}, "presets": { "Default": {…} } }`; IDs absent from `current` use schema defaults. Edit `current` only; select values are strings. Theme style presets ≤5 (PLATFORM [UPD]).
- Adding a global setting = L5 edit of `config/settings_schema.json` (+ locale keys).
- Read but undefined (no effect in stock): `drawer_drop_shadow`, `drawer_shadow_color` (`snippets/color-palette.liquid:221,225`), `social_twitter_link` (`snippets/meta-tags.liquid:87,90`), `button_text_case` (`blocks/add-to-cart.liquid:20`); computed key `settings['--font-body--case']` (`snippets/theme-styles-variables.liquid:345`, always nil, so the paragraph case token is empty).

## 12. JavaScript

Liquid renders; JS enhances. Start with native links, buttons, forms, `details`, `dialog`, `popover`. Business logic, translations and money formatting stay server-side. No build step (`assets/package.json` = `{"type":"module"}`), no bundler, no runtime deps beyond stock (`@shopify/events` CDN module, local `popover-polyfill.js`, `qr-code-generator.js`). Plain ES modules in flat `assets/`, JSDoc-typed (`assets/jsconfig.json`: `checkJs`, `noImplicitAny`, `noUncheckedIndexedAccess`, strict null checks; `@theme/*` → `./*`, `@shopify/events` → local `.d.ts`).

### 12.1 Loading

Three separate decisions: **resolution** (an import-map entry in `snippets/scripts.liquid:7-41` makes `@theme/<name>` resolve; needed only if something imports it), **early fetch** (`<link rel="modulepreload">`: utilities, component, section-renderer, section-hydration, morph, focus, recently-viewed-products, scrolling, events; paginated-list* only on collection/search) and **execution** (`<script type="module" src>` or an import reached from one).

Import map (29 entries): `@theme/{overflow-list, component, dialog, events, view-event-elements, focus, morph, paginated-list, performance, product-form, recently-viewed-products, scrolling, section-renderer, section-hydration, money-formatting, utilities, variant-picker, variant-resolution, media-gallery, quick-add, paginated-list-aspect-ratio, component-cart-items, component-quantity-selector, comparison-slider, sticky-add-to-cart, fly-to-cart, scroll-container, theme-drawer}` + `@shopify/events` → `https://cdn.shopify.com/storefront/standard-events.js`. `templates/gift_card.liquid:13-19` has its own three-entry map (`@theme/component`, `@theme/utilities`, `@theme/qr-code-generator`). Always import with the bare `@theme/<name>` specifier.

Globally executed by `snippets/scripts.liquid`: `standard-actions-override`, `page-view-event`, `overflow-list`, `dialog`, `variant-picker`, `product-card`, `product-form`, `fly-to-cart`, `accordion-custom`, `media`, `product-price`, `product-sku`, `product-inventory`, `show-more`, `slideshow`, `layered-slideshow`, `anchored-popover`, `floating-panel`, `video-background`, `component-quantity-selector`, `media-gallery`, `rte-formatter`, `volume-pricing`, `price-per-item`, `volume-pricing-info`, `auto-close-details`. Gated: `quick-add` (`quick_add or mobile_quick_add`), `cart-discount` (`show_add_discount_code`), `localization` (>1 country/language), `sticky-add-to-cart` + `RecentlyViewed.addProduct` (product template), `view-transitions` (inline, transition settings).

Loaded by their owners (the pattern for new blocks): `facets.js` + `scrolling.js` (`blocks/filters.liquid:13-23`), `local-pickup.js` (`blocks/buy-buttons.liquid:259-263`), `product-custom-property.js` (`blocks/product-custom-property.liquid:207-211`), `marquee.js` (`blocks/_marquee.liquid:1-5`), `comparison-slider.js`, `product-recommendations.js`, `disclosures-summary-fit.js`, `header.js`, `header-menu.js`, `product-hotspot.js`, `collection-links.js`, `announcement-bar.js` (`section.blocks.size > 1`), `results-list.js`, `blog-posts-list.js`, `quick-order-list.js`, `cart-drawer.js`, `cart-note.js` (`show_cart_note`), `zoom-dialog.js` (`zoom`), `drag-zoom-wrapper.js`, `jumbo-text.js`, `predictive-search.js`, `theme-drawer.js`.

`Theme` global (`snippets/scripts.liquid:284-312`): `Theme.routes` (`cart_add_url` (+`.js`), `cart_change_url`, `cart_update_url`, `cart_url`, `predictive_search_url`, `search_url`), `Theme.translations` (8 fixed keys), `Theme.template.name`, `Theme.supportsNativePopover`, `Theme.popoverPolyfillReady`. Use `Theme.routes`, never hard-coded URLs. New UI strings reach JS via `data-*` attributes rendered with `| t` (`blocks/product-custom-property.liquid:28-30`).

Adding a module:
1. Reuse an existing module if it owns the behavior.
2. Put `<script src="{{ 'cx-name.js' | asset_url }}" type="module" fetchpriority="low"></script>` in the file that renders the element, under the same Liquid condition; execute once.
3. Import-map entry only if something imports it (L5 edit of `snippets/scripts.liquid`).
4. If the element can first appear through a Section Rendering morph, the module must already be on the page: full-mode `morph` re-creates only `.shopify-app-block script[src]` (`assets/morph.js:139-142`), hydration mode none (`:124-127`). Stock loads such modules globally.

### 12.2 `Component` (`assets/component.js`)

Skeleton: Appendix A.2.

- `ref="name"` → `this.refs.name`; `ref="name[]"` → array in DOM order. `this.refs` is replaced on every refresh: never cache it; destructure at use time.
- Ref scope stops at the nearest nested component: any `Component` instance **or any tag ending in `-component`** (even before upgrade). Name custom elements `*-component`.
- Refs refresh synchronously in `connectedCallback`, asynchronously via a `MutationObserver` attached only once the browser is idle after connect (earlier mutations are missed), and in `updatedCallback()`. Treat conditional refs as optional (`[name]` in the typedef) and check before use; `requiredRefs` (throws `MissingRefError`) only for fixed markup.
- Declarative events: one capture-phase listener per type on `document`. Supported: `click, change, select, focus, blur, submit, input, keydown, keyup, toggle, pointerdown, pointerenter, pointerleave`.

| Attribute | Target |
|---|---|
| `on:click="/method"` | closest component (incl. the element itself) |
| `on:click="/method/arg"` | same; `arg` coerced (`true`/`false`/numbers), passed **first**, event last |
| `on:click="/method?a=1&b=x"` | same; argument is an object |
| `on:click="selector/method"` | `element.closest(selector)` (any CSS selector without `/` or `?`) |
| `on:click="#id/method"` | `document.querySelector('#id')` |

Argument values cannot contain `/` or `?`. Non-bubbling `toggle` must sit on the `<details>`; `pointerenter`/`pointerleave` match only the exact target. Handlers are public methods documented with JSDoc. Synchronous throws are logged, async rejections are not caught.
- Lifecycle (MUST): `super.connectedCallback()` before using refs; `super.updatedCallback()` before reading refs after DOM changes; `super.disconnectedCallback()`. The base class disconnects only its own `MutationObserver` (`assets/component.js:102-104`): clean up every listener, timer, observer, rAF, media-query listener and request; use an `AbortController` re-created on each connect (`assets/disclosures-summary-fit.js:48-50,82-90`). Reconnects and re-renders must be idempotent.
- Morph skips `isEqualNode`-equal subtrees, so `updatedCallback` does not fire for them. `morphSection` parses the response with `DOMParser` (`assets/section-renderer.js:242`); `morph` calls `updatedCallback` only when the new node is a `Component` instance (`assets/morph.js:270`, `:97-100`), and `DOMParser` nodes are not upgraded, so treat `updatedCallback` as not fired by section morphs. Keep state derivable on connect; stock calls it by hand where needed (`assets/volume-pricing-info.js:22-27,42`).
- Downward via public methods; upward via bubbling events; never depend on parent tag names except declared `on:` targets.
- Plain `HTMLElement` subclasses exist on purpose where refs must stay visible to a parent (`accordion-custom`, `scroll-hint`); `DeclarativeShadowElement` in `overflow-list`. Keep them.
- Style: `async`/`await`, early returns, `for…of`, `URL`/`URLSearchParams`, validate event targets before type-specific APIs.

### 12.3 Events

`@shopify/events` (`StandardEvents`):

| Event | Name | Producers → consumers |
|---|---|---|
| `ProductSelectEvent` | `shopify:product:select` | `variant-picker.js` → product-form, product-price, product-inventory, product-sku, media-gallery, sticky-add-to-cart, local-pickup, product-card, quick-add |
| `CartLinesUpdateEvent` | `shopify:cart:lines-update` | product-form, component-cart-items, quick-order-list → cart-icon, cart-drawer, cart items, header-actions, sticky ATC, quick-add, price-per-item, product-form, gift-card recipient form, quick-order-list |
| `CartErrorEvent` | `shopify:cart:error` | cart producers → sticky ATC, gift-card form |
| `CartNoteUpdateEvent`, `CartDiscountUpdateEvent` | `shopify:cart:note-update`, `…:discount-update` | cart-note, cart-discount → cart items |
| `SearchUpdateEvent`, `CollectionUpdateEvent` | `shopify:search:update`, `shopify:collection:update` | facets, predictive-search → paginated-list, facets |
| `PageViewEvent`, `ProductViewEvent`, `CollectionViewEvent`, `CartViewEvent` | `shopify:*:view` | `assets/page-view-event.js` on DOM ready; view-event elements `product-component`, `collection-component` (`assets/view-event-elements.js`) and `cart-items-component` (`assets/component-cart-items.js:34`) carrying `view-event-payload="{{ x \| standard_event_data: 'view' \| escape }}"` |

Read an event's definition, producers and consumers together before using it. Producer: `const { promise, resolve, reject } = XEvent.createPromise()`; dispatch `new XEvent({…, promise})` on `this` **before** the request; settle after. Consumer: filter by `event.target`; `event.promise.then(({ detail }) => …).catch((e) => { if (e?.name !== 'AbortError') console.warn(e) })`. Listen on `this.closest('.shopify-section, dialog')` for product events (ignore `product-card` sources unless intended), on `document` for cart events.

`@theme/events` (`assets/events.js`, all bubble): `quantity-selector:update` (`{quantity, cartLine}`), `media:started-playing`, `megaMenu:hover`, `zoom-media:selected`, `cart-section:restored`, `slideshow:select`. Elsewhere: `dialog:open`/`dialog:close` (**non-bubbling**, on `dialog-component`), `theme-drawer:open`/`:close` (bubbling), `overflowMinimum`, `reflow`, `recipient:toggle` (`{mode, recipientFormVisible}`, `assets/gift-card-recipient-form.js`). New internal events: `class CxThingEvent extends Event` with `bubbles: true` and `detail`, in the style of `events.js`.

`assets/standard-actions-override.js` configures `Shopify.actions.updateCart` (target `theme-drawer#cart-drawer`, else `document`) and `openCart` (opens the drawer, else `Theme.routes.cart_url`); `getCart` is not overridden. Components POST to the cart themselves with `fetchConfig()` + `Theme.routes`.

### 12.4 Section Rendering, morph, hydration

- `sectionRenderer.renderSection(sectionId, { cache = !Shopify.designMode, mode = 'full' | 'hydration', injectStylesheet = false, url, shouldRender })` fetches `?section_id=` (with `cache` true, the storefront default, it reuses HTML already cached for that URL, seeded from live sections at `load`; pass `cache: false` for fresh HTML), morphs `#shopify-section-<id>` and returns the HTML. It awaits the morph and propagates failures; fetch and missing-wrapper failures reject: always `await` and handle rejection. It aborts an in-flight render of the same section; a superseded call settles with the newer render's promise, resolves to `''` (aborted, no newer render), or resolves to its own unapplied HTML (abort after fetch), so re-check user intent before acting on the result. HTTP status is not checked. Pass unprefixed ids; live DOM and response both need the `shopify-section-<id>` wrapper.
- `sectionRenderer.getSectionHTML(id, useCache, url, signal)`, `abortRender(id)`.
- `morphSection(sectionId, html, { mode, injectStylesheet })` throws if the wrapper is missing in DOM or response; no stale-update guard.
- `morph(old, new, options)`: keyed by `id` (or `getNodeKey`); preserves live `open` on `details`/`dialog` (unless the new node has `declarative-open`) and `slot`/`sizes`; never morphs `SHOPIFY-ACCELERATED-CHECKOUT-CART`. Opt-outs: `data-skip-node-update` on both nodes (keep attrs and form state, recurse); `data-skip-subtree-update` on both (skip children); `data-skip-value-update` on the live node keeps the typed value only while the input is focused and the incoming input is not disabled. Without it, focused values are overwritten.
- Hydration mode morphs only elements with matching non-empty `data-hydration-key` (stable, unique, identical in initial and fetched markup); unmatched targets are never inserted or removed, but a matched target's attributes and descendants can change. Use full mode for structural changes.
- `hydrate(sectionId[, url])` (`@theme/section-hydration`): waits for DOM ready + idle, renders with `cache: false, mode: 'hydration'`, sets `data-hydrated="true"`; accepts `<id>` or `shopify-section-<id>`; skips missing or hydrated sections; its promise covers scheduling only. Stock user: header (`sections/header.liquid:1235-1242`, not in design mode). Other hydration-mode users: product recommendations (`assets/product-recommendations.js:100`, `{mode:'hydration', injectStylesheet:true}`), cart items (`assets/component-cart-items.js:77,104,128,196,402,475`, `mode: this.isDrawer ? 'hydration' : 'full'`), cart discount (`assets/cart-discount.js:92,260,355`).
- CSS in partial responses: full-section insertion keeps the response CSS; for hydration/fragment updates opt in with `injectStylesheet: true`, which takes the first `style[data-section-stylesheet]` in the response and updates or prepends the live section's style (no-op if absent). `hydrate()` does **not** opt in: keep early style-owner renders or do an explicit injecting update.
- Cancel stale requests; never let an older response overwrite newer intent; update the smallest region. Max 5 sections per Section Rendering request (PLATFORM [SRA]).

### 12.5 Theme editor

`assets/theme-editor.js` (design mode only) handles `shopify:block:select`/`deselect` (slideshow select/pause, carousel scroll, layered-slideshow panel select, product-card navigation off) and `shopify:section:load`/`unload` (header CSS vars); persists overlay state across refreshes. Editor events bubble; `event.target` is the section/block element. Make connect/disconnect idempotent (the editor replaces section DOM). Use `Shopify.designMode` / `request.design_mode` only for editor affordances. `oncePerEditorSession(el, key, cb)` (`@theme/utilities`) for intro animations. A new block that must react to selection listens on `document` for `shopify:block:select` in its own module; do not edit `theme-editor.js`.

### 12.6 Utilities

| Module | Exports |
|---|---|
| `@theme/utilities` | `debounce`, `throttle`, `requestIdleCallback`, `yieldToMainThread`, `fetchConfig(type, {body, headers})`, `prefersReducedMotion()`, `mediaQueryLarge`, `isMobileBreakpoint()`, `isDesktopBreakpoint()`, `isTouchDevice()`, `isRTL([element])`, `getDirectionMultiplier(element)`, `lockScroll(owner)`/`unlockScroll(owner)`, `isClickedOutside`, `onDocumentReady`, `onDocumentLoaded`, `onAnimationEnd`, `startViewTransition(cb, types)`, `viewTransition`, `preloadImage`, `getViewParameterValue`, `oncePerEditorSession`, `ResizeNotifier`, `scheduler`, header helpers |
| `@theme/scroll-container` | `getScrollContainer`, `getScrollTop`, `scrollTo`, `getScrollEventTarget()` (`.page-wrapper` ≥990px or `document`), `getIntersectionRoot()`, `scrollContainerMediaQuery` (`(min-width: 990px)`); importing it patches `history.pushState` |
| `@theme/focus` | `trapFocus(container)` (one active trap), `removeTrapFocus()`, `cycleFocus(items, increment)` |
| `@theme/dialog` | `dialog-component` (`showDialog`, `closeDialog`, `toggleDialog`; scroll lock; Escape/outside click) |
| `@theme/theme-drawer` | `theme-drawer` (`open`, `close`, `toggle`; modal <990px, inline + squeeze ≥990px; `no-persist` attr) |
| `@theme/scrolling` | `Scroller`, `scrollIntoView` (RTL-aware) |
| `@theme/morph`, `@theme/section-renderer`, `@theme/section-hydration` | §12.4 |
| `@theme/money-formatting` | `formatMoney`, `convertMoneyToMinorUnits` (prefer server-rendered money) |
| `@theme/performance` | `cartPerformance` (`createStartingMarker`, `measureFromEvent`, `measureFromMarker`, `measure`) |
| Non-mapped globals | `anchored-popover-component` (`ref="trigger"`, `ref="popover"`), `floating-panel-component`, `accordion-custom`, `slideshow-component`, `show-more-component`: reuse via markup. `marquee-component` is loaded by its owners (`blocks/_marquee.liquid:1-5`, `sections/marquee.liquid`): include its script when reusing it |

Reuse an existing component through markup before writing a new one.

## 13. Commerce surfaces

### 13.1 Product page

```
templates/product.json
  main: product-information
    static media-gallery: _product-media-gallery
    static product-details: _product-details   (@theme, @app, content blocks, product blocks, _divider)
       group → text "<h1>{{ closest.product.title }}</h1>", price
       _divider, variant-picker, buy-buttons (static quantity / add-to-cart / accelerated-checkout),
       text "{{ closest.product.description }}"
    dynamic (section area, after the product grid): disclosures
  product_recommendations_qggXJq: product-recommendations (product "{{ closest.product }}", related)
    text "<h2>You may also like</h2>"; static static-product-card: _product-card (gallery, title, price)
```

- `sections/product-information.liquid`: JSON-LD `closest.product | structured_data`; passes `closest.product: closest.product` to both statics; sticky add-to-cart bar when `enable_sticky_add_to_cart` (default true; its markup uses the global `product`, `:32-53`); layout via `snippets/product-information-content.liquid` (shared with `featured-product-information`); `presets: []`.
- `[data-product-grid-content]` (the `<product-component>` wrapping media + details) is what Quick Add clones; section-level blocks (e.g. `disclosures`) are outside it.
- `_product-details` accepts `@theme`, so any public block works in the details column; outside a product context `closest.product` is blank.
- Media: `_product-media-gallery` (grid|carousel, columns, thumbnails, zoom, `hide_variants`, aspect ratio, `constrain_to_viewport`) → `snippets/product-media-gallery-content.liquid` → `snippets/product-media.liquid`; `<media-gallery>` replaces itself from the variant response.

Product form contract (`blocks/buy-buttons.liquid` + `assets/product-form.js`), MUST be preserved:
- Form id `BuyButtons-ProductForm-{{ section.id }}` → **one `buy-buttons` per section**. `{% form 'product', product, id: …, data-type: 'add-to-cart-form' %}` with hidden `<input name="id" ref="variantId">` = selected/first-available variant.
- Static children: `quantity` (`name="quantity"`, min/max/step from `quantity_rule`; markup in `snippets/quantity-selector.liquid:65`), `add-to-cart` (`<add-to-cart-component>` + `button[type=submit][name=add][ref=addToCartButton]`, markup in `snippets/add-to-cart-button.liquid:21-32`), `accelerated-checkout` (`{{ form_obj | payment_button }}`; `form_obj` is passed by `buy-buttons` via `content_for`, `:233-238`; nothing renders without it).
- Availability computed in Liquid (`can_add_to_cart`: Add to cart / Sold out / Unavailable incl. quantity rules + deny policy); volume pricing table; gift-card recipient form; local pickup drawer.
- Line-item properties: `product-custom-property` inputs sit outside the form, bound by `form="BuyButtons-ProductForm-<section.id>"` and `name="properties[<key>]"`. New buyer inputs follow this pattern.
- Submit: `/cart/add.js` with `sections` for every `cart-items-component`; dispatches `CartLinesUpdateEvent` (promise) before the request; errors → `CartErrorEvent` + live region.
- No selling-plan selector exists; subscriptions need an `@app` block. Quick Add routes selling-plan products to the product page.
- Keep the variant id input, availability states, quantity rules, accelerated checkout, `type=submit`, `ref`s and `on:` attributes. Restyle through settings/CSS.

Variant change (`assets/variant-picker.js`):
- On the product page the **whole product page** is fetched (`<url>?option_values=<ids>`). `section_id` is appended only when the picker is inside `quick-add-component` or `swatches-variant-picker-component` (card chooser/swatches → `section-rendering-product-card`) or `featured-product-information` (`:282-286`); the Quick Add modal fetches the full product page. Combined-listing switches morph all of `<main>` (`:361-362`).
- `ProductSelectEvent` → product-form (variant id, button state, quantity rules, volume pricing), `product-price` (by `data-block-id`), `product-inventory`, `product-sku-component`, `media-gallery`, sticky bar, local pickup, URL `?variant=`.
- **Not refreshed on variant change** on the product page: `text`, `product-title`, `review`, `product-description`, `custom-liquid`, `image`, `disclosures`, `product-custom-property`, app blocks (variant-level dynamic sources stay at the initial variant). `featured-product-information` morphs its whole element and combined-listing switches morph `<main>`, so there everything refreshes. Variant-reactive custom UI: a `cx-` component listening for `StandardEvents.productSelect` on `closest('.shopify-section, dialog')`, awaiting `event.promise` and replacing its node from `detail.html` by a stable `data-block-id` (`assets/product-price.js:21-88`). Keep one instance per section of response-matched elements (`media-gallery`, `product-inventory`, `[ref=addToCartButton]`).

### 13.2 Featured product sections

| Section | Notes |
|---|---|
| `featured-product-information` (Featured product) | Product-page layout for a chosen product on any template; static `_featured-product-information-carousel` + `_product-details` (`@theme`); variant changes re-render it via `section_id`; no sticky bar. `product-inventory`'s listener compares `detail.productId` with `data-product-id` (global `product`), so there it updates only through the whole-section morph. |
| `featured-product` (Product highlight) | Static only: `_media-without-appearance` (media) + `_featured-product` → static `product-title`, `_featured-product-price`, `_featured-product-gallery` (+ quick add), `swatches`; no `@app`. |
| `product-hotspots` | `_hotspot-product` (own `product`, x/y %); dialog with price + quick add. |

### 13.3 Product card system

| Host (static `_product-card`) | id | `product_view_context` |
|---|---|---|
| `sections/main-collection.liquid:70-75` | `product-card` | collection |
| `sections/search-results.liquid:56-61` | `product-card` | search |
| `sections/product-list.liquid:37-42` (+ placeholder `:55`) | `static-product-card` | collection |
| `sections/product-recommendations.liquid:100-105` | `static-product-card` | recommendation |
| `blocks/product-recommendations.liquid:99-103` | `static-product-card` | — |

- One card configuration per host instance, edited in that section's static `_product-card` (e.g. `templates/collection.json:122-215`).
- `_product-card` children: `_product-card-gallery`, `product-title`, `price`, `text`, `image`, `review`, `sku`, `swatches`, `buy-buttons`, `custom-liquid`, `_product-card-group`, `@app`. **Not allowed:** `variant-picker`, `product-inventory`, `product-description`, `quantity`, `product-custom-property`, `button`, `icon`. Public `product-card` (own `product` setting) = same list minus `buy-buttons`. `buy-buttons` in cards duplicates form ids per section (stock behavior).
- Badges: only sale (`compare_at_price > price`) and sold-out, styled by global badge settings, rendered by `_product-card-gallery`. **No badge block.** Custom badges: a `text` block (pill settings) bound to a metafield inside the static `_product-card` (L2/L3), or a `cx-` block when logic is needed (L6, plus an L5 edit listing it in `_product-card`, which has no `@theme`, §3.6).
- Card data: `text` blocks with dynamic sources (`{{ closest.product.metafields.ns.key }}`, `{{ closest.product.vendor }}`); `closest.product` is supplied per card (`blocks/_product-card.liquid:10`).
- Global card settings: `quick_add`, `mobile_quick_add`, `show_second_image_on_hover`, `product_card_carousel`, badges, `card_hover_effect`, `transition_to_main_product`, `currency_code_enabled_product_cards`.
- `snippets/product-card.liquid` renders `product-card-styles`; `snippets/card-gallery.liquid` handles media, carousel, hover image, eager loading when `section.index` is nil or `< 5` (`:102`), `sizes` from host `section.settings` (safe fallback).
- Reusing `_product-card` in a `cx-` section: render it statically in the loop with `closest.product: product`; provide `columns`/`layout_type` (or `product_card_size`) for accurate `sizes`; render the owners stock hosts render (`quick-add-styles`, `quick-add-modal-styles`, `swatch-styles`, `quantity-selector-styles`, `variant-picker-styles`, `buy-buttons-styles`, `product-badges-styles`; `sections/featured-product.liquid:13-19`).

### 13.4 Collection and search

- `templates/collection.json`: `section` (title + description `text` bound to `closest.collection`) → `main-collection` (`enabled_on: collection`, static `filters` + `_product-card`).
- `main-collection`: `<collection-component>` → `<results-list infinite-scroll>`; 24 per page with infinite scroll (`enable_infinite_scroll`, default on); otherwise `products_per_page` 8–36 (=24) with pagination (`sections/main-collection.liquid:54-58`, `sections/search-results.liquid:30-34`); `layout_type` grid|organic, `product_card_size`, `product_grid_width`, gaps.
- `filters` (static): horizontal|vertical, drawer on mobile (`<theme-drawer id=filters-drawer>`), sorting, density, price and list/swatch filters. Which filters exist = Search & Discovery app (L0). Flow: `assets/facets.js` → URL params → `sectionRenderer.renderSection(sectionId)` (inside a `product-grid` view transition unless in a dialog) + `CollectionUpdateEvent`/`SearchUpdateEvent` carrying the render promise.
- Pagination: `assets/paginated-list.js` (prefetch ±1 page via `getSectionHTML`, infinite-scroll sentinels, `?page=N`).
- `snippets/product-grid.liquid` empty state pre-renders card style owners so morphed-in cards are styled.
- Search: `search-header` (static `_heading`, `_search-input`) + `search-results` (filters only when results > 0; empty state uses `settings.empty_state_collection`). Predictive search: `snippets/search-modal.liquid` → `sections/predictive-search.liquid` (SRA) via `assets/predictive-search.js`.
- Collection lists: `collection-list` (4 layouts), `main-collection-list` (all collections, max 20), `collection-links`.

### 13.5 Cart

- `settings.cart_type` page|drawer. Drawer = `snippets/cart-drawer.liquid` (rendered by the layout; SRA target `sections/cart-drawer-section.liquid`), **snippet-built, not block-configurable**: change it via cart settings or an L5 snippet edit.
- Cart page = `main-cart` (static `_cart-title`, `_cart-products`, `_cart-summary`; dynamic area `@theme`, `@app`, text/icon/image/button/video/group/spacer) + `product-list`.
- `snippets/cart-products.liquid`: `form#cart-form` (`/cart`, POST), line items (`_`-prefixed properties hidden, selling plan name, discounts), `quantity-selector` with `name="updates[]"`. `snippets/cart-summary.liquid`: discounts (`cart-discount-component`), note (`cart-note`), totals, installments, checkout button `form="cart-form"`, `content_for_additional_checkout_buttons`.
- `CartItemsComponent` (`assets/component-cart-items.js`) updates via `Theme.routes.cart_change_url` with `sections` and morphs (`hydration` in the drawer); listens to `CartLinesUpdateEvent`, `quantity-selector:update`, note/discount events, `pageshow` (bfcache). Drawer auto-opens on add when `auto_open_cart_drawer`.
- Custom code that changes the cart MUST dispatch `CartLinesUpdateEvent` with a promise resolving to `{ cart, detail: { sections, source, sourceId, itemCount, didError, items, productId } }` (`sections` may be absent on error; `cart-items-component` then refetches) and request `sections` for every `cart-items-component` section id (`assets/product-form.js:429-578`).

### 13.6 Quick add

- Enabled by `settings.quick_add` / `mobile_quick_add` (loads `quick-add.js`, renders `quick-add-modal`). `product-hotspot.js` imports `@theme/quick-add`, so hotspots load it regardless.
- `snippets/quick-add.liquid`: single variant → "Add" (inline form `QuickAdd-ProductForm-<product.id>-<block.id>`); single option → "Choose", plus "Add" when the selected variant is available or the option has swatches (`:35-56`; `assets/quick-add.js` flips them on swatch select); multiple options → "Choose" (modal); selling plans → "Choose" that navigates to the product page.
- The modal fetches the **product's own page** and clones `[data-product-grid-content]`. Desktop CSS hides everything in details except groups, buy buttons, the title link, `variant-picker`, `product-price`, `product-inventory` (`snippets/quick-add-modal-styles.liquid:447-467`, net of `:610-613`); mobile rebuilds title + price + picker + form. A **required** `product-custom-property` is hidden on desktop and `checkValidity()` blocks submission (`assets/product-form.js:73`); on mobile the details column is removed (`assets/quick-add.js`) and the property is silently dropped. Avoid `required` with Quick Add on, or route those products to the product page.

### 13.7 Metafields and dynamic sources

- Prefer binding stock blocks to metafields (L2): `text` (`{{ closest.product.metafields.ns.key }}`), `image` (file reference), `button` link, `video`. Needs metafield definitions (L0).
- Hard-coded readers: `review` (`reviews.rating`, `reviews.rating_count`), `disclosures` (`shopify.disclosure`), cart disclosure tooltip.
- `custom-liquid` for formatting a dynamic source cannot express (L4); a `cx-` block when settings/CSS are needed (L6).
- Metaobjects: `metaobject`/`metaobject_list` settings (type must exist and be storefront-visible) in a `cx-` block/section, or `templates/metaobject/<type>.json` for metaobject pages. No stock file uses either.

## 14. Strings and translations

| Kind | Where | How |
|---|---|---|
| Storefront UI text (buttons, states, errors, labels, live-region messages) | `locales/en.default.json` (+ 33 storefront locales) | `{{ 'group.key' \| t }}`; interpolation at the call site (`\| t: count: n`); plural keys `one`/`other`; `_html` suffix for intentional markup |
| Editor labels, info, options, names, preset names/categories, translatable defaults | `locales/en.default.schema.json` (+ 22 schema locales) | literal `"t:group.key"` |
| Shop content (headlines, copy, media, links) | settings, resources, metafields | never in locale files; translated defaults become merchant content once saved |

- Schema namespaces: `names`, `settings`, `options`, `content`, `info`, `categories`, `html_defaults`, `text_defaults`. Reuse stock keys for standard settings.
- Custom keys: top-level `cx` object, `cx.<component>.<key>` (≤3 levels), in `en.default.json` (storefront) and/or `en.default.schema.json` (editor). Locale files are stock files: adding `cx` is an L8 stock edit; record it; re-apply after an update. Add the same keys to each storefront locale the shop publishes; schema-locale fallback to English is UNVERIFIED (keep editor labels in `en.default.schema.json` at least).
- Never change a stock key's text, placeholders or meaning (shared contracts); add a `cx.*` key instead. Plain keys stay plain; `_html` only for markup.
- Theme Check `TranslationKeyExists`, `ValidSchemaTranslations` report missing keys. Both `en.default*` files carry a `/* */` header and `//` comments. Limits: ≤3,400 translations per file, ≤1,000 chars per value (PLATFORM [LOC]).

## 15. Accessibility

Semantic HTML and native keyboard behavior first; ARIA only for what native semantics cannot express. Every control has a name; `aria-controls`/`aria-describedby`/`aria-expanded`/`aria-current` stay in sync. Visible focus (`*:focus-visible`, `assets/base.css:235`), logical tab order, Escape closes overlays, `trapFocus`/`dialog` containment, focus restored after close or DOM replacement. Visual order = DOM order. One live-region owner per independently updated status (stock: `[ref=liveRegion][aria-live=assertive]` in buy buttons, `role=status` in header actions). Informative images get `alt`; decorative images `alt=""`; decorative SVGs `aria-hidden="true" focusable="false"`. Contrast 4.5:1 text, 3:1 large text/UI (`contrast-override`). Touch targets ≥44×44px (`--minimum-touch-target`; PLATFORM [A11Y]), 48 recommended (PLATFORM [TAP]). Verify 200% zoom, wrapping, reduced motion, RTL, and that navigation, forms and purchase work before JS loads. One `h1` per page; stock sets heading levels in richtext (`<h1>{{ closest.product.title }}</h1>`), style via presets.

## 16. Performance

- Render static presentation in Liquid; JS only for interaction. Load component CSS/JS only where their markup can appear; before deferred content appears (empty grids, Quick Add, hydration) make sure its CSS is present (§9.6, §12.4).
- Images: `image_url | image_tag` with `width`/`height`, `widths` (4–6 values) and `sizes` matching the layout. Lazy-load below the fold; LCP image eager + `fetchpriority: 'high'` (stock: `fetchpriority` high when `section.index == 1`, `sections/hero.liquid:108-112`, no `loading` passed), never lazy, no CSS background for LCP. `sizes: 'auto'` only with `loading: 'lazy'`. `section.index` is nil in the editor, SRA and static sections → nil-safe conditions (`snippets/background-media.liquid:33-39`). Reserve space for other media. Re-check eager/preload decisions when composition changes.
- Liquid: `for` ≤50 items → `paginate`; avoid nested loops and metafield access in loops; `assign` repeated results.
- Cancel stale requests; update the smallest region. Test long text, missing media, large collections, sold-out products, slow network.

## 17. Verification

### 17.1 Checklist (every change)

1. Read changed Liquid, schema, locale, CSS and JS together: IDs, settings, refs, selectors, events, loading paths agree.
2. `validate_theme` (shopify-dev-mcp) on every created/changed file: `learn_shopify_api(api: "liquid")` for a `conversationId`, then `validate_theme(conversationId, absoluteThemePath, filesCreatedOrUpdated: [{path}])` with theme-relative paths. Fix errors and warnings; re-run with the same `artifactId` and incremented `revision`. It reads local files only: validate the project's `theme/` folder, where every change is made first (`CLAUDE.md`).
3. Compare findings on stock files with §17.2 before "fixing". `validate_theme` runs Theme Check, so no CLI is needed.
4. Preview affected templates: desktop + narrow viewport; theme editor (add, remove, reorder, select; every setting; empty and populated states); after interactions (filtering, variant change, first Quick Add open, cart drawer, section re-render); non-default locale / RTL when relevant.
5. Compare with the approved design side by side at desktop and phone widths, band by band: layout, spacing, type, color, borders, shadows, images, hover and motion. Each difference is a bug: fix it (§2.1, §9.10), or name it to the user with the reason before calling the change done.
6. Interactions: keyboard and pointer, focus movement, live feedback, reduced motion, delayed module loading, request failure, repeated section rendering.
7. Record the change (§2.4). Report each unavailable check (validation, preview, store auth, representative data) with the exact command or inspection to run next.

### 17.2 `validate_theme` baseline for stock 4.2.0 (run 2026-10-10, twice, same result)

Scope: 420 files (sections, blocks, snippets, layout, templates, config, `en.default*` locales, assets JS/CSS/SVG). 415 pass, 1 passes with 2 warnings, 4 fail with static-id errors. **Do not "fix" these:**

| File | Finding | Why intentional |
|---|---|---|
| `sections/header.liquid` | "The id 'header-menu' is already being used by another static block" ×2 | one static block rendered as 3 variants (`:86-94`) |
| `sections/main-cart.liquid` | same for `cart-page-title`, `cart-page-items` | rendered twice per request: inside `<template id="empty-cart-template">` with `force_empty: true` (`:26,29`) and live (`:45,49`) |
| `sections/product-list.liquid` | same for `static-product-card` | populated loop vs placeholder loop (`:37-42` vs `:55`) |
| `sections/collection-links.liquid` | same for `link` | text links per collection (`:56`) plus image-only copies for the spotlight slideshow (`:79`, `image_only: true`) |
| `snippets/bento-grid.liquid` | 2 warnings: opening/closing tag in different Liquid conditions | boxes opened on `forloop.first or item_modulo == 1` (`:41`), closed on `forloop.last or item_modulo == 0` (`:52`) |

The validator reports errors **and** warnings (missing `img` width/height, Liquid in `{% stylesheet %}`, `block.id` in conditions, preset `block_order` ids missing from `blocks`, unknown preset setting keys, private block not listed in parent, missing `t:` keys, non-existent block types). It did not report `ExcessiveSettingsCount` on `sections/header.liquid` (42 settings).

## 18. Recipes: need → Horizon way

| Need | Do (level) | Avoid |
|---|---|---|
| Brand colors, fonts, type scale, button/input/card style | Global settings (L1); palette references in role settings | CSS token overrides |
| New content band (text + image + buttons, columns, FAQ, icons with text, split banner) | `section` + closest of its 13 presets + `group`/`text`/`image`/`button`/`icon`/`spacer`/`_divider` (L3) | New section |
| Hero / slideshow / marquee / media-with-text / collection or product lists / blog posts | The dedicated stock section and its presets (L3) | Rebuilding them |
| Hand-picked products | `section` + `group` + several `product-card` blocks (L3) | New section |
| Products of a collection | `product-list` (L3) | — |
| Metafield on product page | `text`/`image` block in `_product-details` bound via dynamic source (L2/L3) | Editing `_product-details` |
| Badge / extra line on cards | `text` pill with metafield source in the host's static `_product-card` (L2/L3); `cx-` block if logic is needed (L6 + an L5 edit listing it in `_product-card`, §3.6) | Editing `snippets/card-gallery.liquid` |
| Different layout for some products/pages/collections | Alternate template (L3) assigned per resource | Liquid conditionals on handle |
| Collection banner | The `section` band in `collection.json` bound to `closest.collection` (L2/L3) | — |
| Header/footer changes | Header settings (rows, positions, sticky, transparent, colors); footer blocks (L2/L3) | Editing `sections/header.liquid` CSS |
| Announcement bar | `header-announcements` + `_announcement` (L3) | — |
| Popup / modal content | `popup-link` block with any blocks inside (L3) | Custom modal JS |
| Accordion content | `accordion` + `_accordion-row` (rows accept `@theme`) (L3) | Custom JS |
| Subscriptions, reviews, wishlists, size charts | `@app` blocks (L0/L3) | Theme code |
| Buyer input on product | `product-custom-property` (L3) | Custom form fields |
| Cart drawer vs page | `settings.cart_type` (L1) | — |
| Filter list | Search & Discovery app (L0) + `filters` settings (L2) | Theme code |
| Small one-off Liquid | `custom-liquid` block (L4) | — |
| New interactive component (countdown, calculator, locator) | `cx-` block + `assets/cx-*.js` Component (L6), Appendix A.2 | New section |
| Page-level region with its own data loop or grid | `cx-` section reusing stock blocks/snippets (L7), Appendix A.3 | Copying a stock section |
| Design that would rebuild a stock section (header menu, drawer) | Offer the user the §2.1 choices: new options on the stock file (L5), a `cx-` section that replaces it (L7, may start as a copy), or a design change | Stacking stock edits without asking |
| Variant-reactive custom UI | `cx-` block listening to `ProductSelectEvent` (§13.1) (L6) | Editing `variant-picker.js` |
| Site-wide style rule no setting covers | `assets/cx-theme.css` via `snippets/stylesheets.liquid` (L5 + L8, §9.9) | Editing `base.css` |
| Style detail on a stock section or block that no setting covers | A `cx_style` option on the stock file that owns it, with the look in its `cx-` style snippet (L5 + L8, §9.10) | Dropping the detail; Custom CSS; styling by generated id |

## 19. Known stock defects and quirks (do not imitate; do not fix unless asked)

| Area | Item |
|---|---|
| CSS tokens | `--color-shadow-rgb` never valued (only `inherit`, `blocks/_cart-summary.liquid:117`; read via `--backdrop-color-rgb`); `--font-paragraph-family`/`--font-paragraph-size` (single hyphen) undefined (`assets/overflow-list.css:46-47`, `blocks/_blog-post-info-text.liquid:48`); `--style-border-radius-buttons` undefined (`snippets/quantity-selector-styles.liquid:92`, `assets/template-giftcard.css:85`); `--size-style-height-mobile` never emitted (`assets/base.css:1104`, harmless: falls back to `--size-style-height`); `var(--section-min-height, 'auto')` quoted fallback; `--section-height-auto` undefined (hero `section_height: auto`, `sections/hero.liquid:473`). Read without fallback and defined nowhere: `--font-paragraph-medium--size`, `--font-paragraph-small--size`, `--font-weight-{semibold,medium}` (and `--font-weight-{bold,normal}` in `cart-summary`, `cart-products`, `pagination-controls`; `blocks/disclosures.liquid` gives them fallbacks), `--font-body--size`, `--font-size--body-md/-body-sm/-xlarge`, `--font-size-lg/-xs`, `--font-line-height-md`, `--line-height-tight`, `--color-foreground-secondary`, `--color-background-secondary`, `--color-primary-active`, `--cart-font-size--sm/--xl`, `--spacing-lg/-xs`, `--product-card-alignment`, `--product-card-border-width/-opacity`, `--media-border-opacity`, `--image-opacity`, `--grid-area-alignment`, `--section-padding-inline`, `--facets-panel-width`. Never copy these names. |
| Settings | Undefined global settings read: `drawer_drop_shadow`, `drawer_shadow_color`, `social_twitter_link`, `button_text_case` (§11); `button_text_case_*` emits `default` as a `text-transform` value; paragraph case token built from a non-existent key; `_collection-link` reads undefined `section.settings.image_height`; `_slide` tests `slide_height == 'adapt_image'` (not an option); hero and `_blog-post-title` presets set `font` to `var(--font-primary--family)`, a value the `font` select does not offer and no token defines (`sections/hero.liquid:1440,1464`, `blocks/_blog-post-title.liquid:383`); `media-with-content`, `password` and (via snippets) `_content`, `_content-without-appearance`, `_media-without-appearance`, `_featured-product-gallery`, `_featured-product` render `border-override` without border settings; `_content.background_color` and `page.max_width` have no effect |
| Docs vs code | `gap-style` doc says threshold 20, code 24; `contrast-override` doc mentions accent tokens never emitted; `typography-style` doc says preset defaults come from global settings (it reads the passed settings); `anchored-popover.js` doc example uses `data-ref` (framework reads `ref`); `snippets/media.liquid` doc names a non-existent `_media` block; base.css comment says narrow width 1400px (token is 90rem = 1440px) |
| Editor attributes | Missing `block.shopify_attributes` with `tag: null`: `custom-liquid`, `_marquee`, `_collection-image`; not on the top element: `menu`, `_accordion-row`, `footer-policy-list`, `_search-input`, `_carousel-content`, `_product-details`, `quantity`; printed three times: `_slide` (`snippets/slideshow-slide.liquid:46,59` + `blocks/_slide.liquid:106`); duplicated: `icon`, `product-custom-property`, `product-recommendations`; `snippets/slideshow-slide.liquid` emits `style` twice (`:42`, `:48`), dropping a passed `style`; `class` with `tag: null` has no effect (`accordion`, `_header-logo`) |
| DOM ids | Not unique per instance: `contact-form` (`ContactForm-*`), `popup-link` (`popup-dialog-heading`), `footer-policy-list` (`terms-policies-popover`), `review` (SVG ids), `localization-form` (`country-filter-input`, `no-results-message`; rendered in header and drawer); both header rows carry `ref="headerRowTop"` |
| Commerce | `variant_style == 'dropdown'` vs option value `dropdowns` (`snippets/variant-main-picker.liquid:62`); gallery does not pass `block`/`selected_product`/`first_3d_model` to `product-media`; `product-form.js` appends `sections` inside a `forEach`; `buy-buttons` form id per section (duplicates in cards); standalone `quantity` not wired; standalone `accelerated-checkout` renders no markup; sticky bar uses global `product` (`sections/product-information.liquid:32-53`); `product-inventory` uses `closest.product` except `data-product-id="{{ product.id }}"` (`:58`) |
| JS | 11 components create their `AbortController` once as a field and never re-arm it after reconnect: `accordion-custom`, `media-gallery`, `product-form`, `drag-zoom-wrapper`, `deferred-media` and `product-model` (`assets/media.js`), `price-per-item`, `predictive-search`, `quick-add-component`, `quick-add-dialog`, `sticky-add-to-cart` (copy `disclosures-summary-fit` instead); 4 unguarded `customElements.define` (`product-custom-property`, `gift-card-recipient-form`, `layered-slideshow`, `product-hotspot`); `section-renderer` ignores HTTP status |
| Platform conformance | `sections/header.liquid` has 42 settings (> 40 lint threshold); `<main>` lacks `tabindex="-1"`; JSON ids use `-`/`_` though docs say alphanumeric; `featured-product` has no `@app` |
| Orphans | `_collection-image`, `_collection-info`, `_content`, `_footer-social-icons`, `_inline-text`, `_social-link`; `featured-collection` has `presets: []` and no reference |
| Meta | `<meta name="theme-color">` empty; both layouts emit markup before `meta charset` (`layout/theme.liquid:10-18`: favicon, view-transition; password: stylesheets, favicon, view-transition); `layout/password.liquid` has an unused `no-js` class |

## 20. Platform references (shopify.dev)

| Key | URL |
|---|---|
| BLK / TBS / TBT / TBST / TBD | …/themes/architecture/blocks · …/blocks/theme-blocks/schema · …/targeting · …/static-blocks · …/dynamic-sources |
| AB / AIB | …/blocks/app-blocks · …/blocks/ai-generated-theme-blocks |
| CF / CMT | https://shopify.dev/docs/api/liquid/tags/content_for · …/tags/comment (inline comments are `{% # … %}`) |
| SS / SG | …/architecture/sections/section-schema · …/architecture/section-groups |
| JT / ALT | …/architecture/templates/json-templates · …/templates/alternate-templates |
| IS / SET / DS | …/architecture/settings/input-settings · …/architecture/settings · …/settings/dynamic-sources |
| SLF / STLF / LOC | …/architecture/locales/schema-locale-files · …/storefront-locale-files · …/architecture/locales |
| LIM | …/architecture/limits |
| JSCSS / SUB / LCR | …/best-practices/javascript-and-stylesheet-tags · …/performance/stylesheet-subsetting · …/performance/load-critical-resources-before-content-for-header |
| PERF / RIMG / LCP / TAP | …/best-practices/performance · …/use-responsive-images · …/never-lazy-load-lcp-image · …/increase-tap-target-size |
| A11Y / EDI / PINS | …/best-practices/accessibility · …/editor/integrate-sections-and-blocks · …/editor/preview-inspector |
| TC / LDOC | …/themes/tools/theme-check/checks · …/tools/liquid-doc |
| SRA | https://shopify.dev/docs/api/ajax/section-rendering |
| UPD | …/themes/store/success/updates |

`…` = `https://shopify.dev/docs/storefronts/themes`. Docs inconsistencies (checked 2026-10-10): `closest.*` docs say "accessible exclusively via theme settings", yet Horizon uses it in Liquid; Theme Check examples use `context.product` (use `closest.*`); `UnsupportedDocTag` says `{% doc %}` is for snippets while LiquidDoc covers blocks (use it in snippets and blocks, not sections). Undocumented (UNVERIFIED): cross-scope `visible_if` (works in Horizon); block-level `"disabled": true`; `"alpha": true` on color settings (Horizon uses it).

## Appendix A. Code templates

Validated with `validate_theme` in a scratch copy of stock 4.2.0 together with the locale keys in A.4 (2026-10-10). Skeletons: rename `cx-example*`, delete what the component does not need, keep the conventions.

### A.1 Leaf theme block — `blocks/cx-example.liquid`

Shows: `{% doc %}`, `block_settings`, `contrast-override` + conditional `color-custom-{{ block.id }}`, `"tag": null` + single root with `block.shopify_attributes`, `spacing-style`, lightweight preset class, one `{% stylesheet %}`, stock `t:` keys, `t:cx.*` keys, preset with category.

```liquid
{%- doc -%}
  Renders a heading and rich text with Horizon color and spacing settings.
  @example
  {% content_for 'block', type: 'cx-example', id: 'example' %}
{%- enddoc -%}

{%- liquid
  assign block_settings = block.settings
  assign has_custom_color = false
  if block_settings.background_color != blank or block_settings.text_color != blank
    assign has_custom_color = true
  endif
-%}
{% render 'contrast-override', background_color: block_settings.background_color, text_color: block_settings.text_color, section_id: block.id %}

<div
  class="cx-example spacing-style{% if has_custom_color %} color-custom-{{ block.id }}{% endif %}"
  style="{% render 'spacing-style', settings: block_settings %}"
  {{ block.shopify_attributes }}
>
  {%- if block_settings.heading != blank -%}
    <p class="cx-example__heading {{ block_settings.heading_preset }}">{{ block_settings.heading }}</p>
  {%- endif -%}
  {%- if block_settings.text != blank -%}
    <div class="cx-example__text rte">{{ block_settings.text }}</div>
  {%- endif -%}
</div>

{% stylesheet %}
  .cx-example { display: flex; flex-direction: column; gap: var(--gap-xs); min-width: 0; }
  .cx-example__heading { margin: 0; }
{% endstylesheet %}

{% schema %}
{
  "name": "t:cx.example.name",
  "tag": null,
  "settings": [
    { "type": "inline_richtext", "id": "heading", "label": "t:cx.example.heading", "default": "t:cx.example.heading_default" },
    { "type": "select", "id": "heading_preset", "label": "t:settings.preset", "default": "h4",
      "options": [
        { "value": "paragraph", "label": "t:options.paragraph" },
        { "value": "h3", "label": "t:options.h3" },
        { "value": "h4", "label": "t:options.h4" }
      ] },
    { "type": "richtext", "id": "text", "label": "t:settings.text" },
    { "type": "header", "content": "t:content.appearance" },
    { "type": "color", "id": "background_color", "label": "t:settings.background_color", "placeholder": "t:settings.default" },
    { "type": "color", "id": "text_color", "label": "t:settings.text_color", "placeholder": "t:settings.default" },
    { "type": "header", "content": "t:content.padding" },
    { "type": "range", "id": "padding-block-start", "label": "t:settings.top", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 0 },
    { "type": "range", "id": "padding-block-end", "label": "t:settings.bottom", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 0 }
  ],
  "presets": [{ "name": "t:cx.example.name", "category": "t:categories.basic" }]
}
{% endschema %}
```

### A.2 Interactive block + module — `blocks/cx-countdown.liquid`, `assets/cx-countdown.js`

Shows: server-rendered fallback, gated module `<script>` in the block, `*-component` root, optional ref, storefront string via `| t`, cleanup in `disconnectedCallback`, idempotent restart in `updatedCallback`, guarded `customElements.define`, no import-map entry.

```liquid
{%- doc -%}
  Renders a countdown to a merchant-set end time; assets/cx-countdown.js fills in the remaining time.
  @example
  {% content_for 'block', type: 'cx-countdown', id: 'countdown' %}
{%- enddoc -%}

{%- assign block_settings = block.settings -%}
{%- if block_settings.end_time != blank -%}
  <script src="{{ 'cx-countdown.js' | asset_url }}" type="module" fetchpriority="low"></script>
{%- endif -%}

<cx-countdown-component
  class="cx-countdown spacing-style {{ block_settings.type_preset }}"
  style="{% render 'spacing-style', settings: block_settings %}"
  data-end="{{ block_settings.end_time | escape }}"
  {{ block.shopify_attributes }}
>
  {%- if block_settings.end_time != blank -%}
    <span class="cx-countdown__value" role="timer" aria-label="{{ 'cx.countdown.label' | t }}" ref="value">--:--:--</span>
    {%- if block_settings.expired_text != blank -%}
      <span class="cx-countdown__expired">{{ block_settings.expired_text | escape }}</span>
    {%- endif -%}
  {%- endif -%}
</cx-countdown-component>

{% stylesheet %}
  .cx-countdown { display: block; min-width: 0; }
  .cx-countdown__value { font-variant-numeric: tabular-nums; }
  .cx-countdown__expired, .cx-countdown[data-expired] .cx-countdown__value { display: none; }
  .cx-countdown[data-expired] .cx-countdown__expired { display: inline; }
{% endstylesheet %}

{% schema %}
{
  "name": "t:cx.countdown.name",
  "tag": null,
  "settings": [
    { "type": "text", "id": "end_time", "label": "t:cx.countdown.end_time", "info": "t:cx.countdown.end_time_info" },
    { "type": "text", "id": "expired_text", "label": "t:cx.countdown.expired_text" },
    { "type": "select", "id": "type_preset", "label": "t:settings.preset", "default": "h4",
      "options": [
        { "value": "paragraph", "label": "t:options.paragraph" },
        { "value": "h3", "label": "t:options.h3" },
        { "value": "h4", "label": "t:options.h4" }
      ] },
    { "type": "header", "content": "t:content.padding" },
    { "type": "range", "id": "padding-block-start", "label": "t:settings.top", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 0 },
    { "type": "range", "id": "padding-block-end", "label": "t:settings.bottom", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 0 }
  ],
  "presets": [{ "name": "t:cx.countdown.name", "category": "t:categories.basic" }]
}
{% endschema %}
```

```js
import { Component } from '@theme/component';

/** @typedef {{ value?: HTMLElement }} CountdownRefs */

/** @extends {Component<CountdownRefs>} */
class CxCountdownComponent extends Component {
  /** @type {number | undefined} */
  #intervalId;

  connectedCallback() {
    super.connectedCallback();
    this.#start();
  }

  updatedCallback() {
    super.updatedCallback();
    this.#start();
  }

  disconnectedCallback() {
    super.disconnectedCallback();
    this.#stop();
  }

  #start() {
    this.#stop();
    if (Number.isNaN(this.#endTime)) return;
    this.#tick();
    this.#intervalId = window.setInterval(() => this.#tick(), 1000);
  }

  #stop() {
    if (this.#intervalId === undefined) return;
    window.clearInterval(this.#intervalId);
    this.#intervalId = undefined;
  }

  get #endTime() {
    return Date.parse(this.dataset.end ?? '');
  }

  #tick() {
    const remaining = Math.max(0, this.#endTime - Date.now());
    const total = Math.floor(remaining / 1000);
    const parts = [Math.floor(total / 3600), Math.floor((total % 3600) / 60), total % 60];
    const { value } = this.refs; // destructure at use time; refs are replaced on refresh
    if (value) value.textContent = parts.map((n) => String(n).padStart(2, '0')).join(':');
    if (remaining === 0) {
      this.#stop();
      this.toggleAttribute('data-expired', true);
    }
  }
}

if (!customElements.get('cx-countdown-component')) {
  customElements.define('cx-countdown-component', CxCountdownComponent);
}
```

If this block can first appear through a Section Rendering morph, the module script inside it will not execute (§12.1 step 4): load the module from a file present on full page load.

### A.3 Section — `sections/cx-example-section.liquid`

Only when `section` + blocks cannot express the need (L7). Shows: contract comment, the §5.2 skeleton, a static `group` header with matching preset entry (`"static": true`, key = id, not in `block_order`), a dynamic area accepting `@theme`, `@app`, `_divider`, `disabled_on` header, stock setting IDs.

```liquid
{% comment %}
  Section contract.
  Renders a band with a fixed header group and a merchant-managed area of theme blocks.
  @example
  <!-- Theme editor: Add section > Example section. JSON: "type": "cx-example-section" -->
{% endcomment %}

{%- liquid
  assign section_settings = section.settings
  assign has_custom_color = false
  if section_settings.background_color != blank
    assign has_custom_color = true
    render 'contrast-override', background_color: section_settings.background_color, section_id: section.id
  endif
-%}

<div class="section-background{% if has_custom_color %} color-custom-{{ section.id }}{% endif %}"></div>
<div
  class="section section--{{ section_settings.section_width }} cx-example-section spacing-style gap-style{% if has_custom_color %} color-custom-{{ section.id }}{% endif %}"
  style="{% render 'spacing-style', settings: section_settings %}{% render 'gap-style', value: section_settings.gap %}"
>
  <div class="cx-example-section__header">
    {% content_for 'block', type: 'group', id: 'static-header' %}
  </div>
  <div class="cx-example-section__content">
    {% content_for 'blocks' %}
  </div>
</div>

{% stylesheet %}
  .cx-example-section { row-gap: var(--gap); }
  .cx-example-section__header, .cx-example-section__content { min-width: 0; }
  .cx-example-section__content { display: flex; flex-wrap: wrap; gap: var(--gap); }
{% endstylesheet %}

{% schema %}
{
  "name": "t:cx.example_section.name",
  "disabled_on": { "groups": ["header"] },
  "blocks": [{ "type": "@theme" }, { "type": "@app" }, { "type": "_divider" }],
  "settings": [
    { "type": "select", "id": "section_width", "label": "t:settings.width", "default": "page-width",
      "options": [
        { "value": "page-width", "label": "t:options.page" },
        { "value": "full-width", "label": "t:options.full" }
      ] },
    { "type": "range", "id": "gap", "label": "t:settings.gap", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 24 },
    { "type": "header", "content": "t:content.appearance" },
    { "type": "color", "id": "background_color", "label": "t:settings.background_color", "placeholder": "t:settings.default" },
    { "type": "header", "content": "t:content.padding" },
    { "type": "range", "id": "padding-block-start", "label": "t:settings.top", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 40 },
    { "type": "range", "id": "padding-block-end", "label": "t:settings.bottom", "min": 0, "max": 100, "step": 1, "unit": "px", "default": 40 }
  ],
  "presets": [
    {
      "name": "t:cx.example_section.name",
      "category": "t:categories.layout",
      "blocks": {
        "static-header": {
          "type": "group", "static": true,
          "blocks": { "heading": { "type": "text", "settings": { "text": "t:html_defaults.new_arrivals_h2" } } },
          "block_order": ["heading"]
        },
        "example_1": { "type": "cx-example" }
      },
      "block_order": ["example_1"]
    }
  ]
}
{% endschema %}
```

### A.4 Locale additions

`locales/en.default.schema.json`, top-level `cx` object (keep the file's comments):

```json
"cx": {
  "example": { "name": "Example", "heading": "Heading", "heading_default": "Example heading" },
  "countdown": { "name": "Countdown", "end_time": "End time", "end_time_info": "ISO 8601, e.g. 2026-12-31T23:59:00-05:00", "expired_text": "Text after the countdown ends" },
  "example_section": { "name": "Example section" }
}
```

`locales/en.default.json`: `"cx": { "countdown": { "label": "Time remaining" } }`

### A.5 Template JSON — `templates/page.example.json`

A stock section with dynamic blocks bound to `closest.page`, then the custom section with its static header (`"static": true`, absent from `block_order`) and dynamic custom blocks.

```json
{
  "sections": {
    "main": {
      "type": "main-page",
      "blocks": {
        "heading": { "type": "text", "settings": { "text": "<h1>{{ closest.page.title }}</h1>", "type_preset": "rte" } },
        "page-content": { "type": "page-content", "settings": {} }
      },
      "block_order": ["heading", "page-content"],
      "settings": {}
    },
    "example": {
      "type": "cx-example-section",
      "blocks": {
        "static-header": {
          "type": "group", "static": true, "settings": {},
          "blocks": { "heading": { "type": "text", "settings": { "text": "<h2>Why customers choose us</h2>" } } },
          "block_order": ["heading"]
        },
        "example_a1b2c3": { "type": "cx-example", "settings": { "heading": "Fast shipping", "heading_preset": "h4" } },
        "countdown_d4e5f6": { "type": "cx-countdown", "settings": { "end_time": "2026-12-31T23:59:00-05:00", "expired_text": "The sale has ended." } }
      },
      "block_order": ["example_a1b2c3", "countdown_d4e5f6"],
      "settings": { "gap": 24 }
    }
  },
  "order": ["main", "example"]
}
```
