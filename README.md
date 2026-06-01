# Eclective Atlas — Claude Code plugin

Connect Claude Code to the **Eclective reporting** data in one step. This
repo is a Claude Code **plugin marketplace** containing a single plugin
(`eclective-atlas`) that bundles:

- the **MCP server** connection (`https://api.novus.pfeifferai-demo.com/mcp`,
  remote / Streamable HTTP) — authenticated with your Personal Access
  Token, read from the `ECLECTIVE_PAT` environment variable;
- the **`eclective-data` skill** — auto-loads when you ask about venue
  reporting and teaches the agent the data conventions (money is EUR
  cents, percentages are decimals, weeks are ISO Mondays) and the tools.

> Access is gated: the plugin is useless without a valid token, and a
> token only ever sees the venues its owner can. No secrets live in this
> repo — just the public API URL and an env-var placeholder.

## Install

1. **Mint a Personal Access Token** in the dashboard:
   **Settings → API tokens → New token**. Copy the `ec_pat_…` secret
   (shown once).

2. **Export it** (add to `~/.zshrc` to persist):

   ```bash
   export ECLECTIVE_PAT=ec_pat_…
   ```

3. **Add this marketplace and install**, from inside a Claude Code session:

   ```text
   /plugin marketplace add MaxPfeifferAI/eclective-atlas
   /plugin install eclective-atlas@eclective
   ```

Verify with `/mcp` (you should see `eclective` connected), then try:
*"List the venues I can see, then show last week's overview."*

To upgrade later: pull the latest and re-run `/plugin install`.

## Other agents (same token)

The MCP server is client-agnostic — any MCP client works with the same
PAT:

- **OpenAI Codex** — in `~/.codex/config.toml`:
  ```toml
  [mcp_servers.eclective]
  url = "https://api.novus.pfeifferai-demo.com/mcp"
  bearer_token_env_var = "ECLECTIVE_PAT"
  ```
- **Bare Claude Code (no plugin):**
  ```bash
  claude mcp add --transport http eclective \
    https://api.novus.pfeifferai-demo.com/mcp \
    --header "Authorization: Bearer $ECLECTIVE_PAT"
  ```

## Tools

`list_venues`, `list_weeks`, `get_overview`, `get_venue_weekly` /
`_daily` / `_reviews` / `_marketing` / `_web_analytics` / `_delivery` /
`_heatmap`, and `query_sql` (one read-only SELECT over the allowlisted
reporting tables, scoped to your venues). See the bundled
[`plugin/skills/eclective-data/SKILL.md`](plugin/skills/eclective-data/SKILL.md).
