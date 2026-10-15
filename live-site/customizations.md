# Theme changes (how this copy of [agent: insert the theme's name and version] differs from its baseline)

The only record of what this project changed in the theme, apart from merchant edits made in the theme editor. The baseline is the copy as the user downloaded it into `theme/` when the project started, not the theme as its author shipped it: changes made before then, by the merchant, an agency or an app, are part of the baseline and aren't logged here. Every change to a theme file is logged here in the same task it is made. How the theme is built lives in `THEME.md`; working rules, the L0–L8 levels and `cx-` naming live in `CLAUDE.md`.

The order to work in (`CLAUDE.md` design rules), stopping at the first step that matches the design:
1. store data and global theme settings (L0–L1);
2. existing sections and blocks, with their presets and settings, and new JSON templates (L2–L3);
3. a `custom-liquid` section or block, only for a few lines (L4);
4. new code that leaves stock files alone: a `cx-` theme block (L6) where the theme's sections accept theme blocks, unless the change is a small addition to the stock file that already does the job: then edit that file (L5);
5. a new `cx-` section (L7);
6. snippets, assets and locale keys that serve those changes (L8).

Theme: **[agent: insert the theme's name and version]** (`config/settings_schema.json` → `theme_info`).

Two kinds of file are tracked:

1. **Modified stock files**: any edit to a file that was in the baseline: `config/settings_data.json` (L1); `templates/*.json` and `sections/*-group.json` (L2–L4); `locales/*.json` (added `cx.*` keys or reworded stock strings, L8); any stock `.liquid`, `.css` or `.js` file or `config/settings_schema.json` (L5, small additions only). Keep its L5 code edits few: each one has to be merged again by hand after a theme update (see Tips).
2. **New custom code**: any file that wasn't in the baseline: `cx-` blocks (L6), `cx-` sections (L7), `cx-` snippets and assets (L8), new templates such as `templates/page.faq.json` (L3; plain name, no `cx-`). On a theme update, they are copied by hand into the new version as they are.

Not tracked: store data (products, collections, metafields, metaobjects, menus, pages, Search & Discovery filters, apps: L0) and changes the merchant makes in the theme editor. When a change depends on store data, name it in the entry.

---

## How to write an entry

One entry per file, in the section for its kind, newest at the bottom. A later change to the same file adds a dated line to its entry; a change that replaces an earlier one rewrites the entry.

```markdown
### `path/to/file` — short title (YYYY-MM-DD)
Level: L1–L8 (`CLAUDE.md` design rules)
What: what the file does, or what changed and where. Quote one-line stock edits verbatim.
Settings: settings touched, if any (table below for `config/settings_data.json`); store data it reads (metafield `namespace.key`, metaobject type, menu handle).
Why: the reason. For L4–L8, the justification block below.
Checks: `validate_theme` result, what was previewed and compared with the design, and any difference left (with the user's answer), or what could not be checked.
Undo: how to put it back.
```

For `config/settings_data.json`, list every setting changed. If the value before the change was not the theme's default (the merchant had changed it), note that value too; Undo restores what the merchant had.

| Setting ID | Stock value | New value | Why |
|---|---|---|---|
| `example_setting` | `old` | `new` | example row |

### Justification (L4–L8)

Write this block **before** writing code. When one change spans several files (a `cx-` block, its script, its locale keys), write it once in the main file's entry and put `Why: see <main file>` in the others.

```
Need:            <observable behavior the merchant/user needs>
Stock checked:   <sections/blocks/settings/presets evaluated, with file names>
Why not stock:   <the specific missing capability, per candidate>
Level chosen:    L4 | L5 | L6 | L7 | L8
Files:           <new or edited files>
Settings/IDs:    <new setting IDs, block types, static ids, locale keys>
Upgrade impact:  <stock files edited (merge again by hand on update) / new files (copy as they are on update)>
```

"Why not stock" must answer, by level: L4, why no stock setting or dynamic source can show it and why it stays a few lines with no CSS, settings or JS; L5, which stock file already does the job and why the edit is a small addition to it (otherwise it is new code, or, if it would rebuild the file, the user's choice under the `CLAUDE.md` design rules); L6, which stock blocks, presets and settings (L1–L3) were tried and what is missing or does not match the design; L7, why the stock sections and blocks, including a `cx-` block where the theme takes them, cannot express it, or that the user picked it to replace a stock section; L8, which L5–L7 change it serves.

### Tips

- **Automatic updates**: Shopify updates a theme-store theme automatically only while every file other than `config/settings_data.json` and the JSON files in `templates/` is unchanged. Any other stock edit stops them, including `sections/*-group.json` and `locales/*.json`; assume any new file outside `templates/` does too.
- **Moving to a new theme version**: the update installs in the Shopify admin as another unpublished theme. The user runs setup again and picks it, and downloads it into `theme/` (emptied first). Then, on it: re-apply every entry under "Modified stock files", copy every file under "New custom code" as it is, check that the stock classes and attributes listed in the entries for `assets/cx-theme.css` still exist, and run `validate_theme` on all of them. The user asks Claude to learn the theme again, which rewrites `THEME.md` for the new version; then update the version in this file's title and in the theme line at the top, and add a dated line to each entry saying it was carried over. Before the user publishes, check the live theme for changes made since (`CLAUDE.md`, "Before the user publishes").
- **`assets/cx-theme.css`**: its entry lists every stock class, attribute and custom property its rules rely on. Update the list whenever a rule changes.
- **Locale keys:** list every `cx.*` key added to the default storefront and schema locale files and each other storefront locale the shop publishes, and every stock string reworded for the brand, with its old and new text.
- **Removing custom code:** ask the user first (`CLAUDE.md`). Check that no template or section group still uses the `cx-` file (saved data that names a missing section or block breaks), then undo its stock edits and delete its entries.

---

## Modified stock files

None yet.

## New custom code

None yet.
