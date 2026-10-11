# claude-horizon

Project docs for designing Shopify's [Horizon](https://github.com/Shopify/horizon) theme with [Claude Code](https://claude.com/claude-code) and [`@acodera/shopify-admin-mcp`](https://github.com/acoderacom/shopify-admin-mcp): working rules, a technical spec for each Horizon version, and a change log, so Claude customizes the theme the Horizon way and records every change it makes.

## Features

- **Working rules** (`CLAUDE.md`): store access, editing and pushing theme files, design rules, mockups with a Horizon map, and logging
- **A spec for each Horizon version** (`versions/<version>/THEME.md`): how the theme is built and how to customize it, with a section map so Claude reads only the sections a task needs
- **Change log** (`customizations.md`): how the theme differs from stock Horizon, ready to carry over to a new version
- **Version index** (`versions.json`): the Horizon versions stores may run, so every store stays on a version the docs cover
- **Placeholders** (`[agent: …]`) for the store, live theme, storefront password and Horizon version, filled in by setup

`CLAUDE.md` and `customizations.md` are shared by every version; each `THEME.md` describes exactly one.

## Quick Start

Run the setup from `@acodera/shopify-admin-mcp` (1.7.0 or later) in your project folder and choose **Full theme design**:

```bash
npx -y @acodera/shopify-admin-mcp@latest setup
```

It reads `versions.json`, asks for your store and credentials, checks them, and compares the store's live theme with the supported versions. On a match, it downloads that Horizon version into `./theme`, writes `CLAUDE.md`, `THEME.md` and `customizations.md` with the placeholders filled in, and writes `.mcp.json` with the settings `CLAUDE.md` expects: live theme writes, uploads from `./uploads`, and the Shopify Dev MCP.

If the live theme is older, newer or not Horizon, setup says which version to switch to and stops without writing `.mcp.json`. Since the Shopify theme store only installs the newest Horizon, it offers to save an upload-ready `horizon-<version>.zip` for **Online Store → Themes → Add theme → Upload zip file**.

Running it again keeps `customizations.md` and your own changes to `CLAUDE.md`, and asks before replacing `theme/` or `THEME.md`.

### Manual setup

Copy `CLAUDE.md`, `customizations.md` and the `THEME.md` for your Horizon version into your project folder, next to a `theme/` folder holding that version. Then fill in the `[agent: …]` placeholders; you can ask Claude to do it.

## Versions

`versions.json` decides which Horizon versions stores may run:

```json
{
  "current": "4.2.0",
  "versions": {
    "4.2.0": { "status": "supported", "horizonCommit": "f9aef27cd24723119fca896eea43bdeda9961022" }
  }
}
```

| Field | Description |
|-------|-------------|
| `current` | The version new and mismatched stores are moved to; must be `supported` |
| `status` | `supported`: stores on it pass setup's check. `draft`: its `THEME.md` is being written, and setup doesn't accept it yet. `retired`: no longer accepted; stores on it are told to move to `current` |
| `horizonCommit` | The [Shopify/horizon](https://github.com/Shopify/horizon) commit the spec was written against, which setup downloads; required for `supported` versions |

Usually only `current` is supported. During a move to a new version, mark both as supported for a while, then retire the old one.

### Adding a Horizon version

1. Write `versions/<version>/THEME.md` and add the version to `versions.json` as `draft`. Keep the section map at the top of `THEME.md` in step with its `##` headings.
2. When the spec is ready, set it to `supported` with its `horizonCommit`, and make it `current`.
3. Retire the old version once stores have moved.

## License

The Horizon theme itself isn't in this repository: its license doesn't allow redistributing it, so setup downloads it from [Shopify/horizon](https://github.com/Shopify/horizon).
