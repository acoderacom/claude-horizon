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

Put the three files in your project folder, next to a `theme/` folder holding the same Horizon version, then fill in the `[agent: …]` placeholders in `CLAUDE.md`: the store, the live theme's version and ID, and the storefront password. You can ask Claude to fill them in.

The Horizon theme itself isn't in this repository. Its license doesn't allow redistributing it, so download it from [Shopify/horizon](https://github.com/Shopify/horizon).
