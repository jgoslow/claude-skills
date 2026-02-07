---
name: posthog-read
description: This skill should be used when the user asks to "check analytics", "query events", "view page views", "check funnels", "get user properties", "list cohorts", "view session recordings", "check feature flags", "run HogQL query", "get retention data", "view trends", or any read-only PostHog API operation. Uses the official PostHog plugin.
---

## PostHog Plugin

All PostHog operations use the **official PostHog plugin** (`posthog@claude-plugins-official`). Auth is handled via OAuth — you authenticate once in the browser and the token persists across sessions.

Before running any PostHog queries, verify the plugin is connected by checking if PostHog tools (like `posthog:query`) are available. If they are not:

1. **Plugin not installed** — tell the user: "The PostHog plugin isn't installed. Run `claude plugin install posthog` in your terminal, then restart Claude Code."
2. **Plugin installed but tools not loading** — tell the user: "The PostHog plugin is installed but needs OAuth authentication. Run `/posthog:query` or restart Claude Code to trigger the browser login flow. You only need to do this once."

## Projects

When the user mentions an environment, use the corresponding project:

| Environment | Project ID env var |
|---|---|
| **Production** (default) | `$POSTHOG_PROD_PROJECT_ID` |
| **Staging** | `$POSTHOG_STAGE_PROJECT_ID` |
| **Development** | `$POSTHOG_DEV_PROJECT_ID` |

Project IDs are stored in the project's `.env` file. If missing, tell the user:

> Add PostHog project IDs to your `.env`:
>
> ```
> POSTHOG_PROD_PROJECT_ID=your_prod_id
> POSTHOG_STAGE_PROJECT_ID=your_stage_id
> POSTHOG_DEV_PROJECT_ID=your_dev_id
> ```
>
> Find project IDs in the URL: `https://us.posthog.com/project/<ID>/`

## HogQL Query Patterns

Use the plugin's query tool with HogQL for flexible analytics. Always add `LIMIT` to avoid huge payloads.

### Page Views

```sql
SELECT timestamp, properties.$current_url, properties.$browser, properties.$os
FROM events WHERE event = '$pageview'
ORDER BY timestamp DESC LIMIT 100
```

### Count Events by Type

```sql
SELECT event, count() as count
FROM events WHERE timestamp > now() - INTERVAL 7 DAY
GROUP BY event ORDER BY count DESC LIMIT 50
```

### Top Pages (Last 7 Days)

```sql
SELECT properties.$current_url as url, count() as views, count(DISTINCT distinct_id) as unique_users
FROM events WHERE event = '$pageview' AND timestamp > now() - INTERVAL 7 DAY
GROUP BY url ORDER BY views DESC LIMIT 25
```

### Unique Users by Event

```sql
SELECT count(DISTINCT distinct_id) as unique_users
FROM events WHERE event = '$pageview' AND timestamp > now() - INTERVAL 30 DAY
```

### Events with Property Filters

```sql
SELECT timestamp, distinct_id, properties.$current_url
FROM events WHERE event = '$pageview' AND properties.$browser = 'Chrome'
AND timestamp > now() - INTERVAL 1 DAY LIMIT 50
```

### Daily Event Counts

```sql
SELECT toDate(timestamp) as day, count() as events
FROM events WHERE event = '$pageview' AND timestamp > now() - INTERVAL 30 DAY
GROUP BY day ORDER BY day
```

### Query Person Properties

```sql
SELECT distinct_id, person.properties.email, person.properties.$initial_browser, count() as event_count
FROM events WHERE timestamp > now() - INTERVAL 7 DAY
GROUP BY distinct_id, person.properties.email, person.properties.$initial_browser
ORDER BY event_count DESC LIMIT 50
```

### Users Who Performed a Specific Event

```sql
SELECT DISTINCT distinct_id, person.properties.email
FROM events WHERE event = 'purchase' AND timestamp > now() - INTERVAL 30 DAY LIMIT 100
```

## TrendsQuery Patterns

Structured trend queries with breakdowns and multiple series.

### Basic Trend

```json
{
  "kind": "TrendsQuery",
  "series": [{"event": "$pageview"}],
  "dateRange": {"date_from": "-30d"},
  "interval": "day"
}
```

### Trend with Breakdown

```json
{
  "kind": "TrendsQuery",
  "series": [{"event": "$pageview"}],
  "dateRange": {"date_from": "-7d"},
  "interval": "day",
  "breakdownFilter": {
    "breakdowns": [{"property": "$browser", "type": "event"}]
  }
}
```

### Multiple Series (Compare Events)

```json
{
  "kind": "TrendsQuery",
  "series": [
    {"event": "$pageview"},
    {"event": "sign_up"},
    {"event": "purchase"}
  ],
  "dateRange": {"date_from": "-30d"},
  "interval": "week"
}
```

### Date Range Patterns

- Relative: `"-7d"`, `"-30d"`, `"-90d"`, `"-12m"`
- Absolute: `"2025-01-01"` to `"2025-01-31"`
- Intervals: `"hour"`, `"day"`, `"week"`, `"month"`

## FunnelsQuery Patterns

### Multi-Step Funnel

```json
{
  "kind": "FunnelsQuery",
  "series": [
    {"event": "$pageview"},
    {"event": "sign_up"},
    {"event": "purchase"}
  ],
  "dateRange": {"date_from": "-30d"},
  "funnelsFilter": {
    "funnelWindowInterval": 14,
    "funnelWindowIntervalUnit": "day"
  }
}
```

### Funnel with Property Filter

```json
{
  "kind": "FunnelsQuery",
  "series": [
    {"event": "viewed_product"},
    {"event": "added_to_cart"},
    {"event": "checkout_completed"}
  ],
  "dateRange": {"date_from": "-7d"},
  "properties": {
    "type": "AND",
    "values": [{"type": "event", "key": "$browser", "value": "Chrome", "operator": "exact"}]
  }
}
```

## RetentionQuery Patterns

```json
{
  "kind": "RetentionQuery",
  "retentionFilter": {
    "targetEntity": {"id": "$pageview", "type": "events"},
    "returningEntity": {"id": "$pageview", "type": "events"},
    "period": "Week",
    "totalIntervals": 8
  },
  "dateRange": {"date_from": "-8w"}
}
```

## PathsQuery Patterns

```json
{
  "kind": "PathsQuery",
  "dateRange": {"date_from": "-7d"},
  "pathsFilter": {
    "includeEventTypes": ["$pageview"],
    "stepLimit": 5
  }
}
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Plugin not connected | Run `claude plugin install posthog` and complete OAuth |
| Wrong project | Check which project ID env var is being used |
| Empty results | Widen date range, check event names with an event count query first |
| Rate limited (429) | Wait and retry; limits are per-org (240/min analytics, 2400/hr queries) |

## Recommended Auto-Allow Settings

Add these read-only PostHog plugin permissions to `~/.claude/settings.json` (global, applies to all projects). All tools below are strictly read-only — write operations (create, update, delete) are excluded and will still require manual approval.

```json
"mcp__plugin_posthog_posthog__query-run",
"mcp__plugin_posthog_posthog__query-generate-hogql-from-question",
"mcp__plugin_posthog_posthog__insight-get",
"mcp__plugin_posthog_posthog__insight-query",
"mcp__plugin_posthog_posthog__insights-get-all",
"mcp__plugin_posthog_posthog__dashboard-get",
"mcp__plugin_posthog_posthog__dashboards-get-all",
"mcp__plugin_posthog_posthog__feature-flag-get-all",
"mcp__plugin_posthog_posthog__feature-flag-get-definition",
"mcp__plugin_posthog_posthog__experiment-get",
"mcp__plugin_posthog_posthog__experiment-get-all",
"mcp__plugin_posthog_posthog__experiment-results-get",
"mcp__plugin_posthog_posthog__error-details",
"mcp__plugin_posthog_posthog__list-errors",
"mcp__plugin_posthog_posthog__docs-search",
"mcp__plugin_posthog_posthog__organization-details-get",
"mcp__plugin_posthog_posthog__organizations-get",
"mcp__plugin_posthog_posthog__projects-get",
"mcp__plugin_posthog_posthog__survey-get",
"mcp__plugin_posthog_posthog__surveys-get-all",
"mcp__plugin_posthog_posthog__surveys-global-stats",
"mcp__plugin_posthog_posthog__survey-stats",
"mcp__plugin_posthog_posthog__logs-query",
"mcp__plugin_posthog_posthog__logs-list-attributes",
"mcp__plugin_posthog_posthog__logs-list-attribute-values",
"mcp__plugin_posthog_posthog__entity-search"
```

Add these to the `permissions.allow` array in settings.json.
