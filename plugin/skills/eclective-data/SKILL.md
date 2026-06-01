---
name: eclective-data
description: >-
  Query Eclective Hospitality's weekly venue reporting data — revenue,
  covers, budgets, KPIs, labour, COGS, reviews, email/web marketing,
  delivery, and booking heatmaps — through the `eclective` MCP server.
  Use whenever the user asks about venue performance, a weekly report,
  portfolio/group totals, or wants ad-hoc SQL over the reporting tables.
---

# Eclective reporting data

You have access to the **`eclective`** MCP server: read-only access to
Eclective Hospitality's weekly venue reporting. Every call is scoped to
the venues the caller's Personal Access Token (PAT) can see — you never
see more than the token's owner can.

If the tools error with **401 / "invalid or missing Personal Access
Token"**, the user hasn't set their PAT. Tell them to mint one at the
dashboard **Settings → API tokens** page and export it:
`export ECLECTIVE_PAT=ec_pat_…` (the plugin's MCP config reads that env
var). The secret is shown only once at creation.

## Conventions — read these before interpreting any number

- **Money is EUR cents.** Divide by 100 for euros. `1234567` = €12,345.67.
- **Percentages are decimals.** `0.15` means 15%. Do not multiply by 100
  at the data boundary — only when displaying.
- **`week_start` is the Monday of an ISO week**, `YYYY-MM-DD`. There is no
  "week number" parameter — resolve to the Monday date first.

## Start every session with discovery

1. `list_weeks` → which weeks have data (newest is usually the one to use
   if the user says "last week" / doesn't specify).
2. `list_venues` → the venue **slugs** + names + category + status. All
   per-venue tools take a `slug`, not a display name.

Don't guess a slug or a week — look them up.

## Tools

Structured reads (each maps to a reporting endpoint, already venue-scoped):

| Tool | Returns |
| --- | --- |
| `list_venues()` | venues you can access: slug, name, category, status |
| `list_weeks()` | ISO-Monday `week_start` dates that have data |
| `get_overview(week_start, category?)` | portfolio rollup: per-venue cards + group totals. `category` ∈ premium, casual, bars, qsr, cinema |
| `get_venue_weekly(slug, week_start)` | full weekly report: revenue, covers, budget, KPIs, labour, COGS, briefing |
| `get_venue_daily(slug, week_start)` | per-day revenue + covers |
| `get_venue_reviews(slug, week_start)` | Google + OpenTable review snapshot |
| `get_venue_marketing(slug, week_start)` | Klaviyo email marketing metrics |
| `get_venue_web_analytics(slug, week_start)` | GA4 sessions + booking conversion |
| `get_venue_delivery(slug, week_start)` | Deliveroo + Just Eat channels |
| `get_venue_heatmap(slug, weeks?)` | day-of-week × hour avg covers over the last N weeks (default 13) |

Prefer a structured tool when one fits the question — it's pre-shaped and
cheaper than SQL. Reach for `query_sql` for cross-venue aggregation,
trends, rankings, or anything the structured tools don't cover.

### `query_sql(sql, reason)`

One read-only `SELECT` (or `WITH`/CTE) over the reporting tables. Returns
`{columns, rows, row_count, truncated, elapsed_ms}`. Provide a short
`reason` — it's written to the audit log.

- **One statement only.** No `INSERT/UPDATE/DELETE/DDL`, no
  schema-qualified names, no `set_config`/`current_setting`/`pg_read_file`.
  A `LIMIT` is injected for you; a 15s statement timeout applies.
- Results are **automatically filtered to your permitted venues** by
  Postgres row-level security — you don't (and can't) add a venue filter
  to widen access.
- **Allowlisted tables:** `venues`, `revenue`, `revenue_daily`, `covers`,
  `budgets`, `kpis`, `labour`, `cogs`, `marketing_email`, `marketing_web`,
  `reviews`, `reservations`, `revenue_historical`, `briefings`.

Example — top 5 venues by revenue for a week (remember: cents):

```sql
SELECT v.name, r.revenue_cents
FROM revenue r
JOIN venues v ON v.slug = r.venue_slug
WHERE r.week_start = '2026-05-25'
ORDER BY r.revenue_cents DESC
LIMIT 5
```

When you present figures to the user, convert cents → euros and decimals
→ percent. State the `week_start` you used so they can confirm the period.
