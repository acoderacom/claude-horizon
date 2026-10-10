# THEME.md — Horizon 4.2.0 technical specification

| Field | Value |
|---|---|
| Theme | Horizon by Shopify |
| **Version covered** | **4.2.0** — exactly this version, nothing else |
| Version source of truth | `config/settings_schema.json` → `[0].theme_version` (`"4.2.0"`, `config/settings_schema.json:2-9`) |
| Source snapshot | Shopify/horizon commit `5acd1b6` (2026-09-21, merge of PR #101 "Horizon release v4.2.0"; release commit `f9aef27`, 2026-09-18) |
| Spec written | 2026-10-10 |
| Scope | Stock Horizon 4.2.0 theme files. Paths are relative to the theme root (the folder that contains `config/settings_schema.json`); every `path:line` citation points at stock 4.2.0. |
| Audience | AI coding agents. Technical, terse, normative. |
| Status | Single source of truth for this theme. |

## 0. Version gate, labels, precedence

### 0.1 Version gate (run before any edit)

1. Read `config/settings_schema.json` `[0].theme_version`.
2. `== "4.2.0"`: this spec applies.
3. `!= "4.2.0"`: every `path:line`, count, setting ID and catalog entry below may be stale. Re-verify each fact you rely on against source before acting, then update this file (version table + affected sections) and the version line in `customizations.md`. Do not "fix" code to match a stale spec.
4. Release notes for the covered version: `release-notes.md` (4.2.0: RTL `dir` attribute, logical alignment values, cart bfcache fixes).

### 0.2 Labels used in this file

| Label | Meaning |
|---|---|
| `path:line` | Verified in stock 4.2.0 source. |
| PLATFORM | Shopify platform behavior, taken from shopify.dev (URL in §20), not provable from theme source. |
| UNVERIFIED | Not proven by source or official docs. Do not rely on it without testing. |
| MUST / MUST NOT | Hard rule. Violations break saved merchant data, the editor, or the storefront. |
| SHOULD | Default; deviate only with a written reason in the change log. |

### 0.3 Precedence

1. Project instructions (`CLAUDE.md` or equivalent user decisions) — workflow, store access, deployment.
2. This file — how the theme works and how to change it.
3. Shopify platform docs (§20) — anything this file does not cover.

### 0.4 Golden rules (summary of the whole spec)

1. **Horizon first.** Walk the customization ladder (§2) top-down. Stop at the first level that meets the need. Between L5 (stock edit) and L6 (new `cx-` theme block) the §2.1 rule decides: L6 unless the change is a small addition to the stock file that already does the job. Custom code is allowed only with a written justification (§2.3).
2. **Theme blocks are the unit of work.** Compose with existing sections + theme blocks + presets. When code is unavoidable, the default artifact is a new theme block, not a new section (§2.2, §6).
3. **Saved data is a contract.** MUST NOT rename, remove or repurpose (give a new meaning to): section/block types (file names), setting IDs, setting `type`s, select/radio option values, static block `id`s, preset keys referenced by templates. Saved JSON points at them by name (§3.8).
4. **JSON data files are owned by the theme editor.** `templates/*.json`, `sections/*-group.json`, `config/settings_data.json` are rewritten on every editor save. Read the current copy before writing; merge, never overwrite (§4.1).
5. **Static block contract.** `{% content_for 'block', type: T, id: I %}` ⇄ JSON key `I` with `"type": T, "static": true`, never in `block_order` (§3.4).
6. **Private blocks (`_name`)** appear only where a parent schema lists them explicitly; `@theme` never includes them (§3.2).
7. **CSS:** one `{% stylesheet %}` per file, no Liquid inside it; instance values via inline custom properties or `{% style %}`; use theme tokens; logical properties; render style-owner snippets wherever their markup is reused (§9).
8. **Color:** no color schemes in 4.2.0. Custom colors go through `snippets/contrast-override.liquid` + `color-custom-<id>` class (§10).
9. **JS:** native HTML first; interactive components extend `Component` (`@theme/component`), use `ref`/`on:` attributes, clean up on disconnect; no new runtime dependencies (§12).
10. **Strings:** storefront UI text → `locales/en.default.json` + `t` filter; editor labels → `locales/en.default.schema.json` + `t:` keys; shop content → settings/metafields (§14).
11. **Validate** every changed file with `shopify-dev-mcp` `validate_theme`; compare against the stock baseline (§17.2) before "fixing" anything.
12. **Record** every stock-file edit and every new file with its level, checks and, for L4–L8, its justification (§2.4).
13. **Custom file prefix `cx-`** for every new block, section, snippet, asset, CSS class, custom element and locale namespace (§2.5).

## 1. Architecture at a glance

```
layout/theme.liquid                         document shell (password page: layout/password.liquid)
├─ <head>  meta-tags, stylesheets (base.css), card-hover-effect-styles, fonts, scripts (import map),
│          theme-styles-variables (tokens), color-palette (color tokens), theme-editor (editor only),
│          {{ content_for_header }}
├─ chat-drawer     (body start, before #header-group)
├─ #header-group   {% sections 'header-group' %}  → sections/header-group.json (header-announcements, header)
├─ <main id="MainContent">  {{ content_for_layout }} → templates/<type>[.<suffix>].json → sections → theme blocks
├─ <footer>        {% sections 'footer-group' %}  → sections/footer-group.json (footer, footer-utilities)
└─ overlays        cart-drawer (outputs only if cart_type=drawer and template ≠ cart), theme-drawer, search-modal,
                   quick-add-modal (if quick_add or mobile_quick_add)
```

Render units, top-down: **layout → section group / template (JSON) → section (`sections/*.liquid`) → theme blocks (`blocks/*.liquid`, nestable up to 8 levels) → snippets (`snippets/*.liquid`, no schema)**.

| Folder | Files (4.2.0) | Role | Contract |
|---|---|---|---|
| `layout/` | 2 | Document shells; own global markup, shared CSS/JS, section groups, `content_for_header` / `content_for_layout` | §7 |
| `templates/` | 13 (12 JSON + `gift_card.liquid`) | Page composition. Editor-owned JSON. Keep IDs stable. | §4 |
| `sections/` | 42 (40 `.liquid` + 2 group `.json`) | Merchant-addable page regions and required page sections; compose static + dynamic theme blocks | §5 |
| `blocks/` | 95 (44 public, 51 private `_*`) | Nestable theme components with own settings and targeting rules | §3, §6 |
| `snippets/` | 145 | Reusable Liquid via `{% render %}`; no schema; 42 are `*-styles` (36 own a `{% stylesheet %}`, 2 emit per-instance `{% style %}`, 4 emit inline custom properties for `style=""`) | §8 |
| `assets/` | 125 (81 JS, 3 CSS, 35 SVG, 6 dev files) | Flat. No subfolders. No build step. | §9, §12 |
| `config/` | 2 | `settings_schema.json` (global setting definitions), `settings_data.json` (saved values, editor-owned) | §11 |
| `locales/` | 57 (34 storefront, 23 schema) | Runtime UI strings + editor labels | §14 |

There is **no `schemas/` folder** in this package. Section and block schemas are edited directly as the JSON inside `{% schema %}…{% endschema %}`; `config/settings_schema.json` is edited directly. If a later package or source checkout ships `schemas/` (a Liquid file with an import marker and matching source under `schemas/`), the embedded JSON is generated output: edit the schema source, run the generator documented by that checkout's contributor instructions, and verify that source and embedded JSON agree. `config/settings_schema.json` stays hand-edited either way.

Search for an existing section, block, snippet, setting group, style utility or JS component before adding one. Extend the contract that already owns the behavior.

Read every file involved in a change before editing it. Verify every example, API, setting ID and class name you use against current theme source and platform behavior; this file is a map, the source is the territory.

## 2. How to customize: the ladder

The user decision for this theme: **use Horizon as much as possible; custom code is allowed, but every custom change must state why Horizon could not do it.**

### 2.1 The ladder (walk top-down, stop at the first level that works; choose between L5 and L6 by the rule below, not by order)

| Level | Mechanism | Writes code? | Survives a Horizon update untouched? | Typical use |
|---|---|---|---|---|
| L0 | Store data / admin: products, collections, metafields + definitions, metaobjects, menus, pages, Search & Discovery filters, apps | No | Yes | Content, navigation, filter lists, structured data for dynamic sources |
| L1 | Global theme settings (`config/settings_data.json` `current`, via editor) | No | Yes | Palette, fonts, type scale, buttons, inputs, cards, badges, cart type, page width, animations |
| L2 | Section/block settings on existing instances, incl. dynamic sources (`{{ closest.product.metafields.ns.key }}`) | No | Yes | Copy, media, layout, colors, spacing of one band or block |
| L3 | Composition in JSON: add/remove/reorder/nest existing sections and theme blocks; pick presets; alternate templates (`templates/<type>.<suffix>.json`) | No (JSON data only) | Yes (`templates/*.json` and `settings_data.json` are excluded from the update check — PLATFORM, §20 [UPD]) | New page layouts, new bands from `section` presets + `group`/`text`/`image`/`button`/… |
| L4 | Merchant escape hatch: `custom-liquid` block / section (Liquid stored as a setting value) | Liquid in data | Yes | Small, content-level Liquid (a metafield print, a one-off embed). Not for features: no schema, no CSS file, no review trail, ≤50 KB; no `layout`/`section`/`javascript`/`stylesheet`/`schema` tags, no `content_for_header`/`content_for_layout`/`content_for_index`, no `settings` (PLATFORM [IS]); whether `closest.*` resolves inside it is UNVERIFIED |
| L5 | Extend a stock file that already owns the responsibility (add a setting, modifier class, small markup change) | Yes — **stock edit** | No → update becomes manual merge | Behavior change inside an existing component used everywhere |
| L6 | New theme block `blocks/cx-*.liquid` | Yes — new file | No (any added/changed file blocks automatic updates; UNVERIFIED for added files, assume yes) | New component that no stock block owns |
| L7 | New section `sections/cx-*.liquid` | Yes — new file | No | Page-level region with its own data loop, grid or JS that `section` + blocks cannot express |
| L8 | Supporting code: `snippets/cx-*`, `assets/cx-*.js/.css`, locale keys, global CSS hook | Yes | No | Only in service of L5–L7 |

Rules:
- L0–L3 are always preferred. They keep Shopify's automatic theme updates available: "A theme is automatically updated only if all theme files, except `settings_data.json` and any JSON files in the `/templates` directory, are in their original state" (PLATFORM [UPD]). Note: `sections/*-group.json` are not in that exception list.
- Between L5 and L6: choose L5 only when the stock file already owns the responsibility and the change is small and additive. Choose L6 in every other case, including when L5 would be possible but large. Reason: a manual Horizon update installs the new version as an unpublished copy (PLATFORM [UPD]); a `cx-` file carries over by copying it, while every stock edit must be re-merged by hand into the new version of that file. Never copy a stock block to a `cx-` file and edit the copy unless the stock block must stay unchanged elsewhere; if you do, say so in the justification.
- L7 only when the need is page-level. Before writing a section, prove that `sections/section.liquid` (13 presets, accepts `@theme`, `@app`, `_divider`) plus blocks cannot do it.
- L4 is not a substitute for L6. A `custom-liquid` setting that grows beyond a few lines, needs CSS, settings or JS becomes a `cx-` block.

### 2.2 Theme blocks first

When code is required, the default artifact is a **public theme block** (`blocks/cx-name.liquid` with a preset):
- It is reusable in every container that accepts `@theme`: `section`, `_blocks`, `group`, `hero`, `main-page`, `main-404`, `password`, `product-list`, `collection-list`, `main-collection-list`, `main-blog`, `main-blog-post`, `main-cart`, `product-recommendations` (section), `product-information` / `featured-product-information` (section area and `_product-details`), `popup-link`, `_card`, `_slide`, `_layered-slide`, `_accordion-row`, `_content-without-appearance`, `featured-collection`, the `product-recommendations` block, `_product-list-content` (§3.6). (Also accepted but useless: `predictive-search` never renders its block area; orphans `_content`, `_collection-info`, §6.4.)
- Merchants can add, remove, reorder and configure it without further code.
- No stock file changes: stock containers already accept `@theme`, so the block appears in them as soon as it has a preset. Theme blocks are reusable and nestable; section-defined blocks "only work in the section they're defined within" (PLATFORM [BLK]).
- Make it private (`_cx-name`) only when it must be restricted to one parent; then list it explicitly in that parent's `blocks` array (a stock edit if the parent is stock).

### 2.3 Justification record (required for L4–L8)

Write this in the change log entry (§2.4) before writing code:

```
Need:            <what the merchant/user needs, observable behavior>
Stock checked:   <sections/blocks/settings/presets evaluated, with file names>
Why not stock:   <the specific missing capability, per candidate>
Level chosen:    L5 | L6 | L7 | L8  (+ L4 if custom-liquid)
Files:           <new or edited files>
Settings/IDs:    <new setting IDs, block types, static ids, locale keys>
Upgrade impact:  <stock files edited (need re-merge on Horizon update) / new files (carry over)>
```

### 2.4 Change log contract

Every stock-file edit and every new file MUST be recorded in the project change log `customizations.md`, in the same task, using its entry format:
- file path; new vs. stock-modified (the log section it goes in);
- level (§2.1);
- what it does / what changed (quote one-line stock edits verbatim);
- settings touched (for `config/settings_data.json`: setting ID, stock value, new value);
- for L4–L8, the §2.3 justification. A change that spans several files carries it once, in the main file's entry; the other entries point to it;
- checks run (`validate_theme`, preview) and checks that could not be run (§17.1);
- how to undo.
The change log is the only record of how this theme differs from stock Horizon 4.2.0, apart from merchant edits made in the theme editor (those live in the theme's JSON files and are not logged).

### 2.5 Naming for custom code

| Artifact | Pattern | Example |
|---|---|---|
| Public block | `blocks/cx-<name>.liquid` | `blocks/cx-countdown.liquid` |
| Private block | `blocks/_cx-<name>.liquid` | `blocks/_cx-countdown-unit.liquid` |
| Section | `sections/cx-<name>.liquid` | `sections/cx-store-locator.liquid` |
| Snippet | `snippets/cx-<name>.liquid`; style owner `snippets/cx-<name>-styles.liquid` | `snippets/cx-badge-styles.liquid` |
| JS module | `assets/cx-<name>.js`; import-map key `@theme/cx-<name>` only if imported | `assets/cx-countdown.js` |
| Custom element | `cx-<name>-component` | `<cx-countdown-component>` |
| CSS classes / vars | BEM root = file name: `.cx-<name>`, `.cx-<name>__el`, `.cx-<name>--mod`; vars `--cx-<name>-*` | `.cx-countdown__value` |
| Schema locale keys | `t:cx.<name>.<key>` in `locales/en.default.schema.json` (top-level `cx` object) | `t:cx.countdown.end_time` |
| Storefront locale keys | `cx.<name>.<key>` in `locales/en.default.json` (top-level `cx` object) | `'cx.countdown.days' \| t` |
| DOM ids | `Cx<Name>-{{ block.id }}` / `-{{ section.id }}` | `CxCountdown-{{ block.id }}` |
| Alternate template | `templates/<type>.<suffix>.json` with a plain, readable suffix: the suffix is the template name merchants pick in the admin (PLATFORM [ALT]); `cx-` not required (JSON data, L3) | `templates/page.faq.json` |

Why: collision-proof against future Horizon files (do not use `custom-`: stock has `custom-liquid` and `.button-custom`, `.custom-typography`), greppable for upgrade merges, one place in each locale file. Reuse stock `t:` keys (`t:settings.*`, `t:options.*`, `t:content.*`, `t:categories.*`, `t:info.*`) for standard settings instead of duplicating them.

### 2.6 Hard "do not" list

- MUST NOT rename/remove/re-type: block or section files, setting IDs, option values, static block ids, preset block keys used in templates. Equivalent to deleting saved merchant data (PLATFORM [UPD]: "changing the ID of a setting is equivalent to removing the current setting and adding a new one with the default value").
- MUST NOT edit `assets/base.css` for project styling (global CSS route: §9.9).
- MUST NOT put Liquid inside `{% stylesheet %}` / `{% javascript %}` (not rendered; Theme Check error).
- MUST NOT add runtime JS dependencies, bundlers or subfolders in `assets/`.
- MUST NOT add `custom_css` keys to JSON (platform/merchant-owned).
- MUST NOT branch on `block.id` / `section.id` values in Liquid; use them only for scoping (CSS classes, DOM ids). Theme Check `BlockIdUsage` flags `block.id` in `if`/`elsif`/`unless`/`case`; the same rule applies to `section.id`, which no check enforces.
- MUST NOT reuse orphan private blocks (§6.4) without wiring them into a parent first.
- MUST NOT modify or wrap `{{ content_for_header }}`.

## 3. Theme blocks: the system

### 3.1 Kinds of blocks

| Kind | Where | Horizon 4.2.0 |
|---|---|---|
| Theme block | `blocks/<type>.liquid`; `type` = file name | All 95 blocks. Reusable, nestable, can be static. |
| Section-defined (local) block | inside one section's schema `blocks` with `name`/`settings` | **None.** A section uses either local blocks or theme blocks, never both; theme blocks cannot define local blocks (PLATFORM [BLK][TBS]). Do not introduce local blocks. |
| App block | from installed apps, rendered where a parent accepts `@app` | Accepted by most content containers (§3.6). |

### 3.2 Targeting: what a parent accepts (schema `blocks` array)

| Entry | Meaning |
|---|---|
| `{"type": "@theme"}` | Every **public** theme block that has `presets` (appears in the picker). Never includes `_*` private blocks. |
| `{"type": "@app"}` | App blocks. `limit` is not allowed on `@app`. |
| `{"type": "<name>"}` without `@theme` | Only the listed types. |
| `@theme` + explicit types | Listed types are shown first ("recommended"), the rest under "Show all". Private types listed here become available. |

- Private block (`_` prefix): available only where a parent lists it explicitly (e.g. `{"type": "_divider"}` in `sections/section.liquid:16-26`).
- Picker visibility needs `presets`. Public blocks **without** presets (`accelerated-checkout`, `add-to-cart`, `contact-form-submit-button`, `featured-collection`, `filters`, `page-content`) can be placed only statically or by JSON (stock `templates/page.json` places `page-content` as a dynamic block via JSON).
- Static-only children do not need to be listed in the parent's `blocks` array (stock: `contact-form` has no `blocks` key and renders static `contact-form-submit-button`, `blocks/contact-form.liquid:2`; `carousel` section has no `blocks` key).
- Theme Check `ValidBlockTarget`: every preset child must be covered by the parent's root `blocks` (`@theme` or explicit; private must be explicit).

### 3.3 Rendering tags

```liquid
{% content_for 'blocks' %}  {% # dynamic area: merchant-managed children in saved order %}
{% content_for 'blocks', closest.product: product %}  {% # + resource context for all children %}
{% content_for 'block', type: 'group', id: 'static-header' %}  {% # one static block %}
{% content_for 'block', type: '_product-card', id: 'static-product-card',
   closest.product: product, product_view_context: 'collection' %}  {% # static + context + keyword arg %}
```

- Render the dynamic area **once per file**. If several branches need it, `capture` it once (`blocks/group.liquid:1-3`, `sections/section.liquid:1-3`).
- Stock calls `content_for` only in sections and blocks. To place block output inside a snippet, `capture` it in the section/block and pass the HTML as a param (`sections/product-information.liquid:5-19` → `snippets/product-information-content.liquid` params `media_gallery`, `product_details`, `additional_blocks`). Calling `content_for` inside a snippet: UNVERIFIED (the `content_for` lines in that snippet are a `{% doc %}` example, not executed code).
- Keyword args on a static block become plain variables inside the block (`variant`, `text`, `index`, …). Declare them with `@param` in the block's `{% doc %}`. Filters on `content_for` args are silently ignored → `assign` first (Theme Check `UnsupportedFilterArguments`).
- Reserved LiquidDoc param names in blocks (collide with `content_for` args): `attributes, block, blocks, class, context, id, inherit, resource, resources, schema, section, sections, settings, snippet, snippets, src, style, styles, template, templates, type`.
- A block cannot receive caller-local variables except static-block keyword args; it sees `block`, the rendering `section`, and globals.

### 3.4 Static vs dynamic blocks

| | Static (`content_for 'block'`) | Dynamic (`content_for 'blocks'`) |
|---|---|---|
| Editor | hide + configure; no reorder/remove/duplicate | add/remove/reorder/duplicate/configure |
| Liquid | may render conditionally, in a loop, or several times | rendered in saved order |
| JSON | `"static": true`; key = Liquid `id`; **never in `block_order`** | listed in `block_order` |
| Presets | `"static": true` + key = `id`; absent statics are still created with defaults | must be in preset to be added with it |
| Limits | not counted toward `max_blocks` | counted (≤50 per section) |

Static block contract (MUST):
1. Liquid `{% content_for 'block', type: 'T', id: 'I' %}` ⇄ JSON `"I": { "type": "T", "static": true, … }` in the same parent.
2. `id` is a string literal, unique within the immediate parent; Shopify never generates static ids.
3. Never rename `I` or change `T` once merchants have saved data.

Horizon patterns built on static blocks:
- **Fixed structure:** product page `media-gallery` + `product-details` (`sections/product-information.liquid:6-17`), header `header-logo` (`sections/header.liquid:82`).
- **Repeated static block (one settings record, N renders):** a static block inside a `for` loop with per-item context → one merchant-configured card applied to every item: `_product-card#static-product-card` (`sections/product-list.liquid:37-42`), `_product-card#product-card` (`sections/main-collection.liquid:70-75`), `_collection-card#static-collection-card` (`sections/collection-list.liquid:56`), `_blog-post-card#static-blog-post-card` (`sections/main-blog.liquid:151-159`), `_collection-link#link` (`sections/collection-links.liquid:56-62`).
- **Render variants:** the same static id rendered several times with different keyword args → one settings record, several UIs: `_header-menu#header-menu` default / `variant: 'mobile'` / `variant: 'navigation_bar'` (`sections/header.liquid:86-94`).
- **Exclusive branches:** same id in mutually exclusive `if`/`else` branches (`sections/product-list.liquid:37-42` populated loop vs `:55` placeholder loop).
- **Template copy:** same id rendered twice on every request, once inside `<template id="empty-cart-template">` with keyword arg `force_empty: true` and once live (`sections/main-cart.liquid:26,29` vs `:45,49`).
- **Context nulling:** `closest.product: null` stops cards on a product page from inheriting the template product (`sections/product-list.liquid:55`, `sections/carousel.liquid:27`).

`validate_theme` flags every reuse of one static id within a file as "The id 'X' is already being used by another static block": `sections/header.liquid` (`header-menu`), `sections/main-cart.liquid` (`cart-page-title`, `cart-page-items`), `sections/product-list.liquid` (`static-product-card`), `sections/collection-links.liquid` (`link`) (§17.2 baseline). A single render inside a loop and context nulling are not flagged. The flagged cases are intentional in stock; reuse them only with the same intent (render variants, template copy, exclusive branches, repeated rendering).

### 3.5 Context: `closest.*`

- `closest.product | collection | article | blog | page | metaobject.<type>` resolves (PLATFORM [TBD]): nearest `content_for … closest.X:` setter → nearest ancestor block resource setting of that type (incl. own) → section resource setting → template resource.
- Horizon uses `closest.*` directly in Liquid (`blocks/buy-buttons.liquid:4`, `blocks/price.liquid:5`, `sections/product-information.liquid:2`). Official docs disagree on whether this is supported outside settings; treat Horizon usage as the working contract.
- Setting values/presets can bind dynamic sources: `"text": "<h1>{{ closest.product.title }}</h1>"` (`templates/product.json:110`), `"product": "{{ closest.product }}"` (`templates/product.json:393`), `"{{ closest.collection.title }}"` (`templates/collection.json:19`).
- Explicit passes in stock: `sections/product-information.liquid:9,17` (`closest.product: closest.product`), `sections/featured-product.liquid:46,50` (`closest.product: section.settings.product`), card loops (`closest.product: product`, `closest.collection: collection`, `closest.article: article`), `blocks/_product-card.liquid:10` and `blocks/_collection-card.liquid:12` (forward to children).
- Implicit: `featured-product-information` passes no `closest.product`; blocks resolve it from the section's `product` setting (PLATFORM). `featured-collection` block's `collection` setting becomes `closest.collection` for its children.
- Dynamic-source limits: 100 per JSON template, 50 per single setting (PLATFORM [DS]).

### 3.6 Containers (who accepts what)

Blocks:

| Block | `@theme` | `@app` | Explicit (incl. private) |
|---|---|---|---|
| `group` | ✓ | ✓ | `_divider` |
| `popup-link`, `featured-collection`, `_accordion-row` | ✓ | ✓ | — |
| `_card` | ✓ | ✓ | `text`, `_heading`, `image`, `video`, `product-card`, `collection-card` |
| `_slide`, `_layered-slide` | ✓ | ✓ | `_heading`, `button`, `text`, `group`, `image`, `video`, `icon`, `jumbo-text` |
| `_content-without-appearance` | ✓ | ✓ | `text`, `icon`, `image`, `button`, `video`, `group`, `spacer`, `_divider` |
| `_product-details` | ✓ | ✓ | product blocks + `_divider` (§13.1) |
| `_product-list-content` | ✓ | ✓ | `text`, `button`, `spacer`, `_divider`, `_product-list-text`, `_product-list-button` |
| `product-recommendations` (block) | ✓ | ✓ | — |
| `accordion` | ✗ | ✗ | `_accordion-row` only |
| `_carousel-content` | ✗ | ✗ | `_card` only |
| `_marquee` | ✗ | ✗ | `text`, `icon`, `_divider` |
| `collection-card`, `_collection-card` | ✗ | ✗ | `text`, `spacer`, `button`, `group`, `collection-title` |
| `product-card`, `_product-card`, `_product-card-group` | ✗ | ✓ | card list (§13.3) + `custom-liquid` |

Not listed above: orphan containers `_content` (`@theme`, `@app`, `_divider`), `_collection-info` (`@theme`, `@app`), `_footer-social-icons` (`_social-link`) — unreachable in 4.2.0 (§6.4).

Sections:

| Section | Accepts |
|---|---|
| `section`, `_blocks`, `main-page`, `main-404`, `password`, `product-list` | `@theme`, `@app`, `_divider` |
| `collection-list`, `main-collection-list`, `main-blog`, `product-recommendations` | `@theme`, `@app`, recommended `text, icon, image, button, video, group, spacer, _divider` |
| `main-cart` | same without `_divider` |
| `main-blog-post` | `@theme`, `@app`, `_blog-post-title`, `_blog-post-info-text`, `_blog-post-featured-image`, `_blog-post-content` |
| `product-information`, `featured-product-information` | `@theme`, `@app` (section-level area renders **after** the product grid, outside Quick Add) |
| `hero` | `@theme`, `text`, `button`, `logo`, `jumbo-text`, `spacer`, `group`, `_marquee` — **no `@app`** |
| `footer` | explicit: `_divider, @app, button, follow-on-shop, group, icon, image, menu, payment-icons, text, logo, jumbo-text, social-links, email-signup` — **no `@theme`** |
| `footer-utilities` | `footer-copyright, footer-policy-list, social-links` (`max_blocks: 3`) |
| `marquee` | `text, icon, logo, _divider` |
| `header-announcements` | `_announcement` |
| `slideshow` | `_slide` |
| `layered-slideshow` | `_layered-slide` (`max_blocks: 6`) |
| `product-hotspots` | `_hotspot-product` |
| `featured-blog-posts` | `_featured-blog-posts-title, _featured-blog-posts-card, _featured-blog-posts-image` |
| `predictive-search` | `@theme` in schema only; the section never calls `content_for 'blocks'` (SRA target of `snippets/search-modal.liquid`), so blocks added there do not render |
| static-only or none (no `blocks` key; `quick-order-list` has `[]`) | `carousel`, `collection-links`, `featured-product`, `header`, `media-with-content`, `main-collection`, `search-header`, `search-results`, `logo`, `divider`, `custom-liquid`, `quick-order-list`, `cart-drawer-section`, `password-footer`, `predictive-search-empty`, `section-rendering-product-card` |

Consequence: a new public `cx-` block with a preset is immediately available in every `@theme` container above. To put it in `footer`, `marquee`, `hero` (already `@theme`), `accordion` rows (`_accordion-row` accepts `@theme`), or cards, check the table; restricted parents need a stock edit (L5) to list it.

### 3.7 Nesting, limits, wrapper

- Nesting ≤ 8 levels below the section; ≤ 50 blocks per section; ≤ 300 files in `blocks/`; ≤ 25 sections per template / section group; ≤ 1,250 merchant-addable blocks per JSON template or section group (static blocks do not count); `blocks/*.liquid` ≤ 256 KB (PLATFORM [BLK][TBS][JT][LIM]).
- Block schema attributes allowed: `name` (≤25 chars after `t:` resolution), `settings`, `blocks`, `presets`, `tag`, `class`. (`limit`, `max_blocks`, `enabled_on`, `disabled_on` are section-only.)
- Wrapper: default `<div id="shopify-block-{id}" class="shopify-block {class}">` with editor attributes added by the platform. With `"tag": null` (86 of 95 stock blocks) there is no wrapper: the file MUST output one top-level element carrying `{{ block.shopify_attributes }}` exactly once. `class` has no effect with `tag: null`.
- 8 stock blocks have no `tag` key and rely on the platform wrapper: `_blog-post-card`, `_blog-post-content`, `_blog-post-info-text`, `_featured-blog-posts-title`, `icon`, `page`, `page-content`, `product-recommendations`. Six print no attributes; `icon` (`blocks/icon.liquid:18`) and `product-recommendations` (`blocks/product-recommendations.liquid:52`) also print `block.shopify_attributes` on an inner element (duplicate, §19). `product-custom-property` sets `"tag": "product-custom-property-component"` (custom-element wrapper) and also prints the attributes.
- New blocks: `"tag": null` + attributes on the single root element.

### 3.8 Saved-data contract

Saved merchant configuration (templates, section groups, `settings_data.json`) references code by name. Changes that silently drop or reset data:

| Change | Effect |
|---|---|
| Rename/remove a block or section file | Instances in JSON break (Theme Check `JSONMissingBlock`/`JSONMissingSection`); editor drops them |
| Rename/remove a setting ID or change its `type` | Saved value lost → default |
| Remove/rename a select/radio option value | Saved value invalid → default |
| Narrow a `range` (`min` up / `max` down) | Out-of-range values reset |
| Repurpose an existing setting ID, option value or block type (same name, new meaning) | Saved values apply with the wrong meaning; no error is raised |
| Rename a static block `id` | Its saved settings orphaned; new id starts at defaults |
| Remove a type from a parent's `blocks` | Existing children of that type become invalid |
| Add `enabled_on`/`disabled_on` restrictions | Existing placements become invalid |

Safe changes: add settings (with defaults), add options, add accepted block types, add presets, change labels/`info`, widen ranges.
When you rename or remove a section setting, grep `blocks/` and `snippets/` for `section.settings.<id>`: many private blocks read parent section settings directly (§6.5).

## 4. Templates and section groups (JSON)

### 4.1 Ownership rules

- `templates/*.json`, `sections/*-group.json`, `config/settings_data.json` start with an "auto-generated … may be overwritten" comment (`templates/product.json:1-9`). The theme editor rewrites them on save.
- Before editing one: read the **current** copy (the live theme is the source of truth, not a local copy), merge your change into it, keep every unrelated key, keep IDs.
- Parsers must accept a leading `/* … */` comment (all JSON templates, both section groups, `config/settings_data.json`, `locales/en.default*.json`) and `//` line comments (stock: only `locales/en.default.json` and `locales/en.default.schema.json`).
- Section and block IDs in JSON: stock uses semantic ids (`main`, `media-gallery`, `static-product-card`) and editor-generated ids (`hero_jVaWmY`, `group_icgrde`). Docs say "alphanumeric"; Horizon uses `-` and `_`. Safe charset: `[a-z0-9_-]`. New dynamic ids: `<type>_<6 random alphanumerics>`.

### 4.2 JSON template format

Shape only: `<…>` are placeholders and `//` notes are annotations, so this is not a file to copy. Appendix A.5 is a complete, validated template.

```jsonc
{
  "sections": {
    "<section_id>": {
      "type": "<sections/ file name>",
      "name": "t:names.x",                      // optional editor name
      "disabled": true,                          // optional: not rendered, still editable (unused in stock)
      "settings": { "<setting_id>": <value> },
      "blocks": {
        "<static_id>": { "type": "<block>", "static": true, "settings": {}, "blocks": {}, "block_order": [] },
        "<block_id>":  { "type": "<block>", "settings": {}, "blocks": { … }, "block_order": [ … ] }
      },
      "block_order": ["<block_id>"]              // dynamic children only
    }
  },
  "order": ["<section_id>"],
  "layout": "password"                           // optional; only templates/password.json uses it
}
```

- Nested blocks: each parent has its own `blocks` map + `block_order`.
- Values may be palette references `"{{ settings.color_palette.<slot> }}"`, dynamic sources `"{{ closest.product.title }}"`, raw colors `"#00000026"`, `rgba(...)`, `shopify://collections/all` links.
- Static blocks are always added with their container's preset using default setting values (PLATFORM [TBST]). A static block called in Liquid but absent from saved template JSON rendering with schema defaults: UNVERIFIED (not stated in docs; consistent with the preset rule).
- Limits: 25 sections per template, 50 blocks per section, 1,000 JSON templates, 512 KB per JSON template (PLATFORM [JT][LIM]).
- Alternate templates: `templates/<type>.<suffix>.json` (stock: `page.contact.json`). Merchants assign them per resource. `gift_card` must stay `.liquid`.

### 4.3 Stock templates

| Template | Sections (order) | Key static blocks |
|---|---|---|
| `index.json` | `hero` → `product-list` | product-list: `static-header` (`_product-list-content`), `static-product-card` (`_product-card` → gallery, title, price) |
| `product.json` | `product-information` (`main`) → `product-recommendations` | `media-gallery` (`_product-media-gallery`), `product-details` (`_product-details` → group(title text + price), `_divider`, `variant-picker`, `buy-buttons`(static `quantity`, `add-to-cart`, `accelerated-checkout`), text(description)); section-level dynamic `disclosures` |
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
| `gift_card.liquid` | own document (`{% layout none %}`), own import map | — |

Stock nesting depth in templates: max 3.

### 4.4 Section groups

- `sections/header-group.json` (`"type": "header"`) and `sections/footer-group.json` (`"type": "footer"`); same shape as templates + `type`, `name`; rendered by `{% sections 'header-group' %}` / `'footer-group'` (`layout/theme.liquid:42,164`).
- Header group (stock): `header-announcements` (one `_announcement`) → `header` (static `header-logo`, `header-menu`; `menu: main-menu`, `menu_style: featured_products`).
- Footer group (stock): `footer` (group(texts + `email-signup`), `menu` footer, `menu` main-menu; background `color1`) → `footer-utilities` (`footer-copyright`, `footer-policy-list`, `social-links`).
- Placement keys (`enabled_on` / `disabled_on`; never both):

| Restriction | Sections |
|---|---|
| `enabled_on.groups: [header]` | `header-announcements` |
| `enabled_on.groups: [footer]` | `footer`, `footer-utilities`, `logo` |
| `enabled_on.templates` | `main-collection` (`collection`), `quick-order-list` (`product`) |
| `disabled_on.groups: [header]` | `section`, `hero`, `featured-product`, `media-with-content`, `main-page`, `main-blog`, `password` |
| `disabled_on.groups: [header, footer]` | `carousel`, `collection-links`, `collection-list`, `featured-blog-posts`, `featured-product-information`, `layered-slideshow`, `main-cart`, `main-collection-list`, `marquee`, `product-hotspots`, `product-information`, `product-list`, `product-recommendations`, `section-rendering-product-card`, `slideshow` |
| none | `_blocks`, `custom-liquid`, `divider`, `header`, `cart-drawer-section`, `main-404`, `main-blog-post`, `password-footer`, `predictive-search*`, `search-header`, `search-results` |

- Sections without `presets` cannot be added in the editor; they exist only through JSON (all `main-*`, `header`, `search-*`, `product-information`, `cart-drawer-section`, `predictive-search*`, `password*`, `section-rendering-product-card`).
- New `cx-` sections SHOULD declare `disabled_on: { "groups": ["header"] }` unless built for the header.

## 5. Sections

### 5.1 The generic section: `section` (and `_blocks`)

`sections/section.liquid` ("Section" in the editor) is the composition workhorse. Build new bands from it before writing a section.

```
sections/section.liquid
  capture children = {% content_for 'blocks' %}                       (:1-3)
  render 'section', section, children, section_id, background_color   (:5-10)
    snippets/section.liquid
      contrast-override (skip_contrast when background media)         (:10-20)
      <div class="section-background color-custom-{id}">              (:22)
      <div class="section section--{section_width} color-custom-{id}"> (:23-24)
        --section-min-height (small|medium|large|full-screen|custom svh)  (:31-43)
        background-media (image|video) + border-override + overlay     (:45-61)
        <div class="spacing-style layout-panel-flex layout-panel-flex--{dir} [mobile-column] section-content-wrapper"
             style="layout-panel-style + spacing-style">               (:63-76)
          {{ children }}                                               (:77)
```

Settings (both files; `sections/section.liquid` lines): `content_direction` column|row (37), `vertical_on_mobile` (53), `horizontal_alignment` (60), `vertical_alignment` (85), `align_baseline` (106), `horizontal_alignment_flex_direction_column` (113), `vertical_alignment_flex_direction_column` (134), `gap` 0-100 =12 (159), `section_width` page-width|full-width (173), `section_height` ""|small|medium|large|full-screen|custom (189) + `section_height_custom` svh (221), `background_media` none|image|video (236), `background_color` (256), `video`/`video_position`, `background_image`/`background_image_position`, `toggle_overlay`/`overlay_color`/`overlay_style`/`gradient_direction`, `border`/`border_width`/`border_opacity`/`border_color`/`border_radius`, `padding-block-start`/`-end`. No inline padding settings.

13 presets of `section` (block trees are composition recipes):

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

`_blocks`: identical markup and schema (minus `disabled_on`), one preset. It is the platform wrapper for AI-generated blocks and top-level app blocks (`sections/_blocks.liquid:1-5`; PLATFORM [AIB][AB]). Treat as platform-reserved: do not repurpose; keep its schema in sync with `section` if the shared schema ever changes.

### 5.2 Hand-rolled section skeleton (every other section)

```liquid
{% liquid
  if section.settings.background_color != blank
    render 'contrast-override', background_color: section.settings.background_color, section_id: section.id
  endif
%}

<div
  class="section-background{% if section.settings.background_color != blank %} color-custom-{{ section.id }}{% endif %}"
></div>
<div
  class="
    section
    section--{{ section.settings.section_width }}
    {% if section.settings.background_color != blank %}color-custom-{{ section.id }}{% endif %}
    spacing-style
    gap-style
  "
  style="
    {% render 'spacing-style', settings: section.settings %}
    {% render 'gap-style', value: section.settings.gap %}
  "
>
  {% content_for 'blocks' %}
</div>
```

Validated with `validate_theme` (2026-10-10) together with a schema holding `section_width`, `gap`, `background_color`, `padding-block-start`, `padding-block-end` (copy them from `sections/product-list.liquid`). Drop `gap-style` and its render when the section has no `gap` setting.

Verbatim precedents: `sections/carousel.liquid:1-24`, `sections/collection-list.liquid:16-40`, `sections/product-list.liquid:1-25`, `sections/divider.liquid:7-20`.

Rules:
- `.section` is a 3-column grid (margin | centre | margin, `assets/base.css:307-340`). `.section--page-width > *` sits in the centre column; `.section--full-width > *` spans all (`:360-367`); `.section--mobile-full-width`, `.section--full-width-margin`, `.force-full-width` exist (`:369-387`). Do not rewrite this as `max-width` + `margin:auto` (comment `:303-306`).
- `.section` is forced transparent inside `.shopify-section` (`:343-345`); the background is the sibling `.section-background` (`:351-357`). Put `color-custom-{{ section.id }}` on **both** divs.
- Recurring setting IDs (reuse them): `background_color` (35 sections), `padding-block-start/-end` (34), `section_width` page-width|full-width (24), `gap` (16), layout set (`content_direction` …), `section_height`, background media set, overlay set, border set, resource-list set (`layout_type`, `columns`, `mobile_columns`, `columns_gap`, `rows_gap`, `carousel_on_mobile`, `mobile_card_size`), `icons_style`/`icons_shape`.
- Transparent header: the first section in `<main>` gets `--section-top-offset` added to `padding-block-start` (`assets/base.css:1820-1838`). Opt out with class `disable-section-top-offset` (used by `media-with-content`, `slideshow`).
- Section schema `class` is added to the platform `shopify-section` wrapper (e.g. `section-wrapper`). Section `tag` ∈ `article|aside|div|footer|header|section`.
- New sections MUST start with a contract comment (stock sections have none; new ones must):

```liquid
{% comment %}
  Section contract.
  <Region and responsibility.>
  @example
  <!-- How the section is configured or reached -->
{% endcomment %}
```

### 5.3 Section catalog (stock 4.2.0)

| Section (editor name) | Purpose | Blocks | Presets |
|---|---|---|---|
| `section` (Section) | Generic band | `@theme`, `@app`, `_divider` | 13 (§5.1) |
| `_blocks` (Section) | Platform wrapper (AI / app blocks) | `@theme`, `@app`, `_divider` | 1 |
| `hero` | Full-bleed banner: 2 desktop + 2 mobile media slots (image/video), overlay, blurred reflection, whole-hero link | dynamic: `@theme`, `text`, `button`, `logo`, `jumbo-text`, `spacer`, `group`, `_marquee` | Hero, Hero: Marquee, Hero: Bottom aligned |
| `slideshow` | `_slide` carousel, full-frame or inset | `_slide` | Full frame, Inset |
| `layered-slideshow` | Tabbed stacked panels (≤6) | `_layered-slide` | 1 |
| `carousel` | Header + horizontal `_card` carousel | static `group#static-header`, `_carousel-content#static-carousel-content` (→ `_card`) | 1 |
| `media-with-content` (Media with text) | Split media / content group | static `_media-without-appearance#media`, `_content-without-appearance#content` | Editorial, Editorial: Jumbo text |
| `marquee` | Infinite scrolling strip | `text`, `icon`, `logo`, `_divider` | 1 |
| `collection-links` | Collection titles; spotlight image slideshow or text list | static `_collection-link#link` (loop) | Spotlight, Text |
| `collection-list` | Collection cards: grid, carousel, bento, editorial | header area `@theme…`; static `_collection-card#static-collection-card` (loop) | Bento, Grid, Carousel, Editorial |
| `product-list` (Featured collection) | Products of one collection: grid, carousel, editorial | static `_product-list-content#static-header`, `_product-card#static-product-card` (loop); trailing area `@theme`, `@app`, `_divider` | Grid, Carousel, Editorial |
| `featured-blog-posts` (Blog posts) | Articles of one blog | static `_featured-blog-posts-title`, `_featured-blog-posts-card` (loop); no free area | Carousel, Grid, Editorial |
| `featured-product` (Product highlight) | One product: media + card | static `_media-without-appearance#media`, `_featured-product#featured-product` | 1 |
| `featured-product-information` (Featured product) | Product-page layout for a chosen product, any template | static `_featured-product-information-carousel#media-gallery`, `_product-details#product-details`; `@theme`, `@app` | 1 |
| `product-hotspots` | Image with product hotspots | static `text#heading`; `_hotspot-product` | 1 |
| `product-recommendations` | Related / complementary products (lazy, hydration) | header `@theme…`; static `_product-card#static-product-card` | 1 |
| `logo` (footer only) | Large logo band | none | 1 |
| `divider` | Horizontal rule band | none | 1 |
| `custom-liquid` | Merchant Liquid band (L4) | none | 1 |
| `header-announcements` (header only) | Rotating announcements | `_announcement` | 1 |
| `header` | Header (§5.4) | static `_header-logo`, `_header-menu` ×3 | none |
| `footer` (footer only) | Footer grid | explicit list (§3.6) | 1 |
| `footer-utilities` (Policies and links; footer only) | Copyright, policies, social | 3 types, `max_blocks: 3` | 1 |
| `main-page` | Page body | `@theme`, `@app`, `_divider` | none |
| `main-404` | 404 body | `@theme`, `@app`, `_divider` | none |
| `main-blog` | Blog listing (editorial grid) | free header area; static `_blog-post-card` (loop) | none |
| `main-blog-post` | Article body + comments | `@theme`, `@app`, `_blog-post-*` | none |
| `main-collection` | Collection grid + filters | static `filters`, `_product-card#product-card` | none |
| `main-collection-list` | All collections | header area; static `_collection-card` | none |
| `main-cart` | Cart page | static `_cart-title`, `_cart-products`, `_cart-summary`; area `@theme…` | none |
| `search-header`, `search-results` | Search page | statics (`_heading`, `_search-input`; `filters`, `_product-card`) | none |
| `product-information` | Product page | static `_product-media-gallery`, `_product-details`; area `@theme`, `@app` | none |
| `quick-order-list` | Variant table (product templates only) | none | 1 |
| `password`, `password-footer` | Password page | `@theme`, `@app`, `_divider` / none | none |
| `cart-drawer-section`, `predictive-search`, `predictive-search-empty`, `section-rendering-product-card` | Section Rendering API targets only | — (`predictive-search` declares `@theme` but renders no block area) | none |

Preset names in this table drop the `<Section name>: ` prefix that stock uses (e.g. `Slideshow: Full frame`, `Collection list: Bento`, `Featured collection: Grid`, `Blog posts: Carousel`).

Resource lists (collection-list, main-collection-list, product-list, featured-blog-posts) render through `snippets/resource-list.liquid` (layouts `grid|carousel|bento|editorial`, `carousel_on_mobile`); product-recommendations (section and block) write the same `resource-list` markup inline with `resource-list-carousel` (grid|carousel only). Items are separated by `<!--@list/split-->`.

### 5.4 Header

- Static only: `_header-logo#header-logo`; `_header-menu#header-menu` rendered three times: desktop menu, `variant: 'mobile'` (drawer), `variant: 'navigation_bar'` (`sections/header.liquid:81-95`). One settings record, three UIs.
- Rows: `top`/`bottom`, each a `.section.section--full-width-margin` grid with left/center/right columns built by `snippets/header-row.liquid` from `<item>_position` / `<item>_row` settings. Per-row colors via `contrast-override` with `section_id: <row>-<section.id>`.
- Desktop menu vs drawer: `#header-component[data-menu-style="menu"|"drawer"]`, set pre-paint in `layout/theme.liquid:141-149` (touch device or `overflow-list[minimum-reached]` → drawer). All submenus are mega-menu panels (`snippets/mega-menu-list.liquid`; `menu_style` text|collection_images|featured_products|featured_collections).
- Sticky: `enable_sticky_header` always|scroll-up|never → `sticky` attribute; `data-sticky-state` driven by `assets/header.js`.
- Transparent: per-template toggles (home/product/collection) → `transparent` attribute; inverse logo; `--header-height`, `--header-group-height`, `--transparent-header-offset-boolean` measured by `layout/theme.liquid:45-135`. New first-paint consumers of those vars MUST `render 'measure-header-heights'`.
- Hydration: after load, `hydrate(section.id)` re-renders the header in hydration mode; only `data-hydration-key` targets (`header-menu`, `header-drawer-mobile`, `header-menu-mobile-navigation-bar`) are morphed. Featured products in menus are queried only on that second render (`section.index == blank` detection, `snippets/mega-menu-list.liquid:29-41`).
- `sections/header.liquid` is 1,817 lines with 42 settings. Change it through settings first. Header CSS edits are L5.

## 6. Blocks: catalog and authoring

### 6.1 Authoring conventions (derived from stock; templates in Appendix A)

File order: `{% doc %}` → Liquid/markup → one `{% stylesheet %}` → `{% schema %}`.

1. `{% doc %}` header on every new block: what it renders, `@param` for every `content_for` keyword arg, `@example` with `content_for`. (Stock: 20 of 95 blocks have it. Theme Check validates `content_for` keyword args against the block's LiquidDoc: `UnrecognizedContentForArguments`, `MissingContentForArguments`, `ValidContentForArgumentTypes`.) Settings read from `block.settings` are part of the block contract, not `@param`s.
2. First line of logic: `{% assign block_settings = block.settings %}` (or inside `{% liquid %}`).
3. Colors: if the block has `background_color` / `text_color`, `render 'contrast-override'` with `section_id: block.id` before the element and add `color-custom-{{ block.id }}` to the root only when one of them is set (`blocks/page.liquid:3-10`, `blocks/accordion.liquid:3-10`).
4. One root element, `"tag": null`, carrying `{{ block.shopify_attributes }}` once.
5. Utility classes paired with their helper renders (§8.1): `spacing-style` ⇄ `render 'spacing-style'`, `size-style` ⇄ `render 'size-style'`, `border-style` ⇄ `render 'border-override'`, `layout-panel-flex layout-panel-flex--{dir} [mobile-column]` ⇄ `render 'layout-panel-style'`, `custom-typography custom-font-size custom-font-weight` ⇄ `render 'typography-style'`.
6. Instance values: inline custom properties namespaced to the block (`--cx-name-*`). Use `{% style %}` scoped by `.cx-name--{{ block.id }}` only when a rule needs a selector, pseudo-class or media query a custom property cannot express (`blocks/icon.liquid:2-10`).
7. Containers: `{% content_for 'blocks' %}` once; `blocks: [@theme, @app, _divider]` unless restricted.
8. Resource context: read `closest.*`; do not read `section.settings` unless the block is private to one section.
9. Plain merchant strings escaped (`| escape`) in HTML and attributes; `richtext`/`inline_richtext` output as-is.
10. DOM ids unique and stable per instance: append `block.id` (and `section.id` if needed); never derive them from values that change between renders.
11. Presets: at least one, so it appears in the picker; `category` from `t:categories.*` (`basic`, `layout`, `decorative`, `forms`, `links`, `footer`, `collection`, `product`, `blog_post`, `custom`).
12. Schema strings are `t:` keys (§14).

### 6.2 Standard setting groups (copy verbatim so shared helpers work)

| Group | IDs (type, default) | Consumer |
|---|---|---|
| Padding (`t:content.padding`) | `padding-block-start`, `padding-block-end`, `padding-inline-start`, `padding-inline-end` (range 0–100 px step 1, =0; labels `t:settings.top/bottom/left/right`) | `spacing-style` (scales >20px by 0.7 below 990px) or `spacing-padding` (unscaled) |
| Size (`t:content.size`) | `width` fit-content\|fill\|custom =fill; `custom_width` 0–100 % =100 ⟨`width == 'custom'`⟩; `width_mobile` same =fill; `custom_width_mobile`; `height` fit\|fill\|custom =fit; `custom_height` | `size-style` |
| Borders (`t:content.borders`) | `border` none\|solid =none; `border_width` 0–100 =1, `border_opacity` 0–100 % =100, `border_color` (placeholder) ⟨`border != 'none'`⟩; `border_radius` 0–100 =0 | `border-override` (opacity applies only when `border_color` blank) |
| Layout (`t:content.layout`) | `content_direction` column\|row =column; `vertical_on_mobile` =true ⟨row⟩; `horizontal_alignment` flex-start\|center\|flex-end\|space-between ⟨row⟩; `vertical_alignment` flex-start\|center\|flex-end =center ⟨row⟩; `align_baseline` ⟨`vertical_alignment == 'flex-end'`⟩; `horizontal_alignment_flex_direction_column` ⟨`!= 'row'`⟩; `vertical_alignment_flex_direction_column` (+space-between) ⟨`== 'column'`⟩; `gap` 0–100 =12 | `layout-panel-style` |
| Colors (`t:content.appearance`) | optional overrides: `background_color`, `text_color`, `border_color`, … type `color`, **no default**, `"placeholder": "t:settings.default"` (blank = inherit) | `contrast-override` |
| Always-resolving colors | `"default": "{{ settings.color_palette.foreground }}"` / `.background`, or literal `#00000026` (overlays) | direct |
| Background media | `background_media` none\|image\|video; `background_color` ⟨none⟩; `video`, `video_position` cover\|contain; `background_image`, `background_image_position` cover\|fit; `toggle_overlay`, `overlay_color` =#00000026, `overlay_style` solid\|gradient, `gradient_direction` to top\|to bottom | `background-media`, `overlay` |
| Typography (text family) | `type_preset` rte\|paragraph\|h1…h6\|custom =rte (`info: t:info.edit_presets_in_theme_settings`); when custom: `font` (values `var(--font-body--family)` …), `font_size` (rem strings), `line_height`/`letter_spacing` tight\|normal\|loose, `case` none\|uppercase, `wrap` pretty\|balance\|nowrap | class `{{ type_preset }}` + `typography-style` |
| Lightweight preset | select `""`\|paragraph\|h1…h6 emitted as a class (`heading_preset`, `link_preset`, `text_preset`, …) | `.paragraph`, `.h1`–`.h6` (base.css) |
| Link | `link` (url), `open_in_new_tab` checkbox =false | markup |
| Hidden per-instance flag | `"visible_if": "{{ false }}"` set only by presets/templates (`group.placeholder`, `_heading.read_only`) | markup |

ID conventions: kebab-case only for padding (`padding-*`) and `x-position`/`y-position`; everything else snake_case. Same concept, different IDs exist in stock (`alignment` text_alignment vs flex `horizontal_alignment`; `image_ratio` vs `aspect_ratio` vs `ratio`; `font` as CSS var vs bare key in `jumbo-text`/`logo`). New blocks: copy the group from the closest stock block of the same family.

`visible_if`: `"{{ block.settings.x == 'y' }}"` (blocks) / `"{{ section.settings.x … }}"` (sections). Stock also reads `section.settings` from a block's `visible_if` and global `settings` (`blocks/_product-media-gallery.liquid:258`, `blocks/logo.liquid:111`); undocumented but working. Supported only on basic inputs, sidebar settings and these specialized types: `color`, `color_background`, `color_scheme`, `font_picker`, `html`, `image_picker`, `inline_richtext`, `link_list`, `liquid`, `richtext`, `text_alignment`, `url`, `video`, `video_url` (PLATFORM [SET]). Not on `product`, `collection`, `page`, `blog`, `article`, `metaobject`, `*_list` pickers, `color_palette`. Conditions cannot read what a dynamic source resolves to.

### 6.3 Public block catalog (44)

Every public block with a preset is addable in every `@theme` container (§3.6). "—" = no preset (static/JSON only).

| Block | Preset category | Purpose | Children | Notable settings / notes |
|---|---|---|---|---|
| `group` | layout | Flex container: layout, size, background image/video, overlay, border, full-card link, padding. **Default building unit.** | `@theme`, `@app`, `_divider` | 36 settings; renders `snippets/group.liquid` |
| `text` | basic (Text, Heading) | Rich text / headings; heading level from the richtext markup, style from `type_preset` | — | `text` (richtext), width fit-content\|100%, max_width, alignment, typography, `text_color`, background pill (`background`, `background_color`, `corner_radius`), padding (unscaled `spacing-padding`). Supports dynamic sources. |
| `button` | basic | Link styled as button | — | `label`, `link`, `style_class` button\|button-secondary\|button-custom\|button-unstyled (+ custom colors), size |
| `image` | basic | Single image | — | `image`, `link`, `image_ratio` adapt\|portrait\|square\|landscape, size, border, padding |
| `icon` | basic | Built-in icon or uploaded image | — | `icon`, `image_upload`, `width`, `link`, `icon_color` |
| `video` | basic | Uploaded video or YouTube/Vimeo | — | `source`, `video`, `video_url`, autoplay, loop, `cover_image`, `alt`, `aspect_ratio`, border |
| `spacer` | layout | Flexible/fixed gap in a flex parent | — | `size` percent\|pixel, mobile override |
| `accordion` | layout | Collapsible rows | `_accordion-row` only | icon caret\|plus, dividers, `type_preset`, colors, border |
| `custom-liquid` | custom | Merchant Liquid (L4) | — | `custom_liquid` (type `liquid`). Stock omits `block.shopify_attributes`. |
| `email-signup` | forms | Newsletter (`{% form 'customer' %}`) | — | display_type, integrated button, input + button styles |
| `contact-form` | forms | Contact form (`{% form 'contact' %}`) | static `contact-form-submit-button` | size, input styles; DOM ids not unique per instance |
| `contact-form-submit-button` | — | Submit button of `contact-form` (static only) | — | label, style |
| `jumbo-text` | decorative | Text auto-fit to container width | — | `text`, `font` (bare key), `text_effect`, `animation_repeat` |
| `logo` | basic | Shop logo (`settings.logo` / `logo_inverse`), shop-name fallback | — | unit percent\|pixel, sizes, `inverse` |
| `menu` | links | Link list with heading; mobile accordion option | — | `menu`, `heading`, presets for heading/links, accordion |
| `social-links` | footer | Social icon row | — | `icon_color`; per-platform URL settings (12 platforms + `custom_url`) |
| `payment-icons` | footer | Enabled payment type icons | — | alignment, gap |
| `follow-on-shop` | footer | Shop "Follow" button (if `shop.features.follow_on_shop?`) | — | padding |
| `footer-copyright` | footer | © year + shop link (+ powered by) | — | `show_powered_by`, font size, case |
| `footer-policy-list` | footer | Policies popover (`shop.policies`) | — | font size, case |
| `popup-link` | links | Text link opening a modal/drawer dialog | `@theme`, `@app` | `behavior`, `heading`, `type_preset` |
| `comparison-slider` | decorative | Before/after image slider | — | images, labels, slider colors, ratio, size, border |
| `collection-card` | collection | Card for one picked collection | `text`, `spacer`, `button`, `group`, `collection-title`; static `_collection-card-image` | `collection`, placement, alignment, gap, border |
| `collection-title` | collection | `closest.collection` title (text styling) | — | text-family settings |
| `featured-collection` | — | Container providing `closest.collection` to children | `@theme`, `@app` | `collection`. No preset, no stock reference. |
| `page` | basic | Picked page title + content | — | `page`, `max_width` (no CSS consumer), colors |
| `page-content` | — | Current page `page.content` | — | `text_color`. Placed via JSON in `templates/page.json`. |
| `product-title` | product | `closest.product` title (link) | — | text-family settings |
| `product-description` | product | Text block preset bound to `{{ closest.product.description }}` | — | identical to `text` |
| `price` | product | Price, sale, installments, tax note | — | `show_sale_price_first`, `show_installments`, `show_tax_info`, typography |
| `variant-picker` | product | Variant options (buttons/dropdowns, swatches) | — | `variant_style` dropdowns\|buttons, `show_swatches`, custom colors |
| `buy-buttons` | product | Product form: quantity, add to cart, accelerated checkout, pickup, gift-card recipient | static `quantity`, `add-to-cart`, `accelerated-checkout` | `stacking`, `show_pickup_availability`, `gift_card_form`. One form per section (§13.1). |
| `add-to-cart` | — | Add button (static in `buy-buttons`) | — | `style_class` + custom colors |
| `accelerated-checkout` | — | Dynamic checkout buttons (static in `buy-buttons`; outputs nothing standalone) | — | none |
| `quantity` | product | Quantity selector | — | input styles. Not wired to a form when standalone. |
| `product-inventory` | product | Stock level message | — | `inventory_threshold`, `show_inventory_quantity` |
| `sku` | product | Variant SKU | — | typography |
| `review` | product | Stars from `reviews.rating` metafields (needs a reviews app) | — | `stars_style`, `show_number` |
| `swatches` | product | First swatch option of `closest.product` (cards) | — | alignment, padding |
| `product-custom-property` | product | Buyer input → line-item `properties[key]` | — | key, input type text\|checkbox, `required`, max length. Needs `buy-buttons` in same section. |
| `product-card` | product | Card for one picked product | card list (§13.3), no `buy-buttons` | `product`, gap, width, border |
| `product-recommendations` | product | Recommendations inside a container (lazy) | `@theme`, `@app`; static `_product-card` | `recommendation_type` related\|complementary, layout |
| `disclosures` | product | Product disclosures (`shopify.disclosure` metafield) accordion | — | heading, presets, dividers |
| `filters` | — | Filtering, sorting, grid density (static in collection/search) | — | `enable_filtering`, `filter_style` horizontal\|vertical, sorting, density |

### 6.4 Private block catalog (51)

| Block | Owner(s) (where listed or rendered) | Role |
|---|---|---|
| `_divider` | `group`, `section`, `_blocks`, `main-page`, `main-404`, `main-blog`, `main-collection-list`, `password`, `product-list`, `collection-list`, `product-recommendations` (section), `footer`, `marquee`, `_marquee`, `_content-without-appearance`, `_product-details`, `_product-list-content` | Rule; horizontal in column parents, vertical in row parents |
| `_accordion-row` | `accordion` | `<details>` row; accepts `@theme`, `@app` |
| `_card` | `_carousel-content` | Carousel card (layout, background media, link) |
| `_carousel-content` | `carousel` (static) | Carousel track of `_card` |
| `_slide` | `slideshow` | Slide (media + content) |
| `_layered-slide` | `layered-slideshow` | Layered panel |
| `_marquee` | `hero` | Inline marquee inside hero |
| `_heading` | `_card`, `_slide`, `_layered-slide`, statics in blog cards, `search-header` | Heading with `text` keyword arg |
| `_media-without-appearance` | `media-with-content`, `featured-product` (static) | Media half |
| `_content-without-appearance` | `media-with-content` (static) | Content group |
| `_announcement` | `header-announcements` | Announcement message |
| `_header-logo`, `_header-menu` | `header` (static) | Logo; menu (3 variants) |
| `_search-input` | `search-header` (static) | Search field |
| `_image`, `_inline-collection-title`, `_collection-link` | `collection-links` (static) | Collection link parts |
| `_collection-card`, `_collection-card-image` | `_collection-card`: static in `collection-list`, `main-collection-list`; `_collection-card-image`: static in `_collection-card` and public `collection-card` | Collection card + image |
| `_product-card`, `_product-card-gallery`, `_product-card-group` | `_product-card`: static in `product-list`, `main-collection`, `search-results`, `product-recommendations` (section + block); `_product-card-gallery`, `_product-card-group`: dynamic children of `_product-card` / `product-card` (`_product-card-group` nests itself) | Product card system (§13.3) |
| `_product-list-content`, `_product-list-text`, `_product-list-button` | `product-list` (static header) | Header + "View all" |
| `_product-details`, `_product-media-gallery` | `product-information` (static); `_product-details` also `featured-product-information` | Product page columns |
| `_featured-product`, `_featured-product-gallery`, `_featured-product-price` | `featured-product` (static) | Product highlight card |
| `_featured-product-information-carousel` | `featured-product-information` (static) | Media carousel |
| `_hotspot-product` | `product-hotspots` | Hotspot (own `product` setting, `x-position`, `y-position`) |
| `_featured-blog-posts-title`, `_featured-blog-posts-card`, `_featured-blog-posts-image` | `featured-blog-posts` | Blog posts band |
| `_blog-post-card`, `_blog-post-image`, `_blog-post-description` | `main-blog` (static), `_featured-blog-posts-card` | Blog listing card |
| `_blog-post-title`, `_blog-post-info-text`, `_blog-post-featured-image`, `_blog-post-content` | `main-blog-post`; `_blog-post-info-text` also static in `_blog-post-card`, `_featured-blog-posts-card` | Article parts |
| `_cart-title`, `_cart-products`, `_cart-summary` | `main-cart` (static) | Cart page parts |
| **Orphans** `_collection-image`, `_collection-info`, `_content`, `_footer-social-icons`, `_inline-text` (+ `_social-link`, used only by an orphan) | none | Not reachable in 4.2.0. Do not build on them without wiring them into a parent; prefer `group`, `social-links`, `_inline-collection-title`. `_content.background_color` has no effect. |

Reusing a stock private block in a new `cx-` section (static `content_for 'block'`) is allowed and often the right "use Horizon" move (e.g. static `_product-card` for a product loop gives merchants the full card editor). Check its parent coupling first (§6.5) and provide the section settings it reads.

### 6.5 Parent coupling (blocks that read `section.settings` directly)

| Block / snippet | Reads `section.settings.*` | Owning section |
|---|---|---|
| `_carousel-content` | `icons_style`, `section_width`, `columns`, `mobile_columns`, `columns_gap`, `icons_shape` | `carousel` |
| `_collection-link` | `image_height` (undefined in its section → nil), `layout` | `collection-links` |
| `_featured-blog-posts-card` / `-title` | `layout_type` / `blog` | `featured-blog-posts` |
| `_header-logo`, `_header-menu` | transparent flags, inverse logo, `menu_row`, `background_color_{row}`, `section_width` | `header` |
| `_hotspot-product` | `hotspot_color`, `bullseye_color`, `product_title_preset`, `product_price_preset` | `product-hotspots` |
| `_layered-slide` | `drop_shadow` | `layered-slideshow` |
| `_slide` | `background_color`, `content_position`, `slide_height` | `slideshow` |
| `_product-list-button` | `max_products` | `product-list` |
| `snippets/card-gallery` (product cards) | `product_card_size`, `mobile_product_card_size`, `columns_gap_horizontal`, `columns`, `layout_type`, `product_grid_width` (via `util-product-grid-card-size`) (image `sizes` only; falls back to `(min-width: 750px) 50vw, 100vw`) | any card host |
| `snippets/collection-card` (via `_collection-card`, `collection-card`) | `layout_type` (bento/editorial classes) | `collection-list`, `main-collection-list`, any host |
| `snippets/background-media` | `section_width` (image `sizes` in block context) | any |
| `_product-media-gallery`, `_featured-product-information-carousel` (via `util-product-media-sizes-attr`) | `content_width`, `equal_columns`, `gap` | product sections |
| `snippets/header-drawer`, `header-actions` | localization and actions settings | `header` |

Other context reads: `section.id` (DOM/form ids and Quick Add / SRA targets in `product-custom-property`, `buy-buttons`, `product-inventory`, `contact-form`, `filters`, `product-recommendations`, `_product-details`, `_slide`, `_layered-slide`, `_blog-post-card`, `_cart-summary`, `_hotspot-product`, `_product-card-gallery`, `_featured-product-gallery`), `section.index` (`background-media` and `card-gallery` loading/fetchpriority, `_slide`, SRA detection in `mega-menu-list` / `header-drawer`), `section.blocks | find_index` (`_slide`, `_layered-slide`, `_announcement`).

### 6.6 Settings and schema design rules

- Settings are a public customization API. Add a setting only for a choice a merchant should control; keep structural implementation details in code so every combination of controls produces a valid result.
- Theme settings for shop-wide choices; section/block settings for content or presentation of that context (§11).
- Use resource pickers (`product`, `collection`, `page`, `blog`, `article`, `image_picker`, `video`, `link_list`, `url`, `metaobject`, `*_list`) for merchant-selected products, collections, pages, media, menus and links — never free-text handles.
- Prefer focused controls, useful defaults and conditional `visible_if` over many always-visible options. Keep each `visible_if` aligned with the IDs and option values it references; re-check them when you rename or add options.
- Use only attributes supported by the setting type (e.g. `range` needs `min`, `max`, numeric `default`; `richtext` default starts with `<p>`/`<ul>`; `url` default only `/collections` or `/collections/all`; `product`/`collection`/`page`/`blog`/`article` take no `default`; `link_list` default only `main-menu` or `footer` per docs (stock also uses `customer-account-main-menu`, `sections/header.liquid:1324`)) and by the schema kind (block: `name`, `settings`, `blocks`, `presets`, `tag`, `class`; section adds `limit`, `max_blocks`, `enabled_on`/`disabled_on`, `default`, `locales`).
- Omit the `settings` key when a component exposes no settings.
- Presets are useful starting compositions, not empty shells. Keep preset block keys, `block_order`, `"static": true` markers and nested settings aligned with what the Liquid actually renders. Re-check every preset (and templates that use the component) whenever a setting, an accepted block type or the nested composition changes (Theme Check: `ValidSettingsKey`, `SchemaPresetsBlockOrder`, `SchemaPresetsStaticBlocks`, `ValidBlockTarget`).
- Static vs dynamic choice: use a static block for a required structural component (fixed place, fixed identity); use dynamic blocks where merchants control presence and order. A static block may still have settings and nested blocks.
- Keep schema JSON valid; every name, label, option label, header, info, placeholder and preset name is a literal `t:` reference (§14).

### 6.7 Liquid style

- Liquid renders the initial HTML; JS only adds behavior beyond semantic HTML and CSS.
- `{% liquid %}` for multi-line logic, with `#` comments inside it; `{%- -%}` / `{{- -}}` to control whitespace where markup is sensitive.
- Object dot notation (`product.selected_or_first_available_variant.id`); bracket access only for computed keys (`settings[key]`, as in `snippets/spacing-style.liquid`). Use only Shopify objects, tags and filters that exist in current source or platform docs; `validate_theme` rejects unknown ones.
- Render optional content only when it has a value (`{% if block_settings.heading != blank %}…{% endif %}`) so output stays meaningful and accessible; no empty wrappers, labels or headings.
- Escape plain merchant strings in HTML and attributes (`| escape`); output `richtext`/`inline_richtext` unescaped; never build HTML from unescaped plain text.
- Comments: short, only for non-obvious intent or coupling (e.g. "reads `section.settings.columns` from the host section"). No narrating comments.

## 7. Layout and page shell

### 7.1 `layout/theme.liquid` render order

`<html class="[shopify-design-mode]" lang="{{ request.locale.iso_code }}" dir="{{ request.locale.direction | default: 'ltr' }}">` (`:2-8`).

| Line | Render | Gate | Emits |
|---|---|---|---|
| 10-16 | favicon link | `settings.favicon != blank` | |
| 18 | `view-transition-opt-in` | `page_transition_enabled` or `transition_to_main_product` | `@view-transition` + render blocker on `#MainContent` |
| 20 | `meta-tags` | | charset, viewport (`viewport-fit=cover`), OG/Twitter, title, canonical, description |
| 21 | `stylesheets` | | preload `overflow-list.css`; `base.css` (`stylesheet_tag: preload: true`) |
| 22 | `card-hover-effect-styles` | its `{% style %}` only if `card_hover_effect != 'none'` | card hover CSS (global so search-modal cards get it) |
| 23 | `fonts` | non-system fonts | font preloads |
| 24 | `scripts` | | import map, module preloads, global modules, `Theme` global (§12.1) |
| 25 | `theme-styles-variables` | | `@font-face` + most `:root` design tokens + 4 UI color scopes |
| 26 | `color-palette` | | `:root` color tokens + `body` color/background |
| 28-30 | `theme-editor` | `request.design_mode` | `assets/theme-editor.js` |
| 32 | `{{ content_for_header }}` | | Shopify head (last in `<head>`; never wrap/modify) |

Body: `<body class="page-width-{{ settings.page_width }} card-hover-effect-{{ settings.card_hover_effect }}">` → skip link → `chat-drawer` → `<div class="page-wrapper">` [`#header-group` → inline header-measure script (`:45-150`) → `<main id="MainContent" class="content-for-layout" data-template data-page-transition-enabled data-product-transition>` → `<footer>` footer group] → `cart-drawer` (`cart_type == 'drawer'` and not cart template) → `theme-drawer` → `search-modal` → `quick-add-modal` (`quick_add or mobile_quick_add`).

Shell facts that affect custom code:
- At ≥990px the scroll container is `.page-wrapper`, not `window` (`assets/base.css:28-56`). Use `@theme/scroll-container` helpers (§12.6).
- `body.page-width-{narrow|normal|wide}` sets `--page-content-width` (= `--narrow/normal/wide-page-width`, 90/120/150rem, `snippets/theme-styles-variables.liquid:138-140`), `--page-width` (content width + 2 × `--page-margin`) and `--page-margin` (`max(16px, safe-area insets)`; 40px ≥750px) (`assets/base.css:265-296`).
- `<main>` has no `tabindex="-1"` (stock deviation from Shopify a11y guidance).
- `layout/password.liquid`: `stylesheets` first, no card-hover, no `.page-wrapper`, no groups/drawers/search/quick-add; static `{% section 'password-footer' %}`; `{% form 'storefront_password' %}` dialog; `password-layout-styles` (layouts cannot hold `{% stylesheet %}`).
- `templates/gift_card.liquid`: `{% layout none %}`, own head and import map (`@theme/component`, `@theme/utilities`, `@theme/qr-code-generator`); loads `base.css`, `template-giftcard.css`, tokens and palette; not `scripts`.

### 7.2 Editing layouts

Layout edits are L5. Typical legitimate edits: one line linking a global stylesheet or rendering a global `cx-` snippet (§9.9). Keep `content_for_header` last in `<head>`, `content_for_layout` inside `<main id="MainContent">`. Mirror any global addition in `layout/password.liquid` if the password page needs it.

## 8. Snippets

Snippets render with `{% render 'name', param: value %}` in an isolated scope: pass every caller-local value explicitly. Globals (`settings`, `shop`, `cart`, `request`, `routes`, …) are available. Stock snippets also read `block`, `section` and `closest.*` from the rendering context without receiving them (e.g. `snippets/button.liquid:11`, `snippets/card-gallery.liquid:12-13`) — existing contracts; new snippets SHOULD take what they use as params. Snippets have no schema. 134 of 145 stock snippets start with `{% doc %}`; new snippets MUST (`@param {type} name - desc`, `[name]` optional, `@example`).

### 8.1 Style helpers (inline-style emitters, paired with base.css classes)

| Snippet | Call | Reads | Emits | Pair class |
|---|---|---|---|---|
| `spacing-style` | `style="{% render 'spacing-style', settings: block.settings %}"` | `padding-block-start/-end`, `padding-inline-start/-end` | `--padding-*`; values >20 → `max(20px, calc(var(--spacing-scale) * Npx))` (`--spacing-scale` 0.7 <990px, 1.0 ≥990px) | `.spacing-style` |
| `spacing-padding` | same | same | all 4 `--padding-*` in px, **unscaled** | `.spacing-style` |
| `size-style` | `settings:` | `width`, `custom_width`, `height`, `custom_height`, `width_mobile`, `custom_width_mobile` | `--size-style-width(-mobile)`, `--size-style-width-mobile-min`, `--size-style-height` | `.size-style` |
| `gap-style` | `value:`, `[name]`, `[scale_min]` (=24; doc says 20), `[disable_scaling]` | — | `--{name}`; >24 scaled 0.7 <990px | `.gap-style`, `.layout-panel-flex` |
| `layout-panel-style` | `settings:` | layout set + `gap` | `--flex-direction`, `--flex-wrap(-mobile)`, `--gap`, `--horizontal-alignment`, `--vertical-alignment(-mobile)` | `.layout-panel-flex`, `--row`, `--column`, `.mobile-column` |
| `border-override` | `settings:` | `border`, `border_width`, `border_opacity`, `border_color`, `border_radius` | `--border-*` + `overflow: clip` when radius >0 | `.border-style` |
| `typography-style` | `settings:`, `[type]`, `[preset]` | custom typography set | `--font-size`, `--font-weight`, `--font-family`, `--line-height`, `--letter-spacing`, `--text-transform`, `--text-wrap` — **only when preset is `custom`** | `.custom-typography`, `.custom-font-size`, `.custom-font-weight` (caller adds) |
| `contrast-override` | `background_color`, `text_color`, `border_color`, `section_id` (required), `[skip_contrast]`, `[preset]` content\|ui | — | `{% style %}` scoped to `.color-custom-<section_id>` (§10.2) | `color-custom-<id>` (caller adds) |
| `background-media` | `background_media`, `background_video(_position)`, `background_image(_position)`, `[placeholder]` | `section.index` (priority), `section.settings.section_width` (sizes) | media markup + own CSS | — |
| `overlay` | `settings:`, `[layer]` | `overlay_color`, `overlay_style`, `gradient_direction`, `border_radius` | overlay div | — |
| `input-custom-styles` | `block_settings:` | `input_background_color`, `input_text_color`, `input_border_color` (+ global `palette_input_background`, `page_text_color`) | `--color-input-background` (global fallback); `--color-input-text(-rgb)` always (explicit → `page_text_color` if contrast ≥4.5 → palette extreme); `--color-input-border` only when set | — |
| `button-custom-styles` | `block_settings`, `block_id` | `style_class`, `custom_button_*`, `link_text_color` | `{% style %}` `.button-custom--<id>` (+ hover); `.button-unstyled--<id>` link color (+ hover) when `style_class == 'button-unstyled'` and `link_text_color` set | `.button-custom` |

### 8.2 Component renderers (reuse instead of re-implementing)

| Snippet | Renders | Key params |
|---|---|---|
| `section` | generic section wrapper (§5.1) | `section`, `children`, `section_id`, `background_color` |
| `group` | group-like container (layout, media, overlay, link) | `children`, `settings`, `shopify_attributes`, `[class]`, `[style]`, `[background_color]`, `[block_id]` |
| `text` | every text-like block (renders `text-block-styles`; `<rte-formatter>` for rte/paragraph) | `[block]`, `[class]`, `[fallback_text]`, `[width]` |
| `button` | `<a>` button from `block.settings` | `link`, `[block]` |
| `image` | height-based `<img>` with 1x–3x srcset | `image`, `[height]`, `[class]`, … |
| `icon` / `icon-or-image` | SVG `<path>`s (70 names; caller supplies `<svg viewBox="0 0 20 20">`) / svg-or-upload | `icon` / `icon`, `image_upload`, `width`, `class_name` |
| `divider` | horizontal/vertical rule | `id`, `settings`, `[full_width]`, `[attributes]` |
| `video` | `<video>` or YouTube/Vimeo iframe (deferred) | 15 params |
| `media` | media of `_media-without-appearance` | |
| `slideshow`, `slideshow-slide`, `slideshow-controls`, `slideshow-arrows` | slideshow component | many |
| `resource-list`, `resource-list-carousel`, `bento-grid`, `editorial-*-grid` | grid/carousel/bento/editorial layouts for resource lists | `resource-list`: `settings`, `list_items(_array)`, `slide_count`, `content_type`; `resource-list-carousel`: `ref`, `slides`, `slide_count`, `settings`; bento/editorial: `items` |
| `resource-card`, `resource-image` | card for product/collection/article/page (predictive search, mega menu) | |
| `product-card`, `card-gallery`, `price`, `swatch`, `variant-main-picker`, `variant-swatches`, `quantity-selector`, `add-to-cart-button`, `quick-add`, `product-media*`, `product-grid` | commerce (§13) | |
| `accordion-custom-component` | `<accordion-custom>` wrapper (`details`/`summary`) | open/disable per breakpoint |
| `theme-drawer`, `theme-drawer-header` | drawer infra / header | |
| `localization-form`, `search`, `search-modal`, `header-*`, `mega-menu-list` | header parts | |
| `format-price`, `unit-price`, `tax-info`, `volume-pricing-info` | price helpers | |
| `util-*` | computed attributes (`sizes`, card size, hover color) | |
| `measure-header-heights` | triggers pre-paint header measurement | `only_when_first_section`, `only_on_desktop` |

### 8.3 Style-owner snippets (`*-styles.liquid`)

42 snippets; 36 hold one `{% stylesheet %}` that is collected only when the snippet renders. Rule: **any file that emits a component's markup MUST render that component's style owner** (or render the snippet that does), including fragment/hydration/Quick Add paths that bypass normal composition. Full consumer list: §9.6.

## 9. CSS architecture

### 9.1 Layers

| Layer | Where | Delivery |
|---|---|---|
| Global foundations, resets, section grid, utilities, type presets, buttons, form fields, keyframes | `assets/base.css` (1,838 lines) | `stylesheet_tag` in `snippets/stylesheets.liquid:2`; not subsetted |
| Design tokens (`:root`) + `@font-face` | `snippets/theme-styles-variables.liquid` (`{% style %}`) | `<head>` after base.css |
| Color tokens | `snippets/color-palette.liquid` (`{% style %}`) | `<head>` after tokens |
| Component CSS | one `{% stylesheet %}` per section/block/snippet (154 files: 20 sections, 51 blocks, 83 snippets) | platform bundle, injected via `content_for_header`; **stylesheet subsetting** includes only files rendered on the page (PLATFORM [SUB]) |
| Shared component CSS | 36 `snippets/*-styles.liquid` owners | rendered by every consumer |
| Instance values | inline `style="--x: …"` + `{% style %}` (26 tags in 24 files: 22 scoped by `section.id`/`block.id`; 4 global head styles in `theme-styles-variables`, `color-palette`, `card-hover-effect-styles`, `view-transition-opt-in`) | per render, not subsetted |
| Shadow DOM | `assets/overflow-list.css` | inside `<template shadowrootmode>` |
| Gift card page | `assets/template-giftcard.css` | `templates/gift_card.liquid:10` |

No `@layer`, no `@scope`. No stock `{% stylesheet %}` contains Liquid; none has more than one tag.

### 9.2 Rules

- Before any styling work, inspect the matching theme source: settings-backed tokens, shared classes and render helpers, which file owns the styles, and the composition pattern. Verify names in that source.
- Honor active merchant settings and preserve merchant-configurable choices: style through the tokens and classes the settings drive; never hard-code a value a setting controls.
- Preserve existing stylesheet loading paths (`snippets/stylesheets.liquid`, owner `{% stylesheet %}` tags, style-owner renders) and verify CSS delivery both in rendered pages and in Section Rendering responses. Liquid `{% stylesheet %}` support and backend delivery (bundling, subsetting) are separate contracts.
- One static `{% stylesheet %}` per owning file; no Liquid in it. Setting-driven values → inline custom properties on the element (preferred) or `{% style %}` scoped to the instance (`blocks/icon.liquid:2-10`). `{% style %}` color values live-update in the editor.
- Subsetting: CSS is available only if its file is in the page's render tree. Define classes in the file that uses them, a rendered parent, or a style-owner snippet rendered by every consumer. Cross-file class dependencies break (Theme Check `ValidScopedCSSClass`).
- Selectors: BEM-ish, low specificity (single class), shallow `&` nesting; resolve cascade conflicts through component scope and source order, not specificity escalation; root class = component/file name; modifiers from setting values (`cx-name--{{ value }}`); state via attributes (`[open]`, `[aria-expanded]`, `[hidden]`, `[data-*]`). Stock tools: `:where()` (54) to stay overridable; doubled classes (`.h1.h1`, `.text-block.text-block`) only to beat known stock rules.
- Tokens only: colors, spacing, type, radii, layers, motion from §9.3. Namespace component variables (`--cx-name-*`).
- Logical properties only (`margin-inline-start`, `inset-inline`, `padding-block`, `text-align: start|end`). 4.2.0 maps `left`/`right` setting values to `start`/`end`. Physical exceptions need `/* stylelint-disable-line csstools/use-logical -- reason */`. Mirror directional icons under `[dir='rtl']`. Verify LTR and RTL.
- Layout robustness: `min-width: 0` on flex/grid children, constrained media, wrap merchant text; prefer container-aware/intrinsic layouts.
- Motion: animate `transform`/`opacity`; use motion tokens; gate decorative animation in `@media (prefers-reduced-motion: no-preference)` (no global reset exists); keep layout stable.
- Breakpoints: `750px` primary (`min-width: 750px` / `max-width: 749px`), `990px` large (spacing/gap scale, page scroll container, drawers modal↔inline). JS mirror: `mediaQueryLarge` (750), `MODAL_BREAKPOINT` 990. Container queries (`container-type: inline-size` + `@container`) where the component's own width matters (`snippets/resource-list-styles.liquid:20-79`).
- Match selector intent and property ordering of adjacent stock source.

### 9.3 Token reference (stable names; values in `snippets/theme-styles-variables.liquid` unless noted)

| Group | Tokens |
|---|---|
| Fonts (settings) | `--font-{body\|subheading\|heading\|accent}--family/--style/--weight` |
| Type presets (settings) | `--font-{paragraph\|h1…h6}--family/--style/--weight/--size/--line-height/--case`, `--font-h{N}--letter-spacing`, `--font-{preset}--spacing` (margins) |
| Fixed font sizes | `--font-size--3xs` .625rem … `--font-size--6xl` 3.5rem (12 steps) |
| Line height / tracking | `--line-height--{display\|heading\|body}-{tight\|normal\|loose}`, `--letter-spacing--{display\|heading\|body}-{tight\|normal\|loose}` |
| Measure | `--max-width--{body\|heading}-{normal\|narrow}`, `--max-width--display-{normal\|narrow\|tight}` |
| Spacing | `--padding-{3xs…6xl}`, `--margin-{3xs,2xs,xs,sm,md,lg,xl,2xl,4xl,6xl}` (no `3xl`/`5xl`), `--gap-{3xs…3xl}` (rem), `--spacing-scale-md` .7, `--spacing-scale-default` 1 |
| Page | `--page-width`, `--page-margin`, `--page-content-width` (these three in `assets/base.css:265-301`), `--narrow/normal/wide-page-width`, `--narrow/normal-content-width`, `--sidebar-width`, `--theme-drawer-width` |
| Section heights | `--section-height-{small\|medium\|large}` |
| Radii / borders (settings) | `--style-border-radius-{xs\|sm\|md\|lg\|50}`, `--style-border-radius-{pills\|inputs\|buttons-primary\|buttons-secondary\|popover}`, `--style-border-width(-primary\|-secondary\|-inputs)`, `--style-border-popover`, `--style-border-drawer` |
| Layers (z-index) | `--layer-section-background` -2, `--layer-lowest` -1, `--layer-base` 0, `--layer-flat` 1, `--layer-raised` 2, `--layer-heightened` 4, `--layer-sticky` 8, `--layer-window-overlay` 10, `--layer-header-menu` 12, `--layer-overlay` 16, `--layer-menu-drawer` 18, `--layer-temporary` 20 |
| Opacity | `--opacity-{5,8,10,15,20,25,30,40,50,60,70,80,85,90}`, `--opacity-subdued-text` .8, `--opacity-muted-text` .6; brightness-aware `--opacity-5-15`, `-10-25`, `-35-55`, `-40-60`, `-30-60` (color-palette / contrast-override) |
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
| Typography | `.paragraph`, `.h1`…`.h6` (bare `h1`…`h6` also get presets), `.rte` |
| Buttons / links | `.button`, `.button-secondary`, `.button-custom`, `.button-unstyled(--with-icon\|--transparent)`, link underline animation |
| Forms | `.field`, `.field__input`, `.field__label`, radio/checkbox base |
| Visibility / util | `.hidden`, `.hidden--mobile`/`.mobile:hidden`, `.hidden--desktop`/`.desktop:hidden`, `.visually-hidden`, `.hide-when-empty`, `.list-unstyled`, `.text-left/center/right` (logical), `.justify-*`, `.svg-wrapper`, `.icon-*`, `.flex`, `.contents`, `.relative`, `.flip-x`, `.wrap-text`, `.sticky-content(--desktop)` |
| Global behaviors | `*:focus-visible` outline; `.shopify-block:has(> [shopify-block-empty]) {display:none}`; view-transition rules (reduced-motion gated); `.page-wrapper` scroll container ≥990px; `@keyframes` block (`:1472-1682`, followed by resource-list, group-block, scrollbar and first-section-offset rules) |

### 9.5 Typography

- Seven presets: `paragraph`, `h1`–`h6`. Sizes from `type_size_*` settings: `<size/16>rem` below 48px; `clamp(min, size×0.1vw, max)` at ≥48px.
- Apply a preset by class (`class="{{ block_settings.type_preset }}"`, values `paragraph|h1…h6`; `rte` = rich text defaults) or by tokens (`font-size: var(--font-h3--size)`); custom typography via `typography-style` + classes.
- Font loading: `snippets/fonts.liquid` preloads; `@font-face` (regular/bold/italic/bold-italic, `font-display: swap`) in `theme-styles-variables`. Never load fonts elsewhere; use the four font roles.

### 9.6 Style owners (render them wherever their markup appears)

| Owner | Consumers |
|---|---|
| `product-card-styles` | `snippets/product-card` (renders it). Grid layout CSS belongs to `snippets/product-grid`, whose empty branch (`snippets/product-grid.liquid:105-109`) pre-renders `product-card-styles`, `text-block-styles`, `price-styles`, `product-card-gallery-zoom-details-styles` so later results arrive styled |
| `add-to-cart-button-styles` | `blocks/add-to-cart`, `snippets/quick-add`, `sections/product-information` (sticky ATC) |
| `buy-buttons-styles` | `blocks/buy-buttons`, `product-card`, `_product-card`, `accelerated-checkout`, `featured-product`, `product-hotspots` |
| `variant-picker-styles` | `variant-main-picker`, `variant-swatches`, `quick-add` (choose path), `list-filter`, card hosts |
| `quantity-selector-styles` | `quantity-selector` + card/recommendation hosts |
| `price-styles` | `snippets/price`, `unit-price`, `cart-products`, `quick-order-list`, `quick-add`, `product-grid`, `_featured-product-price` |
| `text-block-styles` | `snippets/text` + any file emitting `.text-block` directly |
| `swatch-styles`, `product-badges-styles`, `quick-add-styles`, `quick-add-modal-styles` | card hosts (`_product-card`, `featured-product`, `product-hotspots`, `product-recommendations` block + section); also swatch → `variant-swatches`, `variant-main-picker`, `filters`, `blocks/product-card`; badges → `_product-card-gallery`; quick-add → `snippets/quick-add`; quick-add-modal → `snippets/quick-add-modal` |
| `product-media-container-styles`, `product-media-gallery-content-styles` | both: `product-media-gallery-content`, `quick-add-modal` (rendered before content arrives); container only: `card-gallery` |
| `deferred-media-video-styles` | `snippets/video`, `product-information`, `featured-product-information` |
| `deferred-media-product-model-styles` | `snippets/product-media`, `snippets/quick-add`, `product-information`, `featured-product-information` |
| `dialog-styles` | files emitting `.dialog-modal`/`.dialog-drawer` (`popup-link`, `search-modal`, `quick-add-modal`, `cart-disclosure-tooltip`) |
| `accordion-styles`, `summary-icon-styles` | `accordion`, `disclosures`; `<summary>` icon emitters |
| `slideshow-styles`, `resource-list-styles`, `theme-drawer-styles`, `pills-styles`, `predictive-search-styles`, `checkbox-styles`, `fly-to-cart-styles`, `payment-terms-styles`, `video-background-styles`, `video-placeholder-styles`, `cart-typography-styles`, `collection-wrapper-styles`, `gift-card-recipient-form-styles`, `product-card-gallery-zoom-details-styles`, `product-recommendations-styles`, `text-component-shimmer-styles`, `card-hover-effect-styles` (layout), `password-layout-styles` (password layout) | their components |

Verify the paths that bypass composition: empty → populated grid after filtering, first open of Quick Add, media swaps, hydration responses.

### 9.7 Spacing system

- Padding settings → `spacing-style` (scaled >20px) or `spacing-padding` (raw). Text-family blocks, `page`, `payment-icons`, `follow-on-shop`, `_blog-post-description`, and sections `custom-liquid`, `header-announcements`, `slideshow`, `layered-slideshow` use unscaled padding.
- Gap → `gap-style` (scaled >24px) via `layout-panel-style` or direct.
- Section padding defaults are usually 0; first `<main>` section adds the transparent-header offset.

### 9.8 Writing CSS for a new `cx-` block

Precedents: `blocks/spacer.liquid` (modifier classes + namespaced vars + context selectors `.layout-panel-flex--row > …`), `blocks/image.liquid` (spacing/size/border helpers), `blocks/page.liquid` (contrast-override). Template: Appendix A.1.

### 9.9 Global CSS without editing `base.css`

Only for rules no setting, block stylesheet or instance style can express (site-wide overrides of stock components).
- Route A (recommended): `assets/cx-theme.css` + one line in `snippets/stylesheets.liquid` after `base.css`: `{{ 'cx-theme.css' | asset_url | stylesheet_tag }}` (L5 stock edit; covers theme, password and gift-card shells). It wins over `base.css` at equal specificity. It precedes the inline token `<style>` blocks, so overriding `:root` tokens needs higher specificity (e.g. `:root:root`). It loads before `{{ content_for_header }}` (`layout/theme.liquid:21` vs `:32`), which injects the compiled `{% stylesheet %}` bundle, so the bundle wins equal-specificity ties (PLATFORM [LCR]) → override component CSS with higher specificity or scoping, not source order.
- Route B: a `snippets/cx-<name>-styles.liquid` owner rendered from the layout (precedent: `card-hover-effect-styles`, `layout/theme.liquid:22`). Subject to subsetting; add to `layout/password.liquid` if needed.
- Settings-driven global values: a `{% style %}` rendered in `<head>` (pattern: `theme-styles-variables`).
- Never put global overrides in a block's `{% stylesheet %}` (only shipped when that block renders).

## 10. Color system

### 10.1 Model

Horizon 4.2.0 has **no color schemes** (`color_scheme` / `color_scheme_group` unused). Color comes from:
1. One global `color_palette` setting (`config/settings_schema.json:63-70`; schema default `background`, `foreground`; saved data adds `color1`–`color3`) and role settings (`page_background_color`, `page_text_color`, `palette_primary_button_*`, `palette_secondary_button_*`, `palette_input_*`, `palette_variant_*`, `palette_selected_variant_*`, `popover_shadow_color` / `popover_drop_shadow`) → `:root` tokens by `snippets/color-palette.liquid`. Drawer/popover/badge background, text and border colors feed the UI scopes (item 2) and `--style-border-popover/-drawer`; `quick_add_background` / `quick_add_text` feed `.custom-color-quick-add` (`snippets/theme-styles-variables.liquid:653`).
2. Four global UI scopes (`preset: 'ui'`): `.color-custom-popover`, `.color-custom-drawer`, `.color-custom-badge-sale`, `.color-custom-badge-sold-out` (`snippets/theme-styles-variables.liquid:659-687`).
3. Per-section/per-block `background_color`/`text_color` → `contrast-override` → `.color-custom-<id>` scope.
4. Per-block inline overrides (`--color` on text blocks, `button-custom-styles`, `input-custom-styles`, `variant-button-custom-styles`).

Palette references: a `color` setting default/value may be the string `"{{ settings.color_palette.<slot> }}"`; Shopify resolves it to the slot's current color before Liquid reads it; a deleted palette color is stored as a reference to its replacement (PLATFORM [IS]). Prefer palette references over hex in saved settings so palette edits propagate.

### 10.2 `contrast-override`

- Params: `background_color`, `text_color`, `border_color`, `section_id` (required, unique), `skip_contrast`, `preset` (`content` default | `ui`).
- No output when background blank/alpha 0 and no text color.
- Text color: explicit `text_color` → global `page_text_color` if contrast ≥ 4.5 → `--palette-darkest` (bg brightness >128) or `--palette-lightest`.
- With a background: emits on `.color-custom-<id>, .color-custom-<id> .text-block`: `--color-background(-rgb)`, brightness opacities, `--color(-rgb)`, `--color-foreground(-rgb)`, `--color-foreground-muted`, `--color-foreground-subdued`, `--color-border(-rgb)`; on `.color-custom-<id>`: `--input-disabled-*`, `--checkbox-border`, `color`, `background-color` (`snippets/contrast-override.liquid:104-144`).
- Text color only: `--color(-rgb)`, `--color-foreground(-rgb/-muted/-subdued)`, `--input-disabled-*`, `--checkbox-border`, `color`; no background tokens; `--color-border(-rgb)` only for `preset: 'ui'`; with `content` it also sets `color` on `.text-block`/footer/social descendants (`:145-197`).
- Never emits button or input chrome tokens: buttons and inputs keep global palette colors inside a custom-color scope.
- `skip_contrast: true` when the surface shows background image/video (only background tokens emitted).
- Caller contract: render it once per scope with a unique id; add `color-custom-<id>` to the element(s) only when a color is set (sections: both `.section-background` and `.section`). Stock `snippets/section.liquid:22-24` adds it unconditionally; harmless, because nothing is emitted without a color.

### 10.3 Writing color-aware CSS

- Follow the active color context: read `--color-*` tokens inherited from the enclosing scope (`:root` palette, `.color-custom-<id>`, UI scopes) instead of re-declaring them, so merchant-selected colors stay effective.
- Complete colors directly: `color: var(--color-foreground)`, `background: var(--color-background)`, `border-color: var(--color-border)` (preserves configured alpha).
- Text: `color: var(--color, var(--color-foreground))`.
- Explicit alpha: `rgb(var(--color-foreground-rgb) / var(--opacity-70))`. `-rgb` companions exist only for: background, foreground, border, palette extremes (`--palette-lightest/-darkest`), input text, variant text, selected variant text, popover/drawer shadow, white, `--color` (scoped), `--backdrop-color-rgb` (points to the never-valued `--color-shadow-rgb`, §19).
- Tints that must stay visible on dark custom backgrounds: brightness-aware `--opacity-40-60`, `--opacity-35-55`, `--opacity-10-25`.
- Muted text: `var(--color-foreground-muted)` / `var(--color-foreground-subdued)`.
- When overriding a color token, verify its `-rgb` companion exists and override both together; check consumers that use the complete color and consumers that use `rgb(var(--x-rgb) / alpha)` together.
- A custom scope that changes `--color-foreground-rgb` MUST re-declare derived tokens (`--color-foreground-muted/-subdued`, `--input-disabled-*`, `--checkbox-border`) — reuse `contrast-override` instead of hand-writing scopes.
- No hard-coded hex in component CSS. Verify actual contrast (4.5:1 text, 3:1 large text/UI).

## 11. Global settings (`config/settings_schema.json`)

17 groups after `theme_info`. Full definitions live in the file; tokens they drive are in §9.3/§10.1.

| Group (line) | Settings (IDs) |
|---|---|
| Logo and favicon (11) | `logo`, `logo_inverse`, `logo_height`, `logo_height_mobile`, `favicon` |
| Colors (60) | `color_palette` (only `color_palette` setting in the theme) |
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
| Search (1896) | `empty_state_collection`; search-popover only: `product_corner_radius`, `card_corner_radius`, `card_title_case` |
| Swatches (1947) | `show_variant_image`, `variant_swatch_{width,height,radius,border_style,border_width,border_opacity}` |
| Variant pickers (2026) | `palette_variant_*`, `palette_selected_variant_*`, `variant_button_border_width`, `variant_button_radius`, `variant_button_width` |

- `config/settings_data.json`: `{ "current": {…}, "presets": { "Default": {…} } }`; IDs absent from `current` use schema defaults. Edit `current` only; select values are strings. Theme style presets ≤5 (PLATFORM).
- Theme settings are shop-wide choices. Content or presentation of one band/component belongs in section/block settings.
- Adding a global setting = L5 stock edit of `config/settings_schema.json` (+ locale keys).
- Referenced but undefined in the schema (no effect in stock): `drawer_drop_shadow`, `drawer_shadow_color`, `social_twitter_link`, `button_text_case` (in `blocks/add-to-cart.liquid:20`); computed key `settings['--font-body--case']` (`snippets/theme-styles-variables.liquid:345`, always nil, so the paragraph case token is empty).

## 12. JavaScript

Principles: Liquid renders; JS enhances. Start with native links, buttons, forms, `details`, `dialog`, `popover`. Business logic, translations and money formatting stay server-side. No build step (`assets/package.json` = `{"type":"module"}`), no bundler, no runtime dependencies beyond stock (`@shopify/events` CDN module, local `popover-polyfill.js`, local `qr-code-generator.js`). Plain ES modules in flat `assets/`, typed with JSDoc (`assets/jsconfig.json`: `checkJs`, strict null checks; `@theme/*` → `./*`).

### 12.1 Loading

Three separate decisions:
1. **Resolution** — an import-map entry in `snippets/scripts.liquid:7-41` makes `@theme/<name>` resolve to an asset URL. Needed only if another module or an inline `<script type="module">` imports it.
2. **Early fetch** — `<link rel="modulepreload">` (stock preloads utilities, component, section-renderer, section-hydration, morph, focus, recently-viewed-products, scrolling, events; paginated-list* only on collection/search).
3. **Execution** — a `<script type="module" src=…>` (or an import reached from an executing module) runs it and registers its custom elements.

Import map (29 entries): `@theme/{overflow-list, component, dialog, events, view-event-elements, focus, morph, paginated-list, performance, product-form, recently-viewed-products, scrolling, section-renderer, section-hydration, money-formatting, utilities, variant-picker, variant-resolution, media-gallery, quick-add, paginated-list-aspect-ratio, component-cart-items, component-quantity-selector, comparison-slider, sticky-add-to-cart, fly-to-cart, scroll-container, theme-drawer}` + `@shopify/events` → `https://cdn.shopify.com/storefront/standard-events.js`. Always import with the bare `@theme/<name>` specifier, never a relative path.

Globally executed by `snippets/scripts.liquid`: `standard-actions-override`, `page-view-event`, `overflow-list`, `dialog`, `variant-picker`, `product-card`, `product-form`, `fly-to-cart`, `accordion-custom`, `media`, `product-price`, `product-sku`, `product-inventory`, `show-more`, `slideshow`, `layered-slideshow`, `anchored-popover`, `floating-panel`, `video-background`, `component-quantity-selector`, `media-gallery`, `rte-formatter`, `volume-pricing`, `price-per-item`, `volume-pricing-info`, `auto-close-details` (classic). Gated: `quick-add` (`quick_add or mobile_quick_add`), `cart-discount` (`show_add_discount_code`), `localization` (>1 country/language), `sticky-add-to-cart` + `RecentlyViewed.addProduct` (product template), `view-transitions` (inlined classic, transition settings).

Loaded by their owners (pattern for new blocks): `facets.js` + `scrolling.js` (`blocks/filters.liquid:13-23`), `local-pickup.js` (`blocks/buy-buttons.liquid:259-263`, gated), `product-custom-property.js` (`blocks/product-custom-property.liquid:207-211`), `marquee.js` (`blocks/_marquee.liquid:1-5`), `comparison-slider.js`, `product-recommendations.js`, `disclosures-summary-fit.js` (gated), `header.js`, `header-menu.js`, `product-hotspot.js`, `collection-links.js`, `announcement-bar.js` (gated `section.blocks.size > 1`), `results-list.js`, `blog-posts-list.js`, `quick-order-list.js`, `cart-drawer.js`, `cart-note.js` (gated `show_cart_note`), `zoom-dialog.js` (gated `zoom`), `drag-zoom-wrapper.js` (gated), `jumbo-text.js`, `predictive-search.js`, `theme-drawer.js`.

`Theme` global (inline, `snippets/scripts.liquid:284-312`): `Theme.routes` (`cart_add_url` (+`.js`), `cart_change_url`, `cart_update_url`, `cart_url`, `predictive_search_url`, `search_url`), `Theme.translations` (8 fixed keys), `Theme.template.name`, `Theme.supportsNativePopover`, `Theme.popoverPolyfillReady`. Use `Theme.routes` instead of hard-coded URLs. For new UI strings, pass translated text via `data-*` attributes rendered with `| t` (`blocks/product-custom-property.liquid:28-30`).

Adding a feature module:
1. Reuse an existing mapped module if it owns the behavior.
2. Put `<script src="{{ 'cx-name.js' | asset_url }}" type="module" fetchpriority="low"></script>` in the file that renders the element, gated by the same Liquid condition that renders the element; execute once.
3. Add an import-map entry only if something imports it (L5 edit of `snippets/scripts.liquid`).
4. If the element can first appear through a Section Rendering API morph (not a full page load), the module must already be on the page: full-mode `morph` re-creates only `.shopify-app-block script[src]`, not block scripts (`assets/morph.js:139-142`), and hydration mode re-creates none (it returns before that step, `:124-127`). Stock solves this by loading such modules globally.
5. Keep asset imports (`@theme/*` specifiers) and script placement (`type="module"`, `fetchpriority="low"`, Liquid gating) consistent with `snippets/scripts.liquid`.

### 12.2 `Component` (`assets/component.js`)

```js
import { Component } from '@theme/component';

/** @typedef {{ toggle: HTMLButtonElement, items?: HTMLElement[] }} Refs */

/** @extends {Component<Refs>} */
class CxExampleComponent extends Component {
  requiredRefs = ['toggle']; // only for fixed internal markup; throws MissingRefError otherwise

  connectedCallback() {
    super.connectedCallback(); // refs and on: handlers are ready after this
  }

  updatedCallback() {
    super.updatedCallback(); // refresh state after a section re-render
  }

  disconnectedCallback() {
    super.disconnectedCallback(); // then clean up everything this class created
  }

  /** @param {MouseEvent} event */
  handleToggle(event) {
    event.preventDefault();
    const { toggle } = this.refs; // destructure at use time; refs are replaced on refresh
    toggle.setAttribute('aria-expanded', String(toggle.getAttribute('aria-expanded') !== 'true'));
  }
}

if (!customElements.get('cx-example-component')) {
  customElements.define('cx-example-component', CxExampleComponent);
}
```

Validated: `validate_theme` pass and `tsc` clean under `assets/jsconfig.json` (`checkJs`, `noImplicitAny`, strict null checks), 2026-10-10.

- `ref="name"` → `this.refs.name`; `ref="name[]"` → array in DOM order. `this.refs` is replaced on every refresh: never cache it.
- Ref scope stops at the nearest nested component: any `Component` instance **or any tag ending in `-component`** (even before upgrade). Refs inside a nested `*-component` belong to it. Name custom elements `*-component`.
- Refs refresh synchronously in `connectedCallback`, asynchronously via a `MutationObserver` (watching `ref` attributes and child changes) that is attached only once the browser is idle after connect (mutations before that are missed), and synchronously in `updatedCallback()`. Treat merchant-dependent/conditional refs as optional (`[name]` in the typedef) and check before use; `requiredRefs` only for fixed markup.
- Declarative events: one capture-phase listener per event type on `document`. Supported: `click, change, select, focus, blur, submit, input, keydown, keyup, toggle, pointerdown, pointerenter, pointerleave`.

| Attribute | Target |
|---|---|
| `on:click="/method"` | closest component (incl. the element itself) |
| `on:click="/method/arg"` | same; `arg` coerced (`true`/`false`/numbers), passed **first**, event last |
| `on:click="/method?a=1&b=x"` | same; argument is an object |
| `on:click="selector/method"` | `element.closest(selector)` (the element itself or an ancestor; any CSS selector without `/` or `?`) |
| `on:click="#id/method"` | `document.querySelector('#id')` (global) |

Argument values cannot contain `/` or `?`. Non-bubbling `toggle` must sit on the `<details>`; `pointerenter`/`pointerleave` match only the exact target. Handlers are public methods; document their signatures with JSDoc (`@param {MouseEvent} event`, data argument first). Synchronous throws are logged, async rejections are not caught.
- Lifecycle (MUST): call `super.connectedCallback()` before using refs; `super.updatedCallback()` before reading refs after DOM changes; `super.disconnectedCallback()`. The base class disconnects only its own `MutationObserver` (`assets/component.js:102-104`). Clean up every listener, timer, observer, rAF, media-query listener and in-flight request; use an `AbortController` re-created on each connect (`assets/disclosures-summary-fit.js:48-50,82-90`). Reconnects and section re-renders must be idempotent.
- Morph skips `isEqualNode`-equal subtrees, so `updatedCallback` does not fire for them; check render completion separately (await the render promise, then inspect the live DOM).
- `morphSection` parses the response with `DOMParser` (`assets/section-renderer.js:242`); `morph` passes that parsed node to `onAfterUpdate` (`assets/morph.js:270`), which calls `updatedCallback` only when `node instanceof Component` (`:97-100`). Elements in a `DOMParser` document are not upgraded custom elements (browser behavior), so treat `updatedCallback` as not fired by section morphs. Keep state derivable on connect; stock calls `updatedCallback` by hand where needed (`assets/volume-pricing-info.js:22-27,42`).
- Communication: downward via public methods; upward via bubbling events; never depend on parent tag names (except declared `on:` targets).
- Plain `HTMLElement` subclasses exist on purpose where refs must stay visible to a parent (`accordion-custom`, `scroll-hint`). `DeclarativeShadowElement` is used by `overflow-list`. Keep these as they are; do not convert them to `Component`.
- JS style: `async`/`await`, early returns, `for…of`, `URL`/`URLSearchParams`, validate event targets before type-specific APIs.

### 12.3 Events

`@shopify/events` (`StandardEvents`), for cart, product selection, search, collection:

| Event | Name | Producers → consumers |
|---|---|---|
| `ProductSelectEvent` | `shopify:product:select` | `variant-picker.js` → product-form, product-price, product-inventory, product-sku, media-gallery, sticky-add-to-cart, local-pickup, product-card, quick-add |
| `CartLinesUpdateEvent` | `shopify:cart:lines-update` | product-form, component-cart-items, quick-order-list → cart-icon, cart-drawer, cart items, header-actions, sticky ATC, quick-add, price-per-item, product-form, gift-card recipient form, quick-order-list |
| `CartErrorEvent` | `shopify:cart:error` | cart producers → sticky ATC, gift-card form |
| `CartNoteUpdateEvent`, `CartDiscountUpdateEvent` | `shopify:cart:note-update`, `…:discount-update` | cart-note, cart-discount → cart items |
| `SearchUpdateEvent`, `CollectionUpdateEvent` | `shopify:search:update`, `shopify:collection:update` | facets, predictive-search → paginated-list, facets |
| `PageViewEvent`, `ProductViewEvent`, `CollectionViewEvent`, `CartViewEvent` | `shopify:*:view` | `PageViewEvent`: `assets/page-view-event.js` on DOM ready. Product/Collection/CartViewEvent: view-event elements `product-component`, `collection-component`, `cart-items-component` carrying `view-event-payload="{{ x \| standard_event_data: 'view' \| escape }}"` |

Before producing or consuming an event, read its definition, its producers and its consumers together. Producer pattern: `const { promise, resolve, reject } = XEvent.createPromise()`; dispatch `new XEvent({…, promise})` on `this` **before** the request; resolve/reject after. Consumer pattern: filter by `event.target`; `event.promise.then(({ detail }) => …).catch((e) => { if (e?.name !== 'AbortError') console.warn(e) })`. Listen on `this.closest('.shopify-section, dialog')` for product events (ignore events from `product-card` unless intended), on `document` for cart events.

`@theme/events` (`assets/events.js`, all bubble): `quantity-selector:update` (`{quantity, cartLine}`), `media:started-playing`, `megaMenu:hover`, `zoom-media:selected`, `cart-section:restored`, `slideshow:select`. Elsewhere: `dialog:open`/`dialog:close` (**non-bubbling**, dispatched on `dialog-component`), `theme-drawer:open`/`:close` (bubbling), `overflowMinimum`, `reflow`, `recipient:toggle` (bubbling, `{mode, recipientFormVisible}`, `assets/gift-card-recipient-form.js`). New theme-internal events: `class CxThingEvent extends Event` with `bubbles: true` and a `detail`, in the style of `events.js`.

`assets/standard-actions-override.js` configures `Shopify.actions.updateCart` (event target `theme-drawer#cart-drawer`, else `document`) and `openCart` (opens the drawer, else goes to `Theme.routes.cart_url`); `getCart` is not overridden. Stock components POST to the cart themselves with `fetchConfig()` + `Theme.routes` (`/cart/add.js`; `/cart/change` and `/cart/update` with JSON).

### 12.4 Section Rendering, morph, hydration

- `sectionRenderer.renderSection(sectionId, { cache = !Shopify.designMode, mode = 'full' | 'hydration', injectStylesheet = false, url, shouldRender })` → fetches `?section_id=` (or, when `cache` is true — the storefront default — reuses HTML already cached for that URL with no request; the cache is also seeded from live sections at `load`; pass `cache: false` for fresh server HTML) and morphs `#shopify-section-<id>`; returns the HTML. In 4.2.0 it awaits the selected morph and propagates its failures; fetch failures and missing-wrapper failures reject — always `await` and handle rejection. Aborts any in-flight render for the same section; a superseded call settles with the newer render's promise (including its rejection), resolves to `''` (aborted with no newer render, e.g. `abortRender`), or resolves to its own unapplied HTML (abort landed after its fetch/cache read), so re-check current user intent before acting on the result. Does not check HTTP status. Pass unprefixed section ids; live DOM and response both need the `shopify-section-<id>` wrapper.
- After skipped or superseded renders, verify the intended live DOM and component readiness separately from the promise.
- `sectionRenderer.getSectionHTML(id, useCache, url, signal)`, `abortRender(id)`.
- `morphSection(sectionId, html, { mode, injectStylesheet })` throws if `#shopify-section-<id>` is missing in DOM or response; await it and handle rejection. Direct calls have no stale-update guard: coordinate competing requests yourself.
- `morph(old, new, options)`: keyed by `id` (or `getNodeKey`); preserves live `open` on `details`/`dialog` (unless the new node has `declarative-open`) and `slot`/`sizes`; never morphs `SHOPIFY-ACCELERATED-CHECKOUT-CART`. Opt-outs: `data-skip-node-update` on both nodes (keep node attrs and form state, recurse); `data-skip-subtree-update` on both nodes (skip children); `data-skip-value-update` on the live node (default `MORPH_OPTIONS` copy it to the new node) keeps the typed value only while the live input is focused and the incoming input is not disabled. Without it, focused input values are overwritten.
- Hydration mode morphs only elements with matching non-empty `data-hydration-key` (stable, unique, identical in initial and fetched markup); unmatched targets are never independently inserted or removed, but a matched target's attributes and descendants can change. Choose key boundaries that preserve intended state, focus and refs; use full mode for structural changes outside matched targets.
- `hydrate(sectionId[, url])` from `@theme/section-hydration`: waits for DOM ready + idle, renders with `cache: false, mode: 'hydration'`, sets `data-hydrated="true"`; its promise covers scheduling only. Stock `hydrate()` user: header only (`sections/header.liquid:1235-1242`, not in design mode). Other hydration-mode users call `morphSection` directly: product recommendations (`assets/product-recommendations.js:100`, `{mode:'hydration', injectStylesheet:true}`) and the cart drawer (`assets/component-cart-items.js:104,128`, `mode: this.isDrawer ? 'hydration' : 'full'`).
- CSS in partial responses: full-section insertion keeps nested response CSS naturally; for hydration/fragment updates the response stylesheet must reach the live DOM. Opt-in `injectStylesheet` (default `false`) takes the first `style[data-section-stylesheet]` in the response and updates the live section's style text or prepends it; if the response has none it is a no-op. Use `morphSection(id, html, { mode: 'hydration', injectStylesheet: true })` or `sectionRenderer.renderSection(id, { mode: 'hydration', injectStylesheet: true })`. The deferred `hydrate()` helper does **not** opt in: keep early style-owner renders, or do an explicit update with injection when it needs missing response CSS.
- `hydrate()` accepts either id form (`<id>` or `shopify-section-<id>`), skips missing or already hydrated sections, and its promise covers scheduling only (not completion or render failures): coordinate dependent work explicitly.
- Cancel stale requests; never let an older response overwrite newer user intent; update the smallest DOM region.
- Max 5 sections per Section Rendering API request (PLATFORM [SRA]).

### 12.5 Theme editor in JS

- `assets/theme-editor.js` (design mode only) handles `shopify:block:select`/`deselect` (slideshow select/pause, carousel scroll, product-card navigation off) and `shopify:section:load`/`unload` (header CSS vars); persists overlay open state across editor refreshes. Editor events bubble; `event.target` is the section/block element.
- Components rely on custom-element connect/disconnect when the editor replaces section DOM: make connect/disconnect idempotent. Use `Shopify.designMode` / `request.design_mode` only for editor affordances, never to change shopper-facing behavior. `oncePerEditorSession(el, key, cb)` (`@theme/utilities`) for intro animations.
- A new block that must react to selection (e.g. open a dialog when selected) needs a `theme-editor.js`-style listener: put it in the block's own module (listen on `document` for `shopify:block:select` with `event.target` inside the component) instead of editing `theme-editor.js`.

### 12.6 Utilities worth reusing

| Module | Exports |
|---|---|
| `@theme/utilities` | `debounce`, `throttle`, `requestIdleCallback`, `yieldToMainThread`, `fetchConfig(type, {body, headers})`, `prefersReducedMotion()`, `mediaQueryLarge`, `isMobileBreakpoint()`, `isDesktopBreakpoint()`, `isTouchDevice()`, `isRTL()`, `getDirectionMultiplier()`, `lockScroll(owner)`/`unlockScroll(owner)`, `isClickedOutside`, `onDocumentReady`, `onDocumentLoaded`, `onAnimationEnd`, `startViewTransition(cb, types)`, `viewTransition`, `preloadImage`, `getViewParameterValue`, `oncePerEditorSession`, `ResizeNotifier`, `scheduler`, header helpers |
| `@theme/scroll-container` | `getScrollContainer`, `getScrollTop`, `scrollTo`, `getScrollEventTarget()` (`.page-wrapper` ≥990px or `document`), `getIntersectionRoot()`; importing it patches `history.pushState` |
| `@theme/focus` | `trapFocus(container)` (one active trap globally), `removeTrapFocus()`, `cycleFocus()` |
| `@theme/dialog` | `dialog-component` (`showDialog`, `closeDialog`, `toggleDialog`; scroll lock; Escape/outside click) |
| `@theme/theme-drawer` | `theme-drawer` (`open`, `close`, `toggle`; modal <990px, inline + squeeze ≥990px; `no-persist` attr) |
| `@theme/scrolling` | `Scroller`, `scrollIntoView` (RTL-aware) |
| `@theme/morph`, `@theme/section-renderer`, `@theme/section-hydration` | §12.4 |
| `@theme/money-formatting` | `formatMoney`, `convertMoneyToMinorUnits` (prefer server-rendered money) |
| `@theme/performance` | `cartPerformance` (an instance of the unexported `ThemePerformance`: `createStartingMarker`, `measureFromEvent`, `measureFromMarker`, `measure`) |
| Non-mapped globals | `anchored-popover-component` (`ref="trigger"`, `ref="popover"`, hover/anchor positioning), `floating-panel-component`, `accordion-custom`, `slideshow-component`, `show-more-component` — reuse via markup. `marquee-component` is loaded by its owners (`blocks/_marquee.liquid:1-5`, `sections/marquee.liquid`): include its script when reusing it |

Reuse an existing component through markup before writing a new one.

## 13. Commerce surfaces

### 13.1 Product page

```
templates/product.json
  main: product-information
    static media-gallery: _product-media-gallery
    static product-details: _product-details   (accepts @theme, @app + product blocks, _divider)
       group → text "<h1>{{ closest.product.title }}</h1>", price
       _divider, variant-picker, buy-buttons (static quantity / add-to-cart / accelerated-checkout), text "{{ closest.product.description }}"
    dynamic (section area, rendered after the product grid): disclosures
  product_recommendations: product-recommendations (product "{{ closest.product }}", related)
```

- `sections/product-information.liquid`: JSON-LD `closest.product | structured_data`; passes `closest.product: closest.product` to both statics; sticky add-to-cart bar when `enable_sticky_add_to_cart` (default true; markup uses the global `product`); layout via `snippets/product-information-content.liquid` (shared with `featured-product-information`). `presets: []` (template-only).
- `[data-product-grid-content]` (the `<product-component>` wrapping media + details) is what Quick Add clones; section-level blocks (e.g. `disclosures`) are outside it.
- `_product-details` accepts `@theme` → any public block works in the details column; outside a product context `closest.product` is blank.
- Media: `_product-media-gallery` (grid|carousel, columns, thumbnails, zoom, `hide_variants`, aspect ratio, `constrain_to_viewport`) → `snippets/product-media-gallery-content.liquid` → `snippets/product-media.liquid`; `<media-gallery>` replaces itself from the variant response.

Product form contract (`blocks/buy-buttons.liquid` + `assets/product-form.js`) — MUST be preserved when editing product UI:
- Form id `BuyButtons-ProductForm-{{ section.id }}` → **one `buy-buttons` per section**. `{% form 'product', product, id: …, data-type: 'add-to-cart-form' %}` with hidden `<input name="id" ref="variantId">` = selected/first-available variant.
- Static children: `quantity` (`name="quantity"`, min/max/step from `quantity_rule`), `add-to-cart` (`<add-to-cart-component>`, `button[type=submit][name=add][ref=addToCartButton]`), `accelerated-checkout` (`{{ form | payment_button }}`; renders nothing without the form passed in).
- Availability computed in Liquid (`can_add_to_cart`, Add to cart / Sold out / Unavailable incl. quantity rules + deny policy); volume pricing table; gift-card recipient form; local pickup drawer.
- Line-item properties: `product-custom-property` inputs live outside the form, bound by `form="BuyButtons-ProductForm-<section.id>"` and `name="properties[<key>]"`. New buyer inputs follow the same pattern.
- Submit: `/cart/add.js` with `sections` for every `cart-items-component`; dispatches `CartLinesUpdateEvent` (promise) before the request; errors → `CartErrorEvent` + live region.
- No selling-plan (subscription) selector exists in the theme. Subscriptions require an `@app` block; Quick Add routes selling-plan products to the product page.
- Keep: variant id input, availability states, quantity rules, accelerated checkout, `type=submit` semantics, `ref`s and `on:` attributes. Restyle freely through settings/CSS.

Variant change flow (`assets/variant-picker.js`):
- On the product page the **whole product page** is fetched (`<url>?option_values=<ids>`); `section_id` is used only inside `featured-product-information` and card swatches (`section-rendering-product-card`); the Quick Add modal picker fetches the full product page. Combined-listing product switches morph all of `<main>` (`assets/variant-picker.js:361-362`).
- `ProductSelectEvent` → updates: product-form (variant id, button state, quantity rules, volume pricing), `product-price` (by `data-block-id`), `product-inventory`, `product-sku-component`, `media-gallery`, sticky bar, local pickup, URL `?variant=`.
- **Not refreshed on variant change** (on the product page; `featured-product-information` morphs its whole element and combined-listing switches morph `<main>`, so there everything refreshes): `text`, `product-title`, `review`, `product-description`, `custom-liquid`, `image`, `disclosures`, `product-custom-property`, app blocks. Variant-level dynamic sources in these blocks stay at the initial variant. Variant-reactive custom UI: a `cx-` component listening for `StandardEvents.productSelect` on `closest('.shopify-section, dialog')`, awaiting `event.promise`, and replacing its node from `detail.html` by a stable `data-block-id` (pattern: `assets/product-price.js:21-88`). Keep one instance per section of response-matched elements (`media-gallery`, `product-inventory`, `[ref=addToCartButton]`).

### 13.2 Featured product sections

| Section | Use | Notes |
|---|---|---|
| `featured-product-information` (Featured product) | Full product-page layout for a chosen product on any template | static `_featured-product-information-carousel` + `_product-details` (accepts `@theme`); variant changes re-render it via `section_id`; no sticky bar; `product-inventory`'s own listener ignores the event outside the matching product page (its `data-product-id` comes from the global `product`), so it is updated only by the whole-section morph |
| `featured-product` (Product highlight) | Media + card for one product | static only; `_featured-product` → static `product-title`, `_featured-product-price`, `_featured-product-gallery` (+ quick add), `swatches`; no `@app` |
| `product-hotspots` | Image with product hotspots | `_hotspot-product` (own `product` setting, x/y %); dialog with price + quick add |

### 13.3 Product card system

| Host (static `_product-card`) | id | `product_view_context` |
|---|---|---|
| `sections/main-collection.liquid:70-75` | `product-card` | collection |
| `sections/search-results.liquid:56-61` | `product-card` | search |
| `sections/product-list.liquid:37-42` (+ placeholder `:55`) | `static-product-card` | collection |
| `sections/product-recommendations.liquid:100-105` | `static-product-card` | recommendation |
| `blocks/product-recommendations.liquid:99-103` | `static-product-card` | — |

- One card configuration per host section instance; edited in that section's static `_product-card` (e.g. `templates/collection.json:122-215`).
- `_product-card` children: `_product-card-gallery`, `product-title`, `price`, `text`, `image`, `review`, `sku`, `swatches`, `buy-buttons`, `custom-liquid`, `_product-card-group` (nested group), `@app`. **Not allowed:** `variant-picker`, `product-inventory`, `product-description`, `quantity`, `product-custom-property`, `button`, `icon`. Public `product-card` (own `product` setting, addable anywhere) = same list minus `buy-buttons`.
- `buy-buttons` in cards: allowed only in `_product-card`; produces duplicate form ids per section (stock behavior).
- Badges: only sale (`compare_at_price > price`) and sold-out (`!available`), styled by global badge settings, rendered by `_product-card-gallery`. **No badge block exists.** Custom badges: a `text` block (with `background`, `background_color`, `corner_radius`) bound to a product metafield via dynamic source inside the static `_product-card` (L2/L3), or a `cx-` block (L6) if logic is needed.
- Card data: `text` blocks with dynamic sources (`{{ closest.product.metafields.ns.key }}`, `{{ closest.product.vendor }}`); `closest.product` is supplied per card (`blocks/_product-card.liquid:10`).
- Global card settings: `quick_add`, `mobile_quick_add`, `show_second_image_on_hover`, `product_card_carousel`, badges, `card_hover_effect`, `transition_to_main_product`, `currency_code_enabled_product_cards`.
- `snippets/product-card.liquid` renders `product-card-styles`; `snippets/card-gallery.liquid` handles media, carousel, hover image, eager loading when `section.index` is nil or `< 5` (`snippets/card-gallery.liquid:102`), `sizes` from host `section.settings` (falls back safely).
- Reusing `_product-card` in a `cx-` section: render it statically in the product loop with `closest.product: product`; provide `columns`/`layout_type` (or `product_card_size`) section settings if you want accurate image `sizes`; render the card style owners the stock hosts render (`quick-add-styles`, `quick-add-modal-styles`, `swatch-styles`, `quantity-selector-styles`, `variant-picker-styles`, `buy-buttons-styles`, `product-badges-styles` — see `sections/featured-product.liquid:13-19`).

### 13.4 Collection and search

- `templates/collection.json`: generic `section` (title + description `text` blocks bound to `closest.collection`) → `main-collection` (`enabled_on: collection`, static `filters` + `_product-card`).
- `main-collection`: `<collection-component>` → `<results-list infinite-scroll>`; 24 per page with infinite scroll (`enable_infinite_scroll`, default on); when off, `products_per_page` (8–36, default 24) with numbered pagination (`sections/main-collection.liquid:54-58`, `sections/search-results.liquid:30-34`); `layout_type` grid|organic, `product_card_size`, `product_grid_width`, gaps.
- `filters` block (static): horizontal|vertical, drawer on mobile (`<theme-drawer id=filters-drawer>`), sorting, grid density, price filter, list/swatch filters. Which filters exist = Search & Discovery app (L0). Flow: `assets/facets.js` → URL params → `sectionRenderer.renderSection(sectionId)` (inside a `product-grid` view transition unless in a dialog) and, immediately, `CollectionUpdateEvent`/`SearchUpdateEvent` carrying the render promise.
- Pagination: `assets/paginated-list.js` (prefetch ±1 page via `getSectionHTML`, infinite scroll sentinels, `?page=N` in history).
- `snippets/product-grid.liquid` empty state pre-renders card style owners so morph-in cards are styled.
- Search: `search-header` (static `_heading`, `_search-input`) + `search-results` (filters only when results > 0; empty state uses `settings.empty_state_collection`). Predictive search: `snippets/search-modal.liquid` → `sections/predictive-search.liquid` (SRA) via `assets/predictive-search.js`.
- Collection lists: `collection-list` (cards, 4 layouts), `main-collection-list` (all collections, max 20), `collection-links` (titles + spotlight images).

### 13.5 Cart

- `settings.cart_type` page|drawer. Drawer = `snippets/cart-drawer.liquid` (rendered by layout; SRA target `sections/cart-drawer-section.liquid`) — **snippet-built, not block-configurable**. Change drawer content via cart settings or an L5 snippet edit.
- Cart page = `main-cart` (static `_cart-title`, `_cart-products`, `_cart-summary`; dynamic area accepts `@theme`, `@app`, text/icon/image/button/video/group/spacer) + `product-list`.
- `snippets/cart-products.liquid`: `form#cart-form` (`/cart`, POST), line items (properties with `_` prefix hidden, selling plan name, discounts), `quantity-selector` with `name="updates[]"`; `snippets/cart-summary.liquid`: discounts (`cart-discount-component`), note (`cart-note`), totals, installments, checkout button `form="cart-form"`, `content_for_additional_checkout_buttons`.
- JS: `CartItemsComponent` (`assets/component-cart-items.js`) updates via `Theme.routes.cart_change_url` (`/cart/change`, JSON) with `sections` and morphs (`hydration` mode in the drawer); listens to `CartLinesUpdateEvent`, `quantity-selector:update`, note/discount events, `pageshow` (bfcache). `cart-drawer` auto-opens on add when `auto_open_cart_drawer`.
- Custom code that changes the cart MUST dispatch `CartLinesUpdateEvent` with a promise resolved to `{ cart, detail: { sections, source, sourceId, itemCount, didError } }` so the drawer, icon and cart sections update (pattern: `assets/product-form.js:429-578`), and request `sections` for every `cart-items-component` section id.

### 13.6 Quick add

- Enabled by `settings.quick_add` / `mobile_quick_add` (loads `quick-add.js`, renders `quick-add-modal`). `product-hotspot.js` imports `@theme/quick-add`, so hotspots load it regardless.
- `snippets/quick-add.liquid`: single variant → "Add" (inline product form `QuickAdd-ProductForm-<product.id>-<block.id>`); single option → "Choose", plus "Add" when the selected variant is available or the option has swatches (`snippets/quick-add.liquid:35-55`; `assets/quick-add.js` flips them on swatch select); multiple options → "Choose" (opens modal); selling plans → "Choose" that navigates to the product page.
- The modal fetches the **product's own product page** and clones `[data-product-grid-content]` (media + `_product-details`). Desktop CSS hides everything in details except groups, buy buttons, the title link, `variant-picker`, `product-price`, `product-inventory`; mobile rebuilds title + price + picker + form. A **required** `product-custom-property`: on desktop it is hidden by CSS (`snippets/quick-add-modal-styles.liquid`) and `checkValidity()` blocks submission (`assets/product-form.js:73`); on mobile the details column is removed (`assets/quick-add.js`), so the property is silently dropped. Avoid `required` when Quick Add is on, or route those products to the product page.

### 13.7 Metafields and dynamic sources

- Prefer binding stock blocks to metafields with dynamic sources (L2): `text` (`{{ closest.product.metafields.ns.key }}`), `image` (file reference metafield), `button` link, `video`, etc. Requires metafield definitions (L0).
- Hard-coded metafield readers: `review` (`reviews.rating`, `reviews.rating_count`), `disclosures` (`shopify.disclosure`), cart disclosure tooltip.
- `custom-liquid` can print metafields when a dynamic source cannot express the formatting (L4); a `cx-` block when it needs settings/CSS (L6).
- Metaobject-driven content: `metaobject` / `metaobject_list` settings (custom types must exist on the shop and be storefront-visible) in a `cx-` block/section, or metaobject templates (`templates/metaobject/<type>.json`) for metaobject pages.

## 14. Strings and translations

| Kind | Where | How |
|---|---|---|
| Storefront UI text (buttons, states, errors, empty states, accessible labels, live-region messages) | `locales/en.default.json` (+ 33 other storefront locales) | `{{ 'group.key' \| t }}`; interpolation at the call site `\| t: count: n`; pluralization keys `one`/`other`; `_html` suffix for intentional markup |
| Editor labels, info, option labels, names, preset names/categories, translatable defaults | `locales/en.default.schema.json` (+ 22 other schema locales) | literal `"t:group.key"` in schema JSON |
| Shop content (headlines, copy, media, links) | section/block settings, resources, metafields | never in locale files; translation-seeded setting defaults become merchant content once saved |

- Schema namespaces: `names`, `settings`, `options`, `content` (headers/paragraphs), `info`, `categories`, `html_defaults` (richtext defaults), `text_defaults`. Reuse stock keys for standard settings (`t:settings.top`, `t:settings.background_color`, `t:options.h2`, `t:content.padding`, `t:settings.default` placeholder, …).
- Custom keys: top-level `cx` object, `cx.<component>.<key>` (≤3 levels), in **both** `en.default.json` (storefront) and `en.default.schema.json` (editor) as needed. Locale files are stock files: adding the `cx` object is an L8 stock edit; record it; re-apply after a Horizon update.
- Other languages: add the same `cx` keys to each storefront locale the shop publishes; whether missing schema-locale keys fall back to English is UNVERIFIED (keep editor labels in `en.default.schema.json` at minimum).
- Keep interpolation at the translation call site. Stock keys are shared contracts used by several files: do not change their text, placeholders or meaning; add a `cx.*` key instead.
- Keep plain-text keys plain; rich-text keys (`_html`) only for intentional markup. Escape plain merchant strings in HTML/attributes; output `richtext`/`inline_richtext` as-is.
- Theme Check errors on missing keys (`TranslationKeyExists`, `ValidSchemaTranslations`); `validate_theme` reports them (§17.2).
- `en.default.json` and `en.default.schema.json` contain a `/* */` header and `//` comments (not strict JSON); keep valid structure. Limits: ≤3,400 translations per file, ≤1,000 chars per value (PLATFORM [LOC]).

## 15. Accessibility (applies to every component and every settings combination)

- Semantic HTML and native keyboard behavior first; ARIA only for names, relationships, states or announcements native semantics cannot provide.
- Every control has an accessible name; labels, relationships (`aria-controls`, `aria-describedby`), states (`aria-expanded`, `aria-current`, `aria-pressed`) and errors stay in sync.
- Visible focus (`*:focus-visible` outline in base.css), logical tab order, Escape closes overlays, modal focus containment (`trapFocus` / `dialog`), focus restoration after close or DOM replacement.
- Visual order = DOM reading order = keyboard order (no CSS reordering of meaningful content).
- One scoped live-region owner per independently updated status; route updates through it (stock: `[ref=liveRegion][aria-live=assertive]` in buy buttons, `role=status` in header actions).
- Images: informative → useful `alt`; decorative → `alt=""`; decorative SVG icons `aria-hidden="true"` + `focusable="false"`.
- Contrast for merchant-selectable colors: 4.5:1 text, 3:1 large text/UI/focus; use `contrast-override`.
- Touch targets ≥ 44×44px on primary controls and links (PLATFORM [A11Y]; token `--minimum-touch-target`); 48×48 recommended (PLATFORM [TAP]); WCAG 2.2 AA floor 24×24.
- Verify zoom (200%), wrapping, reduced motion, RTL, and that navigation, forms, product selection and purchase work while JS loads and without enhancement.
- Headings: one `h1` per page; stock sets heading levels in richtext content (`<h1>{{ closest.product.title }}</h1>`), style via presets — keep levels logical when composing.

## 16. Performance

- Plan loading around three moments: when markup appears, when the browser discovers each resource, and when the page first needs it.
- Render static presentation in Liquid; JS only for interaction.
- Images: `image_url | image_tag` with intrinsic `width`/`height`, `widths` (4–6 values) and `sizes` matching the rendered layout. Lazy-load below-fold/secondary media; for the LCP image: eager + `fetchpriority: 'high'` (stock: `fetchpriority` high when `section.index == 1`, `sections/hero.liquid:108-112`; the hero passes no `loading` and relies on `image_tag`'s default), never lazy, no CSS background-image for LCP, `<picture>` for art direction. `sizes: 'auto'` only with `loading: 'lazy'`. `section.index` is nil in the editor/SRA/static sections → nil-safe conditions (`snippets/background-media.liquid:33-39`). Other media: reserve space (intrinsic dimensions or aspect ratio). Base eager/preload/high-priority decisions on verified section position and visible layout, and recheck them whenever composition changes (a section moved to the top, a hero removed).
- Load component CSS/JS only where their markup or setting can activate them; shared resources once. Before deferred content appears (empty grids, Quick Add, hydration), ensure its CSS is present (§9.6, §12.4).
- Liquid: `for` loops ≤50 items → `paginate`; avoid nested loops and metafield access in loops; `assign` repeated filter results and `block.settings`; limit fetched items.
- Cancel stale requests; update the smallest DOM region; preserve section-rendering and hydration contracts.
- Test long text, missing media, large collections, sold-out products, slow network.

## 17. Verification

### 17.1 Checklist (every change)

1. Read changed Liquid, schema, locale, CSS and JS together: IDs, settings, refs, selectors, events, loading paths agree.
2. `validate_theme` (shopify-dev-mcp) on every created/changed file: first `learn_shopify_api(api: "liquid")` for a `conversationId`, then `validate_theme(conversationId, absoluteThemePath, filesCreatedOrUpdated: [{path}])` with paths relative to the theme root. Fix errors and warnings; re-run with the same `artifactId` and incremented `revision`. `validate_theme` reads local files only: validate a local theme directory that contains the changed files (for live-only edits, a scratch copy of the theme root plus the changed files and any locale keys they reference).
3. Compare any finding on stock files with the baseline (§17.2) before "fixing". `validate_theme` runs Shopify Theme Check; where Shopify CLI is permitted, `shopify theme check` is the equivalent full-theme run. Resolve valid findings in the context of the platform features the theme uses.
4. Preview the affected templates: desktop + narrow viewport; theme editor (add, remove, reorder, select the block; change every setting; empty and populated states); after interactions (filtering, variant change, first Quick Add open, cart drawer open, section re-render); non-default locale / RTL when relevant.
5. Interactions: keyboard and pointer paths, focus movement, live feedback, reduced motion, delayed module loading, request failure, repeated section rendering.
6. Record the change (§2.4). Run every applicable check that is available and resolve valid findings. Report each unavailable item (schema generation, validation, preview, store authentication, representative shop data) with the exact command or inspection a maintainer should run next.

### 17.2 `validate_theme` baseline for stock 4.2.0 (run 2026-10-10)

Scope: 420 files (sections, blocks, snippets, layout, templates, config, `en.default*` locales, assets JS/CSS/SVG). Result: 415 pass, 1 passes with 2 warnings (`snippets/bento-grid.liquid`), 4 fail with static-block-id errors. Re-run 2026-10-10 with the same result. Known stock findings — **do not "fix"**:

| File | Finding | Why it is intentional |
|---|---|---|
| `sections/header.liquid` | "The id 'header-menu' is already being used by another static block" ×2 | one static block rendered as 3 variants (`:86-94`) |
| `sections/main-cart.liquid` | same for `cart-page-title`, `cart-page-items` | same blocks rendered twice on every request: empty-cart copy inside `<template id="empty-cart-template">` with `force_empty: true` (`:26,29`) and live copy (`:45,49`) |
| `sections/product-list.liquid` | same for `static-product-card` | populated loop vs placeholder loop (`:37-42` vs `:55`) |
| `sections/collection-links.liquid` | same for `link` | text links for every layout, one per collection (`:56`), plus image-only copies for the spotlight slideshow (`:79`, `image_only: true`) |
| `snippets/bento-grid.liquid` | 2 warnings: opening/closing tag in different Liquid conditions | bento boxes (≤12 items) opened on `forloop.first or item_modulo == 1` (`:41`), closed on `forloop.last or item_modulo == 0` (`:52`) |

Validator behavior (negative control, 2026-10-10): reports errors **and** warnings, e.g. missing `img` width/height, Liquid inside `{% stylesheet %}`, `block.id` in conditions, preset `block_order` ids missing from `blocks`, unknown preset setting keys, private block not listed in parent, missing `t:` keys in either locale file, non-existent block types. It did not report `ExcessiveSettingsCount` on `sections/header.liquid` (42 settings).

## 18. Recipes: need → Horizon way

| Need | Do this (level) | Avoid |
|---|---|---|
| Brand colors, fonts, type scale, button/input/card style | Global settings (L1); palette references in role settings | CSS overrides of tokens |
| New content band (text + image + buttons, columns, FAQ, icons with text, split banner) | Add `section`, pick the closest of its 13 presets, compose `group`/`text`/`image`/`button`/`icon`/`spacer`/`_divider` (L3) | New section |
| Hero / slideshow / marquee / media-with-text / collection or product lists / blog posts | Use the dedicated stock section and its presets (L3) | Rebuilding them |
| Hand-picked products | `section` + `group` + several `product-card` blocks (each with its own `product`) (L3) | New section |
| Products of a collection | `product-list` (grid/carousel/editorial) (L3) | — |
| Metafield on product page | `text`/`image` block in `_product-details` bound via dynamic source (L2/L3) | Editing `_product-details` |
| Badge / extra line on product cards | `text` block (background pill) with metafield dynamic source in the host section's static `_product-card` (L2/L3); `cx-` block if logic is needed (L6) | Editing `snippets/card-gallery.liquid` |
| Different layout for some products/pages/collections | Alternate template `templates/<type>.<suffix>.json` (L3), assigned per resource | Liquid conditionals on handle |
| Collection banner | The `section` band in `collection.json` with `text`/`image` bound to `closest.collection` (L2/L3) | — |
| Header/footer changes | Header settings (rows, positions, sticky, transparent, colors) and footer blocks (`menu`, `text`, `email-signup`, `social-links`, `payment-icons`, `logo`, `group`) (L2/L3) | Editing `sections/header.liquid` CSS |
| Announcement bar | `header-announcements` + `_announcement` blocks (L3) | — |
| Popup / modal content | `popup-link` block containing any blocks (L3) | Custom modal JS |
| Tabs/accordion content | `accordion` + `_accordion-row` (rows accept `@theme`) (L3) | Custom JS |
| Subscriptions, reviews, wishlists, size charts from apps | `@app` blocks in sections/blocks that accept `@app` (L0/L3) | Theme code |
| Buyer input on product (engraving, notes) | `product-custom-property` block (L3) | Custom form fields |
| Cart drawer vs page | `settings.cart_type` (L1) | — |
| Filter list on collections | Search & Discovery app (L0) + `filters` block settings (L2) | Theme code |
| Small one-off Liquid (print a metafield with formatting, embed) | `custom-liquid` block (L4) | — |
| New interactive component (countdown, calculator, store locator widget) | `cx-` theme block + `assets/cx-*.js` Component (L6) — Appendix A.2 | New section |
| Page-level region with its own data loop or grid that `section` cannot express | `cx-` section reusing stock blocks/snippets (static `_product-card`, `resource-list`) (L7) — Appendix A.3 | Copying a stock section wholesale |
| Variant-reactive custom UI | `cx-` block listening to `ProductSelectEvent` (§13.1) (L6) | Editing `variant-picker.js` |
| Site-wide style rule no setting covers | `assets/cx-theme.css` linked in `snippets/stylesheets.liquid` (L5 + L8, §9.9) | Editing `base.css` |

## 19. Known stock defects and quirks (4.2.0) — do not imitate; do not fix unless asked

| Area | Item |
|---|---|
| CSS tokens | `--color-shadow-rgb` never given a value (only `inherit` in `blocks/_cart-summary.liquid:117`; read via `--backdrop-color-rgb` for the dialog backdrop); `--font-paragraph-family`/`--font-paragraph-size` (single hyphen) undefined (`assets/overflow-list.css:46-47`, `blocks/_blog-post-info-text.liquid:48`); `--style-border-radius-buttons` undefined (`snippets/quantity-selector-styles.liquid:92`, `assets/template-giftcard.css:85`); `--size-style-height-mobile` read but never emitted (`assets/base.css:1104`); `var(--section-min-height, 'auto')` quoted fallback; `--section-height-auto` undefined (hero `section_height: auto`). Also read with no fallback and defined nowhere: `--font-paragraph-medium--size`, `--font-paragraph-small--size`, `--font-weight-{semibold,medium,bold,normal}`, `--font-body--size`, `--font-primary--family`, `--font-size--body-md/-body-sm/-xlarge`, `--font-size-lg/-xs`, `--font-line-height-md`, `--line-height-tight`, `--color-foreground-secondary`, `--color-background-secondary`, `--color-primary-active`, `--cart-font-size--sm/--xl`, `--spacing-lg/-xs`, `--product-card-alignment`, `--product-card-border-width/-opacity`, `--media-border-opacity`, `--image-opacity`, `--grid-area-alignment`, `--section-padding-inline`, `--facets-panel-width`. Never copy these names; use §9.3 tokens |
| Settings | Undefined global settings read: `drawer_drop_shadow`, `drawer_shadow_color`, `social_twitter_link`, `button_text_case`; `button_text_case_*` emits `default` as a `text-transform` value; paragraph case token built from a non-existent key; `_collection-link` reads undefined `section.settings.image_height`; `_slide` tests `slide_height == 'adapt_image'` (not an option); `media-with-content`, `password`, and (via snippets) `_content`, `_content-without-appearance`, `_media-without-appearance`, `_featured-product-gallery`, `_featured-product` render `border-override` without border settings; `_content.background_color` and `page.max_width` have no effect |
| Docs vs code | `gap-style` doc says scale threshold 20, code 24; `contrast-override` doc mentions accent tokens never emitted; `typography-style` doc says preset defaults from global settings (it reads the passed settings); `anchored-popover.js` doc example uses `data-ref` (framework reads `ref`); `snippets/media.liquid` doc references a non-existent `_media` block; base.css comment says narrow width 1400px (token is 90rem = 1440px) |
| Editor attributes | Missing `block.shopify_attributes` with `tag: null`: `custom-liquid`, `_marquee`, `_collection-image`; not on top element: `menu`, `_accordion-row`, `footer-policy-list`, `_search-input`, `_carousel-content`, `_product-details`, `quantity`; output three times: `_slide` (`snippets/slideshow-slide.liquid` prints `attributes` twice, `:46,59`, plus `blocks/_slide.liquid:106`); duplicated: `icon`, `product-custom-property`, `product-recommendations`; `snippets/slideshow-slide.liquid` also emits `style` twice (`:42`, `:48`), so a passed `style` is dropped; `class` with `tag: null` has no effect (`accordion`, `_header-logo`) |
| DOM ids | Not unique per instance: `contact-form` (`ContactForm-*`), `popup-link` (`popup-dialog-heading`), `footer-policy-list` (`terms-policies-popover`), `review` (SVG ids), `localization-form` (`country-filter-input`, `no-results-message`; rendered in header and drawer); both header rows carry `ref="headerRowTop"` |
| Commerce | `variant_style == 'dropdown'` vs option value `dropdowns` (`snippets/variant-main-picker.liquid:62`); gallery does not pass `block`/`selected_product`/`first_3d_model` to `product-media`; `product-form.js` appends `sections` inside a `forEach`; `buy-buttons` form id per section (duplicates in cards); standalone `quantity` not wired; standalone `accelerated-checkout` renders nothing; `product-inventory` and sticky bar use global `product` |
| JS | 11 components create their `AbortController` once as a field and never re-arm it after reconnect: `accordion-custom`, `media-gallery`, `product-form`, `drag-zoom-wrapper`, `deferred-media` and `product-model` (`assets/media.js`), `price-per-item`, `predictive-search`, `quick-add-component`, `quick-add-dialog`, `sticky-add-to-cart` (only `disclosures-summary-fit` re-creates it, the pattern to copy); 4 unguarded `customElements.define` (`product-custom-property`, `gift-card-recipient-form`, `layered-slideshow`, `product-hotspot`); `section-renderer` ignores HTTP status |
| Platform conformance | `sections/header.liquid` has 42 settings (> 40 lint threshold); `<main>` lacks `tabindex="-1"`; JSON ids use `-`/`_` though docs say alphanumeric; `featured-product` has no `@app` |
| Orphans | `_collection-image`, `_collection-info`, `_content`, `_footer-social-icons`, `_inline-text`, `_social-link` (only parent is the orphan `_footer-social-icons`); `featured-collection` has no preset and no reference |
| Meta | `<meta name="theme-color">` empty; both layouts emit markup before `meta charset` (password: stylesheets, favicon, view-transition; theme: favicon, view-transition, `layout/theme.liquid:10-18`); `layout/password.liquid` has an unused `no-js` class |

## 20. Platform references (shopify.dev)

| Key | URL |
|---|---|
| BLK | https://shopify.dev/docs/storefronts/themes/architecture/blocks |
| TBS | https://shopify.dev/docs/storefronts/themes/architecture/blocks/theme-blocks/schema |
| TBT | https://shopify.dev/docs/storefronts/themes/architecture/blocks/theme-blocks/targeting |
| TBST | https://shopify.dev/docs/storefronts/themes/architecture/blocks/theme-blocks/static-blocks |
| TBD | https://shopify.dev/docs/storefronts/themes/architecture/blocks/theme-blocks/dynamic-sources |
| AB / AIB | https://shopify.dev/docs/storefronts/themes/architecture/blocks/app-blocks · …/blocks/ai-generated-theme-blocks |
| CF | https://shopify.dev/docs/api/liquid/tags/content_for |
| SS / SG | https://shopify.dev/docs/storefronts/themes/architecture/sections/section-schema · …/architecture/section-groups |
| JT | https://shopify.dev/docs/storefronts/themes/architecture/templates/json-templates |
| ALT | https://shopify.dev/docs/storefronts/themes/architecture/templates/alternate-templates |
| IS / SET / DS | https://shopify.dev/docs/storefronts/themes/architecture/settings/input-settings · …/architecture/settings · …/settings/dynamic-sources |
| SLF / STLF | https://shopify.dev/docs/storefronts/themes/architecture/locales/schema-locale-files · …/storefront-locale-files |
| JSCSS / SUB | https://shopify.dev/docs/storefronts/themes/best-practices/javascript-and-stylesheet-tags · …/performance/stylesheet-subsetting |
| PERF / RIMG / LCP | https://shopify.dev/docs/storefronts/themes/best-practices/performance · …/performance/use-responsive-images · …/performance/never-lazy-load-lcp-image |
| A11Y | https://shopify.dev/docs/storefronts/themes/best-practices/accessibility |
| EDI / PINS | https://shopify.dev/docs/storefronts/themes/best-practices/editor/integrate-sections-and-blocks · …/editor/preview-inspector |
| TC | https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks |
| SRA | https://shopify.dev/docs/api/ajax/section-rendering |
| UPD | https://shopify.dev/docs/storefronts/themes/store/success/updates |
| LDOC | https://shopify.dev/docs/storefronts/themes/tools/liquid-doc |
| LIM | https://shopify.dev/docs/storefronts/themes/architecture/limits |
| LOC | https://shopify.dev/docs/storefronts/themes/architecture/locales |
| LCR | https://shopify.dev/docs/storefronts/themes/best-practices/performance/load-critical-resources-before-content-for-header |
| TAP | https://shopify.dev/docs/storefronts/themes/best-practices/performance/increase-tap-target-size |
| CMT | https://shopify.dev/docs/api/liquid/tags/comment (inline comments are `{% # … %}`, not `{# … #}`) |

Docs inconsistencies to keep in mind (confirmed present in the docs, 2026-10-10): `closest.*` docs say it is "accessible exclusively via theme settings", yet Horizon uses it in plain Liquid; Theme Check examples use `context.product` while Horizon uses `closest.*` (use `closest.*`); `UnsupportedDocTag` says `{% doc %}` is for snippets while the LiquidDoc page and `ReservedDocParamNames` cover blocks (use it in snippets and blocks, not sections). Undocumented (UNVERIFIED): cross-scope `visible_if` (works in Horizon); whether nested blocks count toward the 50-per-section cap (LIM: caps cover merchant-addable blocks only; static blocks do not count); block-level `"disabled": true`; `"alpha": true` on color settings (Horizon uses it).


## Appendix A. Code templates (validated with `validate_theme`, 2026-10-10)

All four templates passed `validate_theme` in a scratch copy of stock 4.2.0 together with the locale additions in A.4. They are skeletons: the functional content is illustrative. Rename `cx-example*` to the real component name, delete what the component does not need, and keep the conventions.

### A.1 Leaf theme block — `blocks/cx-example.liquid`

Conventions shown: `{% doc %}`, `block_settings`, `contrast-override` + conditional `color-custom-{{ block.id }}`, `"tag": null` + single root with `block.shopify_attributes`, `spacing-style` helper, namespaced instance variable, type preset as class, logical alignment, one static `{% stylesheet %}`, stock `t:` keys reused, custom keys under `t:cx.*`, preset with category.

```liquid
{%- doc -%}
  Renders a heading and rich text with Horizon color, spacing and typography settings.
  Skeleton for a custom leaf theme block.

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

{% render 'contrast-override',
  background_color: block_settings.background_color,
  text_color: block_settings.text_color,
  section_id: block.id
%}

<div
  class="
    cx-example
    cx-example--align-{{ block_settings.alignment }}
    spacing-style
    {% if has_custom_color %}color-custom-{{ block.id }}{% endif %}
  "
  style="
    {% render 'spacing-style', settings: block_settings %}
    --cx-example-radius: {{ block_settings.corner_radius }}px;
  "
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
  .cx-example {
    display: flex;
    flex-direction: column;
    gap: var(--gap-xs);
    min-width: 0;
    border-radius: var(--cx-example-radius, 0);
  }

  .cx-example--align-left {
    text-align: start;
  }

  .cx-example--align-center {
    text-align: center;
  }

  .cx-example--align-right {
    text-align: end;
  }

  .cx-example__heading {
    margin: 0;
  }
{% endstylesheet %}

{% schema %}
{
  "name": "t:cx.example.name",
  "tag": null,
  "settings": [
    {
      "type": "inline_richtext",
      "id": "heading",
      "label": "t:cx.example.heading",
      "default": "t:cx.example.heading_default"
    },
    {
      "type": "select",
      "id": "heading_preset",
      "label": "t:settings.preset",
      "options": [
        { "value": "paragraph", "label": "t:options.paragraph" },
        { "value": "h2", "label": "t:options.h2" },
        { "value": "h3", "label": "t:options.h3" },
        { "value": "h4", "label": "t:options.h4" },
        { "value": "h5", "label": "t:options.h5" },
        { "value": "h6", "label": "t:options.h6" }
      ],
      "default": "h4",
      "info": "t:info.edit_presets_in_theme_settings"
    },
    {
      "type": "richtext",
      "id": "text",
      "label": "t:settings.text"
    },
    {
      "type": "text_alignment",
      "id": "alignment",
      "label": "t:settings.alignment",
      "default": "left"
    },
    {
      "type": "header",
      "content": "t:content.appearance"
    },
    {
      "type": "color",
      "id": "background_color",
      "label": "t:settings.background_color",
      "placeholder": "t:settings.default"
    },
    {
      "type": "color",
      "id": "text_color",
      "label": "t:settings.text_color",
      "placeholder": "t:settings.default"
    },
    {
      "type": "range",
      "id": "corner_radius",
      "label": "t:settings.corner_radius",
      "min": 0,
      "max": 50,
      "step": 1,
      "unit": "px",
      "default": 0
    },
    {
      "type": "header",
      "content": "t:content.padding"
    },
    {
      "type": "range",
      "id": "padding-block-start",
      "label": "t:settings.top",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 0
    },
    {
      "type": "range",
      "id": "padding-block-end",
      "label": "t:settings.bottom",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 0
    },
    {
      "type": "range",
      "id": "padding-inline-start",
      "label": "t:settings.left",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 0
    },
    {
      "type": "range",
      "id": "padding-inline-end",
      "label": "t:settings.right",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 0
    }
  ],
  "presets": [
    {
      "name": "t:cx.example.name",
      "category": "t:categories.basic"
    }
  ]
}
{% endschema %}
```

### A.2 Interactive theme block + module — `blocks/cx-countdown.liquid`, `assets/cx-countdown.js`

Conventions shown: server-rendered fallback, gated module `<script>` in the block, `*-component` custom element as the root, optional refs, storefront strings via `| t`, `role="timer"`, cleanup in `disconnectedCallback`, idempotent restart in `updatedCallback`, guarded `customElements.define`, no import-map entry (nothing imports it).

```liquid
{%- doc -%}
  Renders a countdown to a merchant-set end time. The server renders the end time as text;
  assets/cx-countdown.js fills in the remaining time.
  Skeleton for a custom interactive theme block.

  @example
  {% content_for 'block', type: 'cx-countdown', id: 'countdown' %}
{%- enddoc -%}

{%- assign block_settings = block.settings -%}

{%- if block_settings.end_time != blank -%}
  <script
    src="{{ 'cx-countdown.js' | asset_url }}"
    type="module"
    fetchpriority="low"
  ></script>
{%- endif -%}

<cx-countdown-component
  class="cx-countdown spacing-style {{ block_settings.type_preset }}"
  style="{% render 'spacing-style', settings: block_settings %}"
  data-end="{{ block_settings.end_time | escape }}"
  {{ block.shopify_attributes }}
>
  {%- if block_settings.end_time != blank -%}
    <div
      class="cx-countdown__units"
      role="timer"
      aria-label="{{ 'cx.countdown.label' | t }}"
    >
      <span class="cx-countdown__unit">
        <span class="cx-countdown__value" ref="days">--</span>
        <span class="cx-countdown__label">{{ 'cx.countdown.days' | t }}</span>
      </span>
      <span class="cx-countdown__unit">
        <span class="cx-countdown__value" ref="hours">--</span>
        <span class="cx-countdown__label">{{ 'cx.countdown.hours' | t }}</span>
      </span>
      <span class="cx-countdown__unit">
        <span class="cx-countdown__value" ref="minutes">--</span>
        <span class="cx-countdown__label">{{ 'cx.countdown.minutes' | t }}</span>
      </span>
      <span class="cx-countdown__unit">
        <span class="cx-countdown__value" ref="seconds">--</span>
        <span class="cx-countdown__label">{{ 'cx.countdown.seconds' | t }}</span>
      </span>
    </div>
    {%- if block_settings.expired_text != blank -%}
      <p class="cx-countdown__expired">{{ block_settings.expired_text | escape }}</p>
    {%- endif -%}
  {%- endif -%}
</cx-countdown-component>

{% stylesheet %}
  .cx-countdown {
    display: block;
    min-width: 0;
  }

  .cx-countdown__units {
    display: flex;
    flex-wrap: wrap;
    gap: var(--gap-md);
  }

  .cx-countdown__unit {
    display: flex;
    flex-direction: column;
    align-items: center;
    min-width: 0;
  }

  .cx-countdown__value {
    font-variant-numeric: tabular-nums;
  }

  .cx-countdown__label {
    font-size: var(--font-size--xs);
    color: var(--color-foreground-muted);
  }

  .cx-countdown__expired {
    display: none;
    margin: 0;
  }

  .cx-countdown[data-expired] .cx-countdown__units {
    display: none;
  }

  .cx-countdown[data-expired] .cx-countdown__expired {
    display: block;
  }
{% endstylesheet %}

{% schema %}
{
  "name": "t:cx.countdown.name",
  "tag": null,
  "settings": [
    {
      "type": "text",
      "id": "end_time",
      "label": "t:cx.countdown.end_time",
      "info": "t:cx.countdown.end_time_info"
    },
    {
      "type": "text",
      "id": "expired_text",
      "label": "t:cx.countdown.expired_text"
    },
    {
      "type": "select",
      "id": "type_preset",
      "label": "t:settings.preset",
      "options": [
        { "value": "paragraph", "label": "t:options.paragraph" },
        { "value": "h3", "label": "t:options.h3" },
        { "value": "h4", "label": "t:options.h4" },
        { "value": "h5", "label": "t:options.h5" }
      ],
      "default": "h4"
    },
    {
      "type": "header",
      "content": "t:content.padding"
    },
    {
      "type": "range",
      "id": "padding-block-start",
      "label": "t:settings.top",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 0
    },
    {
      "type": "range",
      "id": "padding-block-end",
      "label": "t:settings.bottom",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 0
    }
  ],
  "presets": [
    {
      "name": "t:cx.countdown.name",
      "category": "t:categories.basic"
    }
  ]
}
{% endschema %}
```

```js
import { Component } from '@theme/component';

/**
 * @typedef {object} CountdownRefs
 * @property {HTMLElement} [days]
 * @property {HTMLElement} [hours]
 * @property {HTMLElement} [minutes]
 * @property {HTMLElement} [seconds]
 */

/**
 * Countdown to the ISO date-time in `data-end`. Values are written into optional refs.
 * @extends {Component<CountdownRefs>}
 */
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
    const totalSeconds = Math.floor(remaining / 1000);
    const { days, hours, minutes, seconds } = this.refs;

    this.#write(days, Math.floor(totalSeconds / 86400));
    this.#write(hours, Math.floor((totalSeconds % 86400) / 3600));
    this.#write(minutes, Math.floor((totalSeconds % 3600) / 60));
    this.#write(seconds, totalSeconds % 60);

    if (remaining === 0) {
      this.#stop();
      this.toggleAttribute('data-expired', true);
    }
  }

  /**
   * @param {HTMLElement | undefined} element
   * @param {number} value
   */
  #write(element, value) {
    if (!element) return;
    element.textContent = String(value).padStart(2, '0');
  }
}

if (!customElements.get('cx-countdown-component')) {
  customElements.define('cx-countdown-component', CxCountdownComponent);
}
```

Note: if this block can first appear through a Section Rendering API morph, the module script inside it will not execute (§12.1 step 4); load the module from a file that is present on full page load instead.

### A.3 Section — `sections/cx-example-section.liquid`

Use only when `section` + blocks cannot express the need (§2.1 L7). Conventions shown: section contract comment, hand-rolled skeleton (`.section-background` + `.section`, `color-custom-{{ section.id }}` on both), `spacing-style`, `layout-panel-style` area, static `group` header with matching preset entry (`"static": true`, key = id, not in `block_order`), dynamic area accepting `@theme`, `@app`, `_divider`, `disabled_on` header, stock setting IDs reused.

```liquid
{% comment %}
  Section contract.
  Renders a page band with a fixed header group and a merchant-managed area of theme blocks.
  Skeleton for a custom section. Use only when sections/section.liquid plus theme blocks cannot express the need.
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
  class="
    section
    section--{{ section_settings.section_width }}
    cx-example-section
    spacing-style
    {% if has_custom_color %}color-custom-{{ section.id }}{% endif %}
  "
  style="{% render 'spacing-style', settings: section_settings %}"
>
  <div class="cx-example-section__header">
    {% content_for 'block', type: 'group', id: 'static-header' %}
  </div>

  <div
    class="
      cx-example-section__content
      layout-panel-flex
      layout-panel-flex--{{ section_settings.content_direction }}
      {% if section_settings.vertical_on_mobile %}mobile-column{% endif %}
    "
    style="{% render 'layout-panel-style', settings: section_settings %}"
  >
    {% content_for 'blocks' %}
  </div>
</div>

{% stylesheet %}
  .cx-example-section {
    row-gap: var(--gap-lg);
  }

  .cx-example-section__header,
  .cx-example-section__content {
    min-width: 0;
  }
{% endstylesheet %}

{% schema %}
{
  "name": "t:cx.example_section.name",
  "disabled_on": {
    "groups": ["header"]
  },
  "blocks": [
    { "type": "@theme" },
    { "type": "@app" },
    { "type": "_divider" }
  ],
  "settings": [
    {
      "type": "select",
      "id": "content_direction",
      "label": "t:settings.direction",
      "options": [
        { "value": "column", "label": "t:options.vertical" },
        { "value": "row", "label": "t:options.horizontal" }
      ],
      "default": "row"
    },
    {
      "type": "checkbox",
      "id": "vertical_on_mobile",
      "label": "t:settings.vertical_on_mobile",
      "default": true,
      "visible_if": "{{ section.settings.content_direction == 'row' }}"
    },
    {
      "type": "select",
      "id": "horizontal_alignment",
      "label": "t:settings.alignment",
      "options": [
        { "value": "flex-start", "label": "t:options.direction_start" },
        { "value": "center", "label": "t:options.direction_center" },
        { "value": "flex-end", "label": "t:options.direction_end" },
        { "value": "space-between", "label": "t:options.space_between" }
      ],
      "default": "flex-start",
      "visible_if": "{{ section.settings.content_direction == 'row' }}"
    },
    {
      "type": "select",
      "id": "vertical_alignment",
      "label": "t:settings.position",
      "options": [
        { "value": "flex-start", "label": "t:options.top" },
        { "value": "center", "label": "t:options.center" },
        { "value": "flex-end", "label": "t:options.bottom" }
      ],
      "default": "flex-start",
      "visible_if": "{{ section.settings.content_direction == 'row' }}"
    },
    {
      "type": "select",
      "id": "horizontal_alignment_flex_direction_column",
      "label": "t:settings.alignment",
      "options": [
        { "value": "flex-start", "label": "t:options.direction_start" },
        { "value": "center", "label": "t:options.direction_center" },
        { "value": "flex-end", "label": "t:options.direction_end" }
      ],
      "default": "flex-start",
      "visible_if": "{{ section.settings.content_direction != 'row' }}"
    },
    {
      "type": "select",
      "id": "vertical_alignment_flex_direction_column",
      "label": "t:settings.position",
      "options": [
        { "value": "flex-start", "label": "t:options.top" },
        { "value": "center", "label": "t:options.center" },
        { "value": "flex-end", "label": "t:options.bottom" },
        { "value": "space-between", "label": "t:options.space_between" }
      ],
      "default": "flex-start",
      "visible_if": "{{ section.settings.content_direction == 'column' }}"
    },
    {
      "type": "range",
      "id": "gap",
      "label": "t:settings.gap",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 24
    },
    {
      "type": "select",
      "id": "section_width",
      "label": "t:settings.width",
      "options": [
        { "value": "page-width", "label": "t:options.page" },
        { "value": "full-width", "label": "t:options.full" }
      ],
      "default": "page-width"
    },
    {
      "type": "header",
      "content": "t:content.appearance"
    },
    {
      "type": "color",
      "id": "background_color",
      "label": "t:settings.background_color",
      "placeholder": "t:settings.default"
    },
    {
      "type": "header",
      "content": "t:content.padding"
    },
    {
      "type": "range",
      "id": "padding-block-start",
      "label": "t:settings.top",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 40
    },
    {
      "type": "range",
      "id": "padding-block-end",
      "label": "t:settings.bottom",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 40
    }
  ],
  "presets": [
    {
      "name": "t:cx.example_section.name",
      "category": "t:categories.layout",
      "blocks": {
        "static-header": {
          "type": "group",
          "static": true,
          "blocks": {
            "heading": {
              "type": "text",
              "settings": {
                "text": "t:html_defaults.new_arrivals_h2"
              }
            }
          },
          "block_order": ["heading"]
        },
        "example_1": {
          "type": "cx-example"
        },
        "example_2": {
          "type": "cx-example"
        }
      },
      "block_order": ["example_1", "example_2"]
    }
  ]
}
{% endschema %}
```

### A.4 Locale additions

`locales/en.default.schema.json` (add as a top-level `cx` object; keep the file's comments):

```json
"cx": {
  "example": { "name": "Example", "heading": "Heading", "heading_default": "Example heading" },
  "countdown": {
    "name": "Countdown",
    "end_time": "End time",
    "end_time_info": "ISO 8601 date and time, for example 2026-12-31T23:59:00-05:00",
    "expired_text": "Text after the countdown ends"
  },
  "example_section": { "name": "Example section" }
}
```

`locales/en.default.json`:

```json
"cx": {
  "countdown": { "label": "Time remaining", "days": "Days", "hours": "Hours", "minutes": "Minutes", "seconds": "Seconds" }
}
```

### A.5 Template JSON — `templates/page.example.json`

Shows a stock section with dynamic blocks bound to `closest.page`, a custom section with a static block (`"static": true`, absent from `block_order`) and dynamic custom blocks.

```json
{
  "sections": {
    "main": {
      "type": "main-page",
      "blocks": {
        "heading": {
          "type": "text",
          "settings": {
            "text": "<h1>{{ closest.page.title }}</h1>",
            "type_preset": "rte"
          }
        },
        "page-content": {
          "type": "page-content",
          "settings": {}
        }
      },
      "block_order": ["heading", "page-content"],
      "settings": {}
    },
    "example": {
      "type": "cx-example-section",
      "blocks": {
        "static-header": {
          "type": "group",
          "static": true,
          "settings": {},
          "blocks": {
            "heading": {
              "type": "text",
              "settings": {
                "text": "<h2>Why customers choose us</h2>"
              }
            }
          },
          "block_order": ["heading"]
        },
        "example_a1b2c3": {
          "type": "cx-example",
          "settings": {
            "heading": "Fast shipping",
            "heading_preset": "h4"
          }
        },
        "countdown_d4e5f6": {
          "type": "cx-countdown",
          "settings": {
            "end_time": "2026-12-31T23:59:00-05:00",
            "expired_text": "The sale has ended."
          }
        }
      },
      "block_order": ["example_a1b2c3", "countdown_d4e5f6"],
      "settings": {
        "content_direction": "row",
        "gap": 24
      }
    }
  },
  "order": ["main", "example"]
}
```
