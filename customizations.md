# Theme changes (how this theme differs from stock Horizon [agent: insert the Horizon version THEME.md covers])

The only record of how this theme differs from stock Horizon, apart from merchant edits made in the theme editor (`THEME.md` §2.4). Every change to a theme file is logged here in the same task it is made. How to choose and build a change (the L0–L8 ladder, `cx-` naming, validation) lives in `THEME.md`; working rules live in `CLAUDE.md`.

The order to work in (`CLAUDE.md` design rules, `THEME.md` §2.1–§2.2), stopping at the first step that matches the design:
1. store data and global theme settings (L0–L1);
2. existing sections and theme blocks, with their presets and settings (L2–L3);
3. a `custom-liquid` block, only for a few lines (L4);
4. a new `cx-` theme block (L6), unless the change is a small addition to the stock file that already does the job: then edit that file (L5);
5. a new `cx-` section only for a page-level region that sections plus blocks cannot express, or to replace a stock section when the user picks that (L7).

Horizon version of this theme: **[agent: insert the Horizon version THEME.md covers]** (`config/settings_schema.json` → `theme_version`).

Two kinds of file are tracked:

1. **Modified stock files**: any edit to a file that shipped with Horizon: `config/settings_data.json` (L1); `templates/*.json` and `sections/*-group.json` (L2–L4); `locales/*.json` (added `cx.*` keys or reworded stock strings, L8); any stock `.liquid`, `.css` or `.js` file or `config/settings_schema.json` (L5, small additions only). Keep its L5 code edits few: each one has to be merged again by hand after a Horizon update (see Tips).
2. **New custom code**: any file that does not exist in stock Horizon: `cx-` blocks (L6), `cx-` sections (L7), `cx-` snippets and assets (L8), new templates such as `templates/page.faq.json` (L3; plain name, no `cx-`). On a Horizon update, they are copied by hand into the new version as they are.

Not tracked: store data (products, collections, metafields, metaobjects, menus, pages, Search & Discovery filters, apps: L0) and changes the merchant makes in the theme editor. When a change depends on store data, name it in the entry.

---

## How to write an entry

One entry per file, in the section for its kind, newest at the bottom. A later change to the same file adds a dated line to its entry; a change that replaces an earlier one rewrites the entry.

```markdown
### `path/to/file` — short title (YYYY-MM-DD)
Level: L1–L8 (`THEME.md` §2.1)
What: what the file does, or what changed and where. Quote one-line stock edits verbatim.
Settings: settings touched, if any (table below for `config/settings_data.json`); store data it reads (metafield `namespace.key`, metaobject type, menu handle).
Why: the reason. For L4–L8, the justification block below.
Checks: `validate_theme` result, what was previewed and compared with the design, and any difference left (with the user's answer), or what could not be checked.
Undo: how to put it back.
```

For `config/settings_data.json`, list every setting changed. If the live value before the change was not the stock value (the merchant had changed it), note that value too; Undo restores what the merchant had.

| Setting ID | Stock value | New value | Why |
|---|---|---|---|
| `page_width` | `narrow` | `normal` | example row |

### Justification (L4–L8)

Write the `THEME.md` §2.3 block **before** writing code. When one change spans several files (a `cx-` block, its script, its locale keys), write it once in the main file's entry and put `Why: see <main file>` in the others.

```
Need:            <observable behavior the merchant/user needs>
Stock checked:   <sections/blocks/settings/presets evaluated, with file names>
Why not stock:   <the specific missing capability, per candidate>
Level chosen:    L5 | L6 | L7 | L8  (+ L4 if custom-liquid)
Files:           <new or edited files>
Settings/IDs:    <new setting IDs, block types, static ids, locale keys>
Upgrade impact:  <stock files edited (merge again by hand on update) / new files (copy as they are on update)>
```

"Why not stock" must answer, by level: L4, why no stock setting or dynamic source can show it and why it stays a few lines with no CSS, settings or JS; L5, which stock file already does the job and why the edit is a small addition to it (otherwise it is an L6 block, or, if it would rebuild the file, the user's choice under `THEME.md` §2.1); L6, which stock blocks, presets and compositions (L1–L3) were tried and what is missing or does not match the design; L7, why `section` (13 presets) filled with theme blocks, including a `cx-` block, cannot express the region, or that the user picked it to replace a stock section; L8, which L5–L7 change it serves.

### Tips

- **Automatic updates** (`THEME.md` §2.1): Shopify updates the theme automatically only while every file other than `config/settings_data.json` and the JSON files in `templates/` is unchanged. Any other stock edit stops them, including `sections/*-group.json` and `locales/*.json`; assume any new file outside `templates/` does too.
- **After a manual Horizon update**: the update installs the new version as an unpublished copy, which the `CLAUDE.md` workflow rules out, so it happens outside that workflow, on the copy and before it is published: re-apply every entry under "Modified stock files", copy every file under "New custom code" as it is, check that the stock classes and attributes listed in the entries for `assets/cx-theme.css` and the `cx-` style snippets still exist, run `validate_theme` on all of them, then publish. Once the new version is live, the version check stops theme work (`THEME.md` §0.1): run setup again first (**Full theme design**), then update the Horizon version in this file's title and in the version line at the top. Setup keeps this file as it is.
- **`assets/cx-theme.css` and the `cx-` style snippets** (`snippets/cx-<file>-styles.liquid`): each entry lists every stock class, attribute and custom property its rules rely on (for example `.group-block`, `[data-menu-style]`); a style snippet's entry also lists its `cx_style` values (`THEME.md` §9.10). Update the list whenever a rule changes.
- **A typical theme-block change** is several entries: `blocks/cx-<name>.liquid` (L6, holds the justification) and `assets/cx-<name>.js` (L8) under "New custom code"; the `cx.*` keys added to `locales/*.json` (L8) under "Modified stock files"; the `templates/*.json` that places it (L3) under "Modified stock files" only if you placed it, not if the merchant did.
- **Locale keys:** list every `cx.*` key added to `locales/en.default.json`, `locales/en.default.schema.json` and each other storefront locale the shop publishes, and every stock string reworded for the brand, with its old and new text (`THEME.md` §14).
- **Removing custom code:** ask the user first (`CLAUDE.md`). Check that no template or section group still uses the `cx-` file (saved data breaks, `THEME.md` §3.8), then undo its stock edits and delete its entries.

---

## Modified stock files

None yet.

## New custom code

None yet.
