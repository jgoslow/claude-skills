---
name: github-read
description: This skill should be used when the user asks to "list repos", "check PRs", "view commits", "investigate branches", "generate a blame report", "show contributor stats", "search GitHub", "compare branches", or any read-only GitHub API operation. Requires GITHUB_USER_READ_TOKEN.
---

## Authentication

Set `GH_TOKEN` from `GITHUB_USER_READ_TOKEN` before every `gh` command. Use the inline pattern:

```bash
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme
```

Before running queries, verify the token is valid:

```bash
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh auth status
```

If auth fails, check that `GITHUB_USER_READ_TOKEN` is set and has the required scopes (`repo`, `read:org`).

Check rate limit before bulk operations:

```bash
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api rate_limit --jq '.rate'
```

## Performance Rules

1. **Use `--jq` for server-side filtering** — avoid downloading full payloads
2. **Use `--json` with explicit field names** — only fetch what is needed
3. **Use GraphQL for bulk queries** (>10 items) — one request instead of many
4. **Use `--paginate`** on REST endpoints for automatic pagination
5. **Run independent queries in parallel** with `&` + `wait`

```bash
# Parallel: fetch repo info and PR list simultaneously
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh repo view LyraDesigns/lux-shopify-theme --json name,description &
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh pr list -R LyraDesigns/lux-shopify-theme --json number,title &
wait
```

## Repository Operations

```bash
# List repos for an org/user
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh repo list LyraDesigns --limit 100 --json name,description,updatedAt

# View repo details
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh repo view LyraDesigns/lux-shopify-theme --json name,description,primaryLanguage,stargazerCount,defaultBranchRef

# Browse directory contents
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/contents/src --jq '.[].name'

# Get file content (base64 decoded)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/contents/README.md --jq '.content' | base64 -d

# Search repos
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh search repos --owner LyraDesigns --limit 50 --json name,description
```

## Pull Request Operations

```bash
# List open PRs
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh pr list -R LyraDesigns/lux-shopify-theme --state open --json number,title,author,createdAt

# List PRs by author
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh pr list -R LyraDesigns/lux-shopify-theme --author username --json number,title,state

# View PR details with commits and reviews
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh pr view 42 -R LyraDesigns/lux-shopify-theme --json title,body,commits,reviews,files,state

# List files changed in a PR
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/pulls/42/files --jq '.[].filename'

# View PR diff
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh pr diff 42 -R LyraDesigns/lux-shopify-theme

# Search PRs across repos
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh search prs --repo LyraDesigns/lux-shopify-theme "bug fix" --state merged --json number,title
```

## Branch & Commit Operations

```bash
# List all branches
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/branches --jq '.[].name'

# Get branch tip commit
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/git/ref/heads/main --jq '.object.sha'

# Compare two branches (commits and files)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/compare/main...feature-branch --jq '{ahead: .ahead_by, behind: .behind_by, commits: [.commits[].commit.message], files: [.files[].filename]}'

# List recent commits (last 20)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/commits --jq '.[0:20] | .[] | {sha: .sha[0:7], message: .commit.message, author: .commit.author.name, date: .commit.author.date}'

# Commits on a specific branch
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api "repos/LyraDesigns/lux-shopify-theme/commits?sha=main&per_page=10" --jq '.[] | {sha: .sha[0:7], message: .commit.message}'

# Commits since a date
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api "repos/LyraDesigns/lux-shopify-theme/commits?since=2025-01-01T00:00:00Z" --jq '.[] | {sha: .sha[0:7], message: .commit.message, date: .commit.author.date}'

# View a specific commit (files changed)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/commits/abc1234 --jq '{message: .commit.message, author: .commit.author.name, files: [.files[] | {name: .filename, changes: .changes, additions: .additions, deletions: .deletions}]}'
```

## Blame & File History

```bash
# File commit history (who changed a file and when)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api "repos/LyraDesigns/lux-shopify-theme/commits?path=src/index.js" --jq '.[] | {sha: .sha[0:7], author: .commit.author.name, date: .commit.author.date, message: .commit.message}'

# Get file at a specific commit
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api "repos/LyraDesigns/lux-shopify-theme/contents/src/index.js?ref=abc1234" --jq '.content' | base64 -d

# Search for code patterns across a repo
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh search code "className" --repo LyraDesigns/lux-shopify-theme --json path,textMatches

# Search code across all org repos
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh search code "API_KEY" --owner LyraDesigns --json repository,path
```

## Contributor & Activity Stats

```bash
# Top contributors by commit count
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/contributors --jq '.[] | {login: .login, contributions: .contributions}'

# Per-author weekly additions/deletions (detailed)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/stats/contributors --jq '.[] | {author: .author.login, total: .total, weeks: (.weeks | length)}'

# Weekly commit activity (last year, 52 entries)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/stats/commit_activity --jq '.[] | {week: .week, total: .total}'

# Code frequency (weekly additions/deletions)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api repos/LyraDesigns/lux-shopify-theme/stats/code_frequency --jq '.[] | {week: .[0], additions: .[1], deletions: .[2]}'
```

## GraphQL Quick Reference

Use GraphQL for bulk queries. Much faster than multiple REST calls.

```bash
# Bulk: list repos with details (up to 100 in one request)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api graphql -f query='
query {
  organization(login: "LyraDesigns") {
    repositories(first: 100, orderBy: {field: UPDATED_AT, direction: DESC}) {
      nodes { name description updatedAt stargazerCount primaryLanguage { name } }
      pageInfo { hasNextPage endCursor }
    }
  }
}' --jq '.data.organization.repositories.nodes'

# Bulk: list PRs with nested commit and review data
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api graphql -f query='
query {
  repository(owner: "LyraDesigns", name: "lux-shopify-theme") {
    pullRequests(first: 50, states: OPEN, orderBy: {field: UPDATED_AT, direction: DESC}) {
      nodes {
        number title author { login } createdAt
        commits(last: 1) { nodes { commit { message } } }
        reviews(first: 5) { nodes { author { login } state } }
      }
    }
  }
}' --jq '.data.repository.pullRequests.nodes'

# Paginated GraphQL (use endCursor for next page)
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api graphql -f query='
query($cursor: String) {
  repository(owner: "LyraDesigns", name: "lux-shopify-theme") {
    refs(refPrefix: "refs/heads/", first: 100, after: $cursor) {
      nodes { name target { ... on Commit { history(first: 1) { nodes { message author { name } committedDate } } } } }
      pageInfo { hasNextPage endCursor }
    }
  }
}' --jq '.data.repository.refs'
```

## Issues & Actions (Read-Only)

```bash
# List open issues
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh issue list -R LyraDesigns/lux-shopify-theme --state open --json number,title,labels,assignees

# View issue details
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh issue view 10 -R LyraDesigns/lux-shopify-theme --json title,body,comments

# List recent workflow runs
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh run list -R LyraDesigns/lux-shopify-theme --limit 10 --json databaseId,name,status,conclusion

# View specific run
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh run view 12345 -R LyraDesigns/lux-shopify-theme --json name,status,conclusion,jobs

# List releases
GH_TOKEN=$GITHUB_USER_READ_TOKEN gh release list -R LyraDesigns/lux-shopify-theme --json tagName,name,publishedAt
```

## Troubleshooting

| Problem | Diagnostic Command |
|---|---|
| Auth failure | `GH_TOKEN=$GITHUB_USER_READ_TOKEN gh auth status` |
| Rate limited | `GH_TOKEN=$GITHUB_USER_READ_TOKEN gh api rate_limit --jq '.rate'` |
| 404 on repo | Check repo name spelling and that token has `repo` scope |
| Empty stats | Stats endpoints return 202 on first call; retry after a few seconds |
| Pagination | Add `--paginate` flag or use GraphQL cursors |

## Recommended Auto-Allow Settings

To enable fast, uninterrupted GitHub queries, add these read-only permissions to the project's `.claude/settings.local.json`. All commands below are strictly read-only — no write operations are included.

```json
{
  "permissions": {
    "allow": [
      "Bash(gh api:*)",
      "Bash(gh repo list:*)",
      "Bash(gh repo view:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr view:*)",
      "Bash(gh pr diff:*)",
      "Bash(gh pr checks:*)",
      "Bash(gh search repos:*)",
      "Bash(gh search prs:*)",
      "Bash(gh search issues:*)",
      "Bash(gh search code:*)",
      "Bash(gh search commits:*)",
      "Bash(gh issue list:*)",
      "Bash(gh issue view:*)",
      "Bash(gh run list:*)",
      "Bash(gh run view:*)",
      "Bash(gh release list:*)",
      "Bash(gh release view:*)",
      "Bash(GH_TOKEN=* gh api:*)",
      "Bash(GH_TOKEN=* gh repo list:*)",
      "Bash(GH_TOKEN=* gh repo view:*)",
      "Bash(GH_TOKEN=* gh pr list:*)",
      "Bash(GH_TOKEN=* gh pr view:*)",
      "Bash(GH_TOKEN=* gh pr diff:*)",
      "Bash(GH_TOKEN=* gh pr checks:*)",
      "Bash(GH_TOKEN=* gh search:*)",
      "Bash(GH_TOKEN=* gh issue list:*)",
      "Bash(GH_TOKEN=* gh issue view:*)",
      "Bash(GH_TOKEN=* gh run list:*)",
      "Bash(GH_TOKEN=* gh run view:*)",
      "Bash(GH_TOKEN=* gh release list:*)",
      "Bash(GH_TOKEN=* gh release view:*)",
      "Bash(GH_TOKEN=* gh auth status:*)"
    ]
  }
}
```

The `GH_TOKEN=*` prefixed variants cover the inline authentication pattern used throughout this skill. Excluded write operations: `create`, `merge`, `close`, `edit`, `delete`, `fork`, `comment`, `review`.

### Available Tokens
* `GITHUB_USER_READ_TOKEN`
