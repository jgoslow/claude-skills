---
name: posthog-read
description: This skill should be used when the user asks to "check analytics", "query events", "view page views", "check funnels", "get user properties", "list cohorts", "view session recordings", "check feature flags", "run HogQL query", "get retention data", "view trends", or any read-only PostHog API operation. Requires POSTHOG_USER_READ_TOKEN.
---

## Authentication

Define a `phog` helper function once at the start of the session. All subsequent commands use it.

```bash
phog() { curl -s -H "Authorization: Bearer $POSTHOG_USER_READ_TOKEN" -H "Content-Type: application/json" "$@"; }
```

- **Base URL**: `https://us.posthog.com`
- **Project ID**: `124499`
- **API key scope**: Read-only. POST requests to the Query API execute read queries only.

Verify access:

```bash
phog "https://us.posthog.com/api/projects/124499/" | jq '{name, id}'
```

### Rate Limits

| Endpoint type | Limit |
|---|---|
| Analytics (insights, persons, recordings) | 240/min, 1200/hr |
| Query API (`/query/`) | 2400/hr |
| CRUD endpoints | 480/min, 4800/hr |

Limits are per-organization, not per-key.

## Performance Rules

1. **Use HogQL for analytics** — most powerful and flexible; one query replaces many REST calls
2. **Use REST endpoints for listing resources** — persons, flags, cohorts, recordings
3. **Always add `LIMIT`** to HogQL queries — avoid huge payloads
4. **Use timestamp-based pagination** for large datasets (not OFFSET)
5. **Pipe through `jq`** for client-side filtering
6. **Run independent queries in parallel** with `&` + `wait`
7. **Queries are cached** by default — repeated calls are fast

## Query API — HogQL

The Query API is the primary tool for analytics. All queries POST to:

```
POST https://us.posthog.com/api/projects/124499/query/
```

### Page Views

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT timestamp, properties.$current_url, properties.$browser, properties.$os FROM events WHERE event = '"'"'$pageview'"'"' ORDER BY timestamp DESC LIMIT 100"
    }
  }' | jq '.results'
```

### Count Events by Type

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT event, count() as count FROM events WHERE timestamp > now() - INTERVAL 7 DAY GROUP BY event ORDER BY count DESC LIMIT 50"
    }
  }' | jq '.results'
```

### Top Pages (Last 7 Days)

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT properties.$current_url as url, count() as views, count(DISTINCT distinct_id) as unique_users FROM events WHERE event = '"'"'$pageview'"'"' AND timestamp > now() - INTERVAL 7 DAY GROUP BY url ORDER BY views DESC LIMIT 25"
    }
  }' | jq '.results'
```

### Unique Users by Event

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT count(DISTINCT distinct_id) as unique_users FROM events WHERE event = '"'"'$pageview'"'"' AND timestamp > now() - INTERVAL 30 DAY"
    }
  }' | jq '.results'
```

### Events with Property Filters

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT timestamp, distinct_id, properties.$current_url FROM events WHERE event = '"'"'$pageview'"'"' AND properties.$browser = '"'"'Chrome'"'"' AND timestamp > now() - INTERVAL 1 DAY LIMIT 50"
    }
  }' | jq '.results'
```

### Daily Event Counts

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT toDate(timestamp) as day, count() as events FROM events WHERE event = '"'"'$pageview'"'"' AND timestamp > now() - INTERVAL 30 DAY GROUP BY day ORDER BY day"
    }
  }' | jq '.results'
```

### Query Person Properties

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT distinct_id, person.properties.email, person.properties.$initial_browser, count() as event_count FROM events WHERE timestamp > now() - INTERVAL 7 DAY GROUP BY distinct_id, person.properties.email, person.properties.$initial_browser ORDER BY event_count DESC LIMIT 50"
    }
  }' | jq '.results'
```

### Users Who Performed a Specific Event

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "HogQLQuery",
      "query": "SELECT DISTINCT distinct_id, person.properties.email FROM events WHERE event = '"'"'purchase'"'"' AND timestamp > now() - INTERVAL 30 DAY LIMIT 100"
    }
  }' | jq '.results'
```

## Query API — TrendsQuery

Structured trend queries with breakdowns and multiple series.

### Basic Trend (Daily Page Views, Last 30 Days)

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "TrendsQuery",
      "series": [{"event": "$pageview"}],
      "dateRange": {"date_from": "-30d"},
      "interval": "day"
    }
  }' | jq '.results'
```

### Trend with Breakdown by Browser

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "TrendsQuery",
      "series": [{"event": "$pageview"}],
      "dateRange": {"date_from": "-7d"},
      "interval": "day",
      "breakdownFilter": {
        "breakdowns": [{"property": "$browser", "type": "event"}]
      }
    }
  }' | jq '.results'
```

### Multiple Series (Compare Events)

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "TrendsQuery",
      "series": [
        {"event": "$pageview"},
        {"event": "sign_up"},
        {"event": "purchase"}
      ],
      "dateRange": {"date_from": "-30d"},
      "interval": "week"
    }
  }' | jq '.results'
```

### Date Range Patterns

- Relative: `"-7d"`, `"-30d"`, `"-90d"`, `"-12m"`
- Absolute: `"2025-01-01"` to `"2025-01-31"`
- Intervals: `"hour"`, `"day"`, `"week"`, `"month"`

## Query API — FunnelsQuery

### Multi-Step Funnel

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
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
  }' | jq '.results'
```

### Funnel with Property Filter

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
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
  }' | jq '.results'
```

## Query API — Retention & Paths

### Retention Query

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "RetentionQuery",
      "retentionFilter": {
        "targetEntity": {"id": "$pageview", "type": "events"},
        "returningEntity": {"id": "$pageview", "type": "events"},
        "period": "Week",
        "totalIntervals": 8
      },
      "dateRange": {"date_from": "-8w"}
    }
  }' | jq '.results'
```

### Paths Query (User Journeys)

```bash
phog -X POST "https://us.posthog.com/api/projects/124499/query/" \
  -d '{
    "query": {
      "kind": "PathsQuery",
      "dateRange": {"date_from": "-7d"},
      "pathsFilter": {
        "includeEventTypes": ["$pageview"],
        "stepLimit": 5
      }
    }
  }' | jq '.results'
```

## REST Endpoints — Persons

```bash
# List persons (paginated)
phog "https://us.posthog.com/api/projects/124499/persons/?limit=100" | jq '.results'

# Search by email
phog "https://us.posthog.com/api/projects/124499/persons/?search=user@example.com" | jq '.results'

# Filter by distinct_id
phog "https://us.posthog.com/api/projects/124499/persons/?distinct_id=user123" | jq '.results'

# Get a specific person by ID
phog "https://us.posthog.com/api/projects/124499/persons/PERSON_ID/" | jq '.'
```

Pagination: follow the `next` URL in the response to get subsequent pages.

## REST Endpoints — Session Recordings, Feature Flags, Cohorts

```bash
# List session recordings
phog "https://us.posthog.com/api/projects/124499/session_recordings/?limit=20" | jq '.results'

# View a specific recording
phog "https://us.posthog.com/api/projects/124499/session_recordings/RECORDING_ID/" | jq '.'

# List feature flags
phog "https://us.posthog.com/api/projects/124499/feature_flags/" | jq '.results'

# View a specific flag
phog "https://us.posthog.com/api/projects/124499/feature_flags/FLAG_ID/" | jq '.'

# List cohorts
phog "https://us.posthog.com/api/projects/124499/cohorts/" | jq '.results'

# View a specific cohort
phog "https://us.posthog.com/api/projects/124499/cohorts/COHORT_ID/" | jq '.'
```

## REST Endpoints — Insights, Dashboards, Actions, Definitions

```bash
# List saved insights
phog "https://us.posthog.com/api/projects/124499/insights/?limit=50" | jq '.results[] | {id, name, short_id}'

# View a specific insight
phog "https://us.posthog.com/api/projects/124499/insights/INSIGHT_ID/" | jq '.'

# List dashboards
phog "https://us.posthog.com/api/projects/124499/dashboards/" | jq '.results[] | {id, name}'

# List actions
phog "https://us.posthog.com/api/projects/124499/actions/" | jq '.results[] | {id, name}'

# List event definitions (discover available events)
phog "https://us.posthog.com/api/projects/124499/event_definitions/" | jq '.results[] | {name, volume_30_day}'

# List property definitions (discover available properties)
phog "https://us.posthog.com/api/projects/124499/property_definitions/" | jq '.results[] | {name, property_type}'

# List groups and group types
phog "https://us.posthog.com/api/projects/124499/groups_types/" | jq '.'

phog "https://us.posthog.com/api/projects/124499/groups/?group_type_index=0&limit=50" | jq '.results'
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Auth failure (401) | Verify `POSTHOG_USER_READ_TOKEN` is set and starts with `phx_` |
| 403 Forbidden | Key lacks required scope — check key scopes in PostHog settings |
| Rate limited (429) | Wait and retry; check org-wide usage |
| Empty results | Verify project ID `124499`, check date range, try broader filters |
| Stats 202 response | PostHog is computing — retry after a few seconds |
| Pagination | Follow the `next` URL in the response body |

## Recommended Auto-Allow Settings

To enable fast, uninterrupted PostHog queries, add these permissions to the project's `.claude/settings.local.json`. The API key is scoped to read-only, so all commands are safe.

```json
{
  "permissions": {
    "allow": [
      "Bash(phog() *)",
      "Bash(phog *)"
    ]
  }
}
```

### Available Tokens
* `POSTHOG_USER_READ_TOKEN`
