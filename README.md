# claude-horizon

Project instructions for working on Shopify's [Horizon](https://github.com/Shopify/horizon) theme with [Claude Code](https://claude.com/claude-code) and [`@acodera/shopify-admin-mcp`](https://github.com/acoderacom/shopify-admin-mcp).

```
claude-horizon/
├─ versions.json            which Horizon versions stores may run
├─ CLAUDE.md                working rules: store access, editing and pushing theme files, design rules, logging
├─ customizations.md        change log template: how the theme differs from stock Horizon
└─ versions/
   └─ 4.2.0/THEME.md        technical spec of stock Horizon 4.2.0
```

`CLAUDE.md` and `customizations.md` are shared by every version. `THEME.md` describes exactly one Horizon version, so each supported version has its own folder.

## Versions

`versions.json` decides which Horizon versions stores may run, so every store stays on a version the docs cover:

```json
{
  "current": "4.2.0",
  "versions": {
    "4.2.0": { "status": "supported", "horizonCommit": "f9aef27cd24723119fca896eea43bdeda9961022" }
  }
}
```

- `current`: the version new and mismatched stores are moved to. It must be `supported`.
- `status`:
  - `supported`: stores on this version pass setup's check.
  - `draft`: its `THEME.md` is being written, and setup doesn't accept it yet.
  - `retired`: no longer accepted; stores on it are told to move to `current`.
- `horizonCommit`: the [Shopify/horizon](https://github.com/Shopify/horizon) commit the spec was written against. Setup downloads exactly this commit. Required for `supported` versions.

Usually only `current` is supported. During a move to a new version, mark both as supported for a while, then retire the old one.

### Adding a Horizon version

1. Write `versions/<version>/THEME.md` for the new version, and add the version to `versions.json` as `draft`.
2. When the spec is ready, set it to `supported` with its `horizonCommit`, and make it `current`.
3. Retire the old version once stores have moved.

## Using it

In your project folder, run the setup from `@acodera/shopify-admin-mcp` (1.7.0 or later) and choose **Full theme design**:

```bash
npx -y @acodera/shopify-admin-mcp@latest setup
```

Setup then:

1. reads `versions.json`;
2. asks for your store and credentials, and checks them;
3. compares your store's live theme with the supported versions:
   - if it matches, setup goes on with that version;
   - if it's older, newer or not Horizon, setup says which version to switch to and stops without setting up the project. The Shopify theme store only installs the newest Horizon, so setup offers to save an upload-ready `horizon-<current>.zip` in the folder, for you to upload in the Shopify admin (**Online Store → Themes → Add theme → Upload zip file**) and publish;
4. downloads that Horizon version into `theme/`;
5. writes `CLAUDE.md`, `THEME.md` and `customizations.md`, filling in the `[agent: …]` placeholders: the store, the live theme's version and ID, the storefront password, and the Horizon version;
6. writes `.mcp.json` with the settings `CLAUDE.md` expects: live theme writes, uploads from `uploads/`, and the Shopify Dev MCP.

Running it again keeps `customizations.md` and your own changes to `CLAUDE.md`, and asks before replacing `theme/` or `THEME.md`.

### By hand

Copy `CLAUDE.md`, `customizations.md` and the `THEME.md` for your Horizon version into your project folder, next to a `theme/` folder holding that version. Then fill in the `[agent: …]` placeholders; you can ask Claude to do it.

The Horizon theme itself isn't in this repository. Its license doesn't allow redistributing it, so download it from [Shopify/horizon](https://github.com/Shopify/horizon).
