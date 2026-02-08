---
name: posthog-read
description: This skill should be used when the user mentions "posthog" or asks to "check analytics", "query events", "view page views", "list top pages", "check funnels", "get user properties", "list cohorts", "view session recordings", "check feature flags", "run HogQL query", "get retention data", "view trends", "check errors", "list projects", or any read-only PostHog API operation. Uses the PostHog MCP server.
---

## PostHog MCP Server

All PostHog operations use the **PostHog MCP server** (`https://mcp.posthog.com/mcp`) registered manually at user scope. Auth is handled via OAuth — you authenticate once in the browser and the token persists across sessions.

> **Note (Feb 2026):** We use manual MCP registration instead of the official PostHog Claude Code plugin (`posthog@claude-plugins-official`). The plugin was buggy and untested when this skill was set up — its nested MCP server failed to load due to an incorrect `.mcp.json` format and likely lack of HTTP MCP support in the plugin system. Manual registration works reliably across both terminal and VSCode sessions.

Before running any PostHog queries, verify the MCP is connected by checking if PostHog tools (like `mcp__posthog__query-run`) are available. If they are not:

1. **MCP not registered** — tell the user: "The PostHog MCP server isn't registered. Run this in your terminal, then restart Claude Code:"

   ```bash
   claude mcp add --transport http posthog https://mcp.posthog.com/mcp -s user
   ```

   This registers the MCP at user scope so it's available in all projects (terminal and VSCode).

2. **MCP registered but tools not loading** — tell the user: "The PostHog MCP is registered but needs OAuth authentication. Restart Claude Code to trigger the browser login flow. You only need to do this once."

## Projects

On the **first PostHog query in a session**, determine which project to use:

1. **Call `projects-get`** to list available projects and their IDs
2. **Auto-infer from context** — check the current working directory name, git remote, or repo name for hints:
   - Contains "stage" or "staging" → use the Staging project
   - Contains "prod" → use the Production project
3. **If ambiguous** — ask the user which project to use for this session
4. **Remember the choice** — use the same project for the rest of the session without re-asking

Only call `switch-project` when needed (first query or user requests a change). Pass the project ID directly from the `projects-get` response.

## HogQL Query Patterns

Use the MCP's query tool with HogQL for flexible analytics. Always add `LIMIT` to avoid huge payloads.

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
| MCP not connected | Run `claude mcp add --transport http posthog https://mcp.posthog.com/mcp -s user` and restart |
| OAuth expired | Restart Claude Code to trigger re-auth in the browser |
| Wrong project | Call `projects-get` to list available projects, ask user or auto-infer from directory context |
| Empty results | Widen date range, check event names with an event count query first |
| Rate limited (429) | Wait and retry; limits are per-org (240/min analytics, 2400/hr queries) |

## Recommended Auto-Allow Settings

Add these read-only PostHog MCP permissions to `~/.claude/settings.json` (global, applies to all projects). All tools below are strictly read-only — write operations (create, update, delete) are excluded and will still require manual approval.

```json
"mcp__posthog__query-run",
"mcp__posthog__query-generate-hogql-from-question",
"mcp__posthog__insight-get",
"mcp__posthog__insight-query",
"mcp__posthog__insights-get-all",
"mcp__posthog__dashboard-get",
"mcp__posthog__dashboards-get-all",
"mcp__posthog__feature-flag-get-all",
"mcp__posthog__feature-flag-get-definition",
"mcp__posthog__experiment-get",
"mcp__posthog__experiment-get-all",
"mcp__posthog__experiment-results-get",
"mcp__posthog__error-details",
"mcp__posthog__list-errors",
"mcp__posthog__docs-search",
"mcp__posthog__organization-details-get",
"mcp__posthog__organizations-get",
"mcp__posthog__projects-get",
"mcp__posthog__survey-get",
"mcp__posthog__surveys-get-all",
"mcp__posthog__surveys-global-stats",
"mcp__posthog__survey-stats",
"mcp__posthog__logs-query",
"mcp__posthog__logs-list-attributes",
"mcp__posthog__logs-list-attribute-values",
"mcp__posthog__entity-search"
```

Add these to the `permissions.allow` array in settings.json.
