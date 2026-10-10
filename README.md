# claude-horizon

Project instructions for working on Shopify's [Horizon](https://github.com/Shopify/horizon) theme with [Claude Code](https://claude.com/claude-code) and [`@acodera/shopify-admin-mcp`](https://github.com/acoderacom/shopify-admin-mcp).

| File | What it is |
|------|------------|
| `CLAUDE.md` | Working rules: store access, how to edit and push a theme file, design rules, logging |
| `THEME.md` | Technical spec of one stock Horizon version: how it's built and how to customize it |
| `customizations.md` | Change log of how the theme differs from stock Horizon, with the entry format |

## Versions

`THEME.md` covers exactly one Horizon version. Each tag pairs the docs with the version they cover; `horizon-4.2.0` is for Horizon 4.2.0.

## Using it

In your project folder, run the setup from [`@acodera/shopify-admin-mcp`](https://github.com/acoderacom/shopify-admin-mcp) (1.4.0 or later) and answer Yes to "Set up a Horizon theme project here?":

```bash
npx -y @acodera/shopify-admin-mcp@latest setup
```

Setup then:

- downloads the Horizon version you pick into `theme/`, with your store's live version preselected;
- writes the three files from the tag for that version (or the newest docs, with a warning, when that version has none);
- fills in the `[agent: …]` placeholders in `CLAUDE.md`: the store, the live theme's version and ID, and the storefront password;
- writes `.mcp.json` with the settings `CLAUDE.md` expects: live theme writes, uploads from `uploads/`, and the Shopify Dev MCP.

Running it again keeps `customizations.md` and your own changes to `CLAUDE.md`, and asks before replacing `theme/` or `THEME.md`.

### By hand

Put the three files in your project folder, next to a `theme/` folder holding the same Horizon version, then fill in the `[agent: …]` placeholders in `CLAUDE.md`. You can ask Claude to fill them in.

The Horizon theme itself isn't in this repository. Its license doesn't allow redistributing it, so download it from [Shopify/horizon](https://github.com/Shopify/horizon).
