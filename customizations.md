# Theme changes (how this theme differs from stock Horizon [agent: insert the Horizon version THEME.md covers])

This is the only record of how this theme differs from stock Horizon, apart from merchant edits made in the theme editor (`THEME.md` §2.4).
Every change made to theme files is logged here in the same task it is made.

How to choose and build a change (the L0–L8 ladder, `cx-` naming, validation) lives in `THEME.md`. Working rules live in `CLAUDE.md`.

The order to work in (`CLAUDE.md` design rules, `THEME.md` §2.1–§2.2):
1. existing settings, sections and theme blocks;
2. then a new `cx-` theme block;
3. a stock file edit only for a small addition to the file that already does the job;
4. a new `cx-` section only for a page-level region.

Keep "Modified stock files" short: every entry there must be redone by hand on each Horizon update, while "New custom code" is copied over as is.

Horizon version of this theme: **[agent: insert the Horizon version THEME.md covers]** (`config/settings_schema.json` → `theme_version`).

Two kinds of file are tracked:

1. **Modified stock files**: any edit to a file that shipped with Horizon. This includes:
   - `config/settings_data.json` (theme settings, L1);
   - `templates/*.json` and `sections/*-group.json` (block settings and composition, L2–L3, and `custom-liquid` content, L4);
   - `locales/*.json` (added `cx.*` keys, L8);
   - any stock `.liquid`, `.css` or `.js` file (L5: only a small addition to the file that already does the job).
2. **New custom code**: any file that does not exist in stock Horizon:
   - `cx-` theme blocks (L6, the default for new code) and `cx-` sections (L7, page-level regions only);
   - `cx-` snippets and assets (L8);
   - new templates such as `templates/page.faq.json` (L3; plain, readable name, no `cx-`: `THEME.md` §2.5).

Not tracked here:
- store data (products, collections, metafields, metaobjects, menus, apps: L0);
- changes the merchant makes in the theme editor.

When a change depends on store data, name that data in its entry (see Tips).

---

## How to write an entry

- One entry per file, placed in the section for its kind. Newest entries go at the bottom of their section.
- If the same file is changed again later, add a dated line to its existing entry.
- If a later change replaces an earlier one, rewrite the entry instead of adding a second one.

```markdown
### `path/to/file` — short title (YYYY-MM-DD)
Level: L1–L8 (`THEME.md` §2.1)
What: what the file does, or what changed and where in the file. Quote one-line stock edits verbatim.
Settings: settings touched, if any (table below for `config/settings_data.json`).
Why: the reason. For L4–L8, the justification block below.
Checks: `validate_theme` result and what was previewed, or what could not be checked.
Undo: how to put it back.
```

For `config/settings_data.json`, list every setting you changed, so the theme can be put back:

| Setting ID | Stock value | New value | Why |
|---|---|---|---|
| `page_width` | `narrow` | `normal` | example row: wider pages for the catalog |

If the live value just before your change was not the stock value (the merchant had changed it in the editor), note that value too. Undo should restore what the merchant had.

### Justification (L4–L8)

L4–L8 means: `custom-liquid` content, stock code edits, and new `cx-` code.

- Write this block from `THEME.md` §2.3 **before** writing code.
- When one change spans several files (for example a `cx-` block, its script and its locale keys), write the block once, in the entry of the main file. In the other entries, put `Why: see <main file>`.

```
Need:            <what the merchant/user needs, observable behavior>
Stock checked:   <sections/blocks/settings/presets evaluated, with file names>
Why not stock:   <the specific missing capability, per candidate>
Level chosen:    L5 | L6 | L7 | L8  (+ L4 if custom-liquid)
Files:           <new or edited files>
Settings/IDs:    <new setting IDs, block types, static ids, locale keys>
Upgrade impact:  <stock files edited (need re-merge on Horizon update) / new files (carry over)>
```

What "Why not stock" must answer, by level:

| Level | Answer |
|---|---|
| L4 `custom-liquid` | Why no stock block setting or dynamic source can show it, and why it stays a few lines with no CSS, settings or JS (otherwise it is an L6 block). |
| L5 stock code edit | Why a new `cx-` theme block cannot do it, which stock file already does the job, and why the edit is a small addition to it. |
| L6 `cx-` theme block | Which stock blocks, presets and compositions were tried (L1–L3) and what is missing. |
| L7 `cx-` section | Why the stock `section` (13 presets) filled with theme blocks, including a `cx-` block, cannot express this page-level region. |
| L8 supporting code | Which L5–L7 change it serves. |

### Tips

- **Automatic updates** (`THEME.md` §2.1):
  - Shopify updates the theme automatically only while every file other than `config/settings_data.json` and the JSON files in `templates/` is unchanged.
  - Any other stock edit stops automatic updates, including `sections/*-group.json` and `locales/*.json`. Assume any new file outside `templates/` stops them too.
  - After a manual Horizon update:
    1. re-apply every entry under "Modified stock files";
    2. copy every file under "New custom code";
    3. re-run `validate_theme` on all of them;
    4. update the version line at the top (and follow `THEME.md` §0.1).
- **A typical theme-block change** is logged as several entries:
  - `blocks/cx-<name>.liquid` under "New custom code" (L6). This entry holds the justification.
  - `assets/cx-<name>.js`, if any, under "New custom code" (L8), with `Why: see blocks/cx-<name>.liquid`.
  - The `cx.*` keys added to `locales/*.json`, under "Modified stock files" (L8), with the same `Why`.
  - Placing the block on a page edits a `templates/*.json` file. Log it under "Modified stock files" (L3) only if you placed it; a merchant placing it in the editor is not logged.
- **One-line stock edits**: quote the line. Example: `{{ 'cx-theme.css' | asset_url | stylesheet_tag }}` added to `snippets/stylesheets.liquid` after the `base.css` line (`THEME.md` §9.9).
- **Locale keys**: list every `cx.*` key added to `locales/en.default.json` and `locales/en.default.schema.json`, and to each other storefront locale the shop publishes (`THEME.md` §14).
- **Store-data dependencies**: if a change reads a metafield (`namespace.key`), a metaobject type or a menu handle, name it under Settings, so the dependency is not lost.
- **Removing custom code**: ask the user first (`CLAUDE.md`: ask before deleting anything). Before deleting a `cx-` file, check that no template or section group still uses it (saved data breaks, `THEME.md` §3.8). Then undo its stock edits and delete its entries.

---

## Modified stock files

None yet.

## New custom code

None yet.
