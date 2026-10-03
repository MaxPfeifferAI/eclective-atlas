---
name: eclective-data
description: >-
  Query Eclective Hospitality's weekly venue reporting data — revenue,
  covers, budgets, KPIs, labour, COGS, reviews, email/web marketing,
  delivery, booking heatmaps and Mystery Diner coordination — through the `eclective` MCP server.
  Use whenever the user asks about venue performance, a weekly report,
  portfolio/group totals, ad-hoc SQL, diner applications and approvals, matching,
  visit scheduling, briefings, reminders or report publication.
---

# Eclective reporting data

The **`eclective`** MCP server provides venue-scoped reporting reads and
superadmin-only Mystery Diner operations. Coordinator tools can create or edit
assignments, email diners, send push notifications and change publication.
Use them only within the user's requested scope. Reporting SQL remains read-only;
never use SQL to bypass the coordinator endpoints or their role checks.

If connecting to MCP fails with **401**, check for a missing, expired or revoked
PAT. The user can mint one at the dashboard **Settings → API tokens** page and export it:
`export ECLECTIVE_PAT=ec_pat_…` (the plugin's MCP config reads that env
var). The secret is shown only once at creation. If reporting works but Mystery
Diner tools return 401/403, the user may lack management access. Replacing the PAT
does not grant a role. The API checks current roles, including additive grants,
on every call. Never request or expose the service API key.

## Conventions — read these before interpreting any number

- **Money is EUR cents.** Divide by 100 for euros. `1234567` = €12,345.67.
- **Percentages are decimals.** `0.15` means 15%. Do not multiply by 100
  at the data boundary — only when displaying.
- **`week_start` is the Monday of an ISO week**, `YYYY-MM-DD`. There is no
  "week number" parameter — resolve to the Monday date first.

## Start reporting work with discovery

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


## Mystery Diner coordinator

Atlas is the companion frontend for viewing and editing the same saved briefings.
All tools call the same REST API, with the caller's identity and existing
superadmin permissions. No interactive browser session is required for these
calls. In a headless run, use the caller's task and configured tool permissions as
authorization; do not add an approval pause for actions they already requested.
Do not contact diners during an audit or read-only reporting task.

1. `list_open_venues()` to get a schedulable venue slug. This registry includes
   external venues excluded from reporting's `list_venues()`. Never change venue
   eligibility to make an assignment work.
2. `match_diners(venue_slug, visit_date, limit?)` to see ranked eligible diners.
   Explain the reasons for a recommendation. Availability is free text: read it
   before selecting a diner. Do not claim the venue is booked by scheduling it.
3. `get_briefing_defaults(venue_slug, playbook?)` to inspect the default briefing.
   Premium EUR180, casual EUR120, bars EUR80; money arguments are integer cents.
   There is no default booking URL: use a verified URL for the selected venue.
4. `schedule_visit(diner_id, venue_slug, visit_date, client_schedule_id,
   playbook?, briefing_overrides?)` creates the assignment with defaults plus
   supplied overrides. Generate a UUID for `client_schedule_id` and reuse the
   same ID and request on a timeout/retry. A new UUID means a new assignment.
   Scheduling notifies the diner. Return the saved briefing, not a guessed draft.
5. `update_diner_visit(visit_id, briefing_overrides?, visit_date?, status?)`
   merges only supplied briefing fields; null clears a field. The other values
   remain unchanged. Changes may notify the diner. Submitted visits are immutable.

Pool and progress tools:

| Tool | Purpose |
| --- | --- |
| `list_diners(status?)` | Profiles, completeness and active/applicant/inactive status |
| `approve_diner(id)` | Activate an applicant and send the approval email |
| `set_diner_status(id, status)` | Change pool status; use approve_diner for applicant approval |
| `list_visits(week_start?, status?)` | Progress timestamps, briefing and publication; week_start is Monday |
| `get_diner_visit(visit_id)` | Saved assignment and capture evidence |
| `nudge_diner(visit_id)` | State-specific push + email; one per visit/state/day in Dublin time |
| `get_diner_report(visit_id)` | Calculated scores, recorded answers and available AI narrative |
| `unpublish_report(visit_id)` | Remove the report from venue reporting while preserving its evidence |
| `republish_report(visit_id)` | Restore an unpublished submitted report |

A valid submission publishes automatically. `approved_at` indicates publication;
there is no routine report approval gate. The AI summary may arrive later and
never changes scores or publication. Empty narrative does not mean unpublished.
If the user unpublishes a report, retries and later narrative writes must leave
it unpublished. Never invent answers or ratings to make a report look complete.

Nudge responses contain independent push/email statuses. `sent` means accepted
by the provider, not read by the diner. Report `failed`, `skipped`, `no_token`,
`partial` or `pending` honestly. Repeating the same nudge that day returns its
recorded outcomes; do not promise a retry will resend it. There is no scheduler.
