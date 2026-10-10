# claude-horizon

Project instructions for working on Shopify's [Horizon](https://github.com/Shopify/horizon) theme with [Claude Code](https://claude.com/claude-code) and [`@acodera/shopify-admin-mcp`](https://github.com/acoderacom/shopify-admin-mcp).

| File | What it is |
|------|------------|
| `CLAUDE.md` | Working rules: store access, how to edit and push a theme file, design rules, logging |
| `THEME.md` | Technical spec of one stock Horizon version: how it's built and how to customize it |
| `customizations.md` | Change log of how the theme differs from stock Horizon, with the entry format |

## Versions

`THEME.md` covers exactly one Horizon version. Each tag pairs the docs with the version they cover; `horizon-4.2.0` is for Horizon 4.2.0. Setup reads these tags to check that your store's live theme is a version the docs cover.

## Using it

In your project folder, run the setup from [`@acodera/shopify-admin-mcp`](https://github.com/acoderacom/shopify-admin-mcp) (1.5.0 or later) and choose **Full theme design**:

```bash
npx -y @acodera/shopify-admin-mcp@latest setup
```

Setup then:

1. checks this template's Horizon versions on GitHub;
2. asks for your store and credentials, and checks them;
3. compares your store's live theme with the template. If it's a Horizon version the template covers, setup says so and recommends that version. If it's older, newer or not Horizon, setup says which version to upgrade or downgrade to in the Shopify admin and stops without writing anything;
4. downloads that Horizon version into `theme/`;
5. writes the three files from the matching tag and fills in the `[agent: …]` placeholders in `CLAUDE.md`: the store, the live theme's version and ID, and the storefront password;
6. writes `.mcp.json` with the settings `CLAUDE.md` expects: live theme writes, uploads from `uploads/`, and the Shopify Dev MCP.

Running it again keeps `customizations.md` and your own changes to `CLAUDE.md`, and asks before replacing `theme/` or `THEME.md`.

### By hand

Put the three files in your project folder, next to a `theme/` folder holding the same Horizon version, then fill in the `[agent: …]` placeholders in `CLAUDE.md`. You can ask Claude to fill them in.

The Horizon theme itself isn't in this repository. Its license doesn't allow redistributing it, so download it from [Shopify/horizon](https://github.com/Shopify/horizon).
