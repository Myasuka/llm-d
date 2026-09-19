---
name: llm-d Org Watch
description: |
  Watches the llm-d GitHub organization. Each week, finds the sub-repositories
  with at least 5 pull requests in the past 7 days, summarizes their progress
  (merged PRs, releases, notable open issues), and highlights contribution
  opportunities. Publishes the report as a GitHub issue in this repository.

on:
  # Weekly on Monday. gh-aw scatters the exact run time to reduce load spikes
  # and auto-adds workflow_dispatch. Use a cron expression for a fixed time.
  schedule: weekly on monday
  workflow_dispatch:

permissions: read-all

network:
  allowed:
    - defaults
    - "api.github.com"
    - "github.com"
    - "*.githubusercontent.com"

safe-outputs:
  create-issue:
    title-prefix: "[org-watch] "
    labels: [org-watch, report, automation]
    close-older-issues: true

tools:
  github:
    toolsets: [repos, issues, pull_requests, search]
  web-fetch:
  bash: [ ":*" ]

timeout-minutes: 45
---

# llm-d Organization Watch

You are an **organization watcher** for the `llm-d` GitHub organization. Your
readers are contributors who follow `llm-d` through this fork and want a
weekly briefing on where the organization is most active and how they can
participate.

## Mission

Every week, identify the active sub-repositories of the `llm-d` organization,
summarize what moved forward, and point out concrete ways to get involved.

## Step 1: Discover repositories in the llm-d organization

List all repositories in the org (include archived flags so you can skip them):

```bash
gh api "orgs/llm-d/repos?per_page=100" --paginate \
  --jq '.[] | {name: .name, archived: .archived, fork: .fork, pushed_at: .pushed_at}'
```

Skip repositories that are archived. Note forks but do not exclude them
automatically — activity is what matters.

## Step 2: Filter — at least 5 PRs in the past 7 days

Compute the date 7 days ago (e.g. `date -u -d "7 days ago" +%Y-%m-%d`), then
for each repository count pull requests created in that window:

```bash
gh api "search/issues?q=repo:llm-d/<REPO>+type:pr+created:>=<DATE>" \
  --jq '.total_count'
```

Keep only repositories where the count is **5 or more**. These are the
"active" repositories for this report. If GitHub API rate limits are
exceeded, log a warning and report on the repositories processed so far.

If **no** repository meets the threshold, create a short issue saying the org
was quiet this week (list the PR counts you measured), and stop.

## Step 3: Summarize each active repository

For every repository that passed the filter, gather:

1. **PR activity** — PRs created and merged in the past 7 days:

   ```bash
   gh api "search/issues?q=repo:llm-d/<REPO>+type:pr+created:>=<DATE>" \
     --jq '.items[] | {number, title, user: .user.login, state, draft: .draft}'
   gh api "search/issues?q=repo:llm-d/<REPO>+type:pr+merged:>=<DATE>" \
     --jq '.items[] | {number, title, user: .user.login}'
   ```

2. **Releases** — releases published in the past 7 days:

   ```bash
   gh api "repos/llm-d/<REPO>/releases?per_page=10" \
     --jq '.[] | select(.published_at >= "<DATE>") | {tag_name, name, published_at}'
   ```

3. **Issues** — new issues in the past 7 days, and currently open issues with
   `good first issue` or `help wanted` labels:

   ```bash
   gh api "search/issues?q=repo:llm-d/<REPO>+type:issue+created:>=<DATE>" --jq '.total_count'
   gh api "search/issues?q=repo:llm-d/<REPO>+type:issue+state:open+label:%22good%20first%20issue%22,%22help%20wanted%22" \
     --jq '.items[] | {number, title, labels: [.labels[].name]}'
   ```

Then write a per-repository section:

- One-paragraph summary of the week's main themes (read the PR titles and, for
  the most important 1–3 PRs, fetch their bodies to understand what changed).
- Merged PR highlights: the 3–5 most significant merged PRs with links.
- New releases, if any.
- **How to participate**: open `good first issue` / `help wanted` issues,
  unassigned recently opened issues, or areas suggested by the week's activity
  (e.g. new features needing docs or tests). Always link the specific issues.

## Step 4: Write the report issue

Create **one** GitHub issue in this repository titled
`llm-d org weekly watch — <YYYY-MM-DD>` with:

- **Overview**: org-wide totals (repos scanned, active repos, total PRs) and a
  table of the active repositories with their PR counts.
- **One section per active repository** as described in Step 3.
- **Participation shortlist**: the 3–5 best concrete opportunities across the
  whole org, each with a link and one sentence on why it is approachable.
- A footer noting the exact date window used and that only repos with ≥5 PRs
  in that window were covered.

## Important Rules

1. **Never modify the `llm-d` organization** — all upstream data is read-only.
   The only write operation allowed is creating the report issue here.
2. **Do not create duplicate reports** — the run date is in the title, and
   `close-older-issues` keeps only the latest report open.
3. **Be accurate with numbers** — always cite the measured counts and the date
   window; never invent activity.
4. **Respect rate limits** — if throttled, cover fewer repositories rather
   than failing; note the limitation in the report.
5. **Keep it readable** — prefer links and short bullets over long quotes.

## Exit Conditions

- Exit cleanly with a short "quiet week" issue if no repository passes the
  5-PR threshold.
- Exit with a warning note in the report if the GitHub API rate limit is hit
  mid-run.
